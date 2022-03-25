# Rootless serverless containers with podman and systemd
This project walks through all the steps in this [blog post](https://www.redhat.com/en/blog/painless-services-implementing-serverless-rootless-podman-and-systemd)
by [Pietro Bertera](https://www.redhat.com/en/authors/pietro-bertera).
Please refer to that blog post if you have questions related to
this.

I would charaterize this as quasi-serverless. Simply put, a TCP
connection request spins up a simple web server that will time out
after ten seconds. This won't scale beyond one instance and spin
down is simply based on a timeout. Full serverless implementations
can provide more sophisticated features like auto-scaling.

## Setup
### Install RHEL 9
Start with a minimal install of RHEL 9 (in beta now). You can
download RHEL 9 for free at the [Red Hat Developer site](https://developers.redhat.com/products/rhel/overview).

In my install, I enabled a single network interface. Please make
sure that the command `ip route get 8.8.8.8 |awk '{print $7; exit}'`
returns the IP address of your host, whether its a virtual or
physical installation.

RHEL 9 beta is used since it includes systemd release 250. The
auto-scale down feature was added to systemd-socket-proxyd in systemd
release 246.

### Run the setup scripts
Copy (or git clone) this project onto your Linux host. Change to
the directory where you've copied this content (e.g. my directory
matches the name of this project as shown below):

    cd ~/rootless-serverless-with-podman-and-systemd

Make sure the above directory matches the location of the files in
this project.

Edit the `demo.conf` file and set your credentials to register your
system with Red Hat. These can be your [Red Hat Developer](https://developers.redhat.com)
credentials or your credentials for the
[Red Hat Customer Portal](https://access.redhat.com).

Next, simply run the three scripts in numerical order to set up the
demo:

    sudo ./01-setup-rhel.sh
    sudo ./02-config-registry.sh
    ./03-build-containers.sh

I'd recommend logging out/in to the system when done.

## Demo
### Tabula Rasa
Clean up any prior work and reset everything to a known good state.

    source demo.conf
    
    podman stop --all
    podman rm -f --all
    
    podman pull --all-tags $HOSTIP:5000/httpd
    podman tag $HOSTIP:5000/httpd:v1 $HOSTIP:5000/httpd:prod
    podman push $HOSTIP:5000/httpd:prod
    
    podman rmi -f --all
    
    rm -fr ~/.config/systemd/user
    sudo loginctl disable-linger $USER
    systemctl --user daemon-reload

### Check local registry
Verify that your local registry is up and running and that it
contains the `httdp` repository. The `systemctl` command below
should show the registry service as `active` and the curl command
should return the `httpd` repository.

    source demo.conf

    systemctl status container-registry.service
    curl -s http://$HOSTIP:5000/v2/_catalog | jq

### Create an httpd container service
We can use podman to generate the required systemd service file to
manage our httpd container application. The first thing we need to
do is create a container instance. The `podman create` command does
not run the container, but instead allocated space on the filesystem,
pulls the image, and sets the configuration metadata for the container
instance.

    podman create --name httpd -p 127.0.0.1:8080:80 \
        --label=io.containers.autoupdate=registry $HOSTIP:5000/httpd:prod

The label `io.containers.autoupdate` will be used later when we
demonstrate the `podman auto-update` command.

Create the local directory to hold the various systemd configuration
files for rootless operation.

    mkdir -p ~/.config/systemd/user

Podman can be used to generate the systemd service file needed to
run the container application.

    podman generate systemd --files --name httpd --new

Edit the generated `container-httpd.service` file and remove the
`[Install]` section and the lines below it, because we aren't going
to enable this service to automatically start and boot time. Use
your favorite editor to make this change. The command for `vi` is
shown below as an example.

    vi container-httpd.service

After editing the file, move it to the systemd user configuration
directory.

    mv container-httpd.service ~/.config/systemd/user

### Create a socket listener
Create a systemd socket listener to accept connections on port 8080
and pass that to the similarly named service that will proxy the
traffic to the container application service. When the container
application isn't running, systemd manages listening for socket
connections so very little compute resources are used.

Use a heredoc to create the socket unit file as shown below. NB:
The `ListenStream` does not bind only to port 8080 as that would
cause systemd to listen on all network interfaces, and we want to
reserve the `localhost` interface for our web server application.

    cat <<EOF > container-httpd-proxy.socket
    [Socket]
    ListenStream=$HOSTIP:8080
    
    [Install]
    WantedBy=sockets.target
    EOF

Move the socket listener unit file to the local configuration
directory.

    mv container-httpd-proxy.socket ~/.config/systemd/user

### Create the proxy service
The proxy service, which is started by the socket listener when it
receives a connection request, must have the same name as the socket
unit file. Use a heredoc, as shown below, to create this file.

    cat <<EOF > container-httpd-proxy.service
    [Unit]
    Requires=container-httpd.service
    After=container-httpd.service
    Requires=container-httpd-proxy.socket
    After=container-httpd-proxy.socket
    
    [Service]
    ExecStart=/usr/lib/systemd/systemd-socket-proxyd 127.0.0.1:8080
    EOF

Move the proxy service unit file to the local configuration directory.

    mv container-httpd-proxy.service ~/.config/systemd/user

# Reload systemd daemon and start listener 
After adding the systemd local configuration files, notify systemd
of the changes and then start the socket listener:

    systemctl --user daemon-reload
    systemctl --user enable --now container-httpd-proxy.socket

Let's validate that the socket listener is bound to port 8080 on
the host IP address.

    systemctl --user list-sockets --no-pager -l
    ss -natlp | grep 8080

Also, enable the socket listener to run after boot even when the
user is not logged in. We can use the systemd `loginctl` command
to do this by enabling the `linger` option..

    sudo loginctl enable-linger $USER
    loginctl show-user $USER

### Try it out
Let's test the socket activation facility by connecting to our web
server. When we send an http request to our host IP address and
port 8080, the socket listener will start the proxy service which
in turn will start the container httpd service, since it's a
dependency of the proxy service. The proxy service will accept the
file descriptor from the socket listener and forward the http request
to the web server which will provide an http response.

    curl http://$HOSTIP:8080

You can also observe the running container by listing the podman
container processes.

    podman ps

### Auto-update of container applications
Podman provides a facility to automatically update running containers,
if a alternative container image is available. Let's take a look
at the existing container systemd service file.

    systemctl --user cat container-httpd.service --no-pager -l

Notice the label `io.containers.autoupdate=registry` in the `ExecStart`
option. This label indicates that podman should determine if an
alternative image exists in the registry for this container. If so,
the new image is pulled and the container is restarted.

Let's tag version 2 of our container web application with the `prod`
tag as shown below.

    podman images
    podman pull --all-tags $HOSTIP:5000/httpd
    podman tag $HOSTIP:5000/httpd:v2 $HOSTIP:5000/httpd:prod
    podman push $HOSTIP:5000/httpd:prod

If the `HOSTIP` environment variable is not set, simply type
`source demo.conf` to set it.

The `podman auto-update` command can be issued directly or run via
a timer to periodically check for application updates. Let's list
the running container processes and then apply the update.

    podman ps
    podman auto-update

Listing the running container processes shows that the container
was restarted.

    podman ps

You can view the updated output from the web application by sending
an http request.

    curl http://$HOSTIP:8080

### Scale down the application
In all of the above examples, the container web application
continuously runs after being started by a socket connection request.
We may not want to consume resources when there's no requests. To
prevent that, we can enable automatic scale down of our service.

Two changes must be made to our local systemd configuration files.
The first is to add the following option under the `[Unit]` header
in the container service unit file. Edit the file and add the
following line under the `[Unit]` header:

    StopWhenUnneeded=yes

Use your preferred editor to make this change. The vi editor is
used in the line below.

    vi ~/.config/systemd/user/container-httpd.service

This option will stop the container web application service when
no other systemd units depend on it. For our example, only the proxy
service depends on our container web application service, so its
termination will also stop the container web application service.

The second needed change is to set the idle timeout on the proxy
service unit file. This option was added to `systemd-socket-proxyd`
with the systemd 246 release. RHEL 9 includes systemd 250.

Modify the `ExecStart` line for the proxy service to include the
`--exit-idle-time` option. Make sure that `ExecStart` line resembles
the following:

    ExecStart=/usr/lib/systemd/systemd-socket-proxyd --exit-idle-time=10 127.0.0.1:8080

That line sets the timeout to ten seconds of inactivity. Notify
systemd that the local configuration files have changed.

    systemctl --user daemon-reload

Let's list the running containers and then stop the proxy service
which will in turn, because of the `StopWhenUnnneeded` option, stop
the container web application service.

    podman ps
    systemctl --user stop container-httpd-proxy.service
    podman ps 

You should see that there are no running podman container processes.
If we now issue an http request and then wait, you will see the web
application response and then the container service will shutdown
after ten seconds.

    curl http://$HOSTIP:8080 && for i in {1..10}; do sleep 1; podman ps; done

### Review the solution
Let's do a wrap up of this capability. We can list all of the
supporting configuration files using the following command:

    systemctl --user list-unit-files | grep container-httpd

We can list the contents of those files using the following commands:

    systemctl --user cat --no-pager container-httpd-proxy.socket
    systemctl --user cat --no-pager container-httpd-proxy.service
    systemctl --user cat --no-pager container-httpd.service

You can also use the socket statistics command to see that the
socket listener is bound to the host IP address and port 8080.

    ss -natlp | grep 8080

We can confirm that there are no running containers by listing the
podman processes:

    podman ps

Use the following command to demonstrate the serverless behavior
of our web application.

    curl http://$HOSTIP:8080 && for i in {1..10}; do sleep 1; podman ps; done
