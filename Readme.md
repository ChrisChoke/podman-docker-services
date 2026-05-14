# Podman

My journey with podman and quadlets...

## Topology
What i want with this repo is to document my setup in my homelab for this podman services. I try to maintain a docker-compose equivalent for those who more like using docker.  
I decided for the following topology:

```
Client ----> Reverse Proxy ----> Traefik ----> Services
              External RP       |     Podman Host    |
```
So the Traefik on the Podman Host is the only entry point on this host/machine.  
No service publish their ports directly.  
The second reverse proxy in front of the podman host manage all the web certificates for my services. This is easier for me to have one single source where i need configuring those things.
But it forces me to extend configuration of traefik to trust this external proxy. This is necessary to get the x-forwared-for informations and get the origin client ip address.
And this is necessary to implement things like fail2ban or ACL on application layer within the services.

## User

If you want to use a dedicated user for the podman service you need to install ```systemd-container``` which is not installed
by default on every OS. With this package ```machinectl``` comes as binary. This is what we need to switch to the users context
. Debian default ``su`` command doesnt work, because there is any missing environment variable.

```
Failed to connect to user scope bus via local transport: $DBUS_SESSION_BUS_ADDRESS and $XDG_RUNTIME_DIR not defined (consider using --machine=<user>@.host --user to connect to bus of other user)
```

if you installed this, you can switch to the user with 
```
sudo machinectl shell --uid 1001
or
sudo machinectl shell username@
```

```useradd -d /mnt/data/podmanusr -s /bin/bash``` (yes, no password and yes, other home location)
This user will contains the container images in his home directory under ```~/.local/share/containers/```
and therefor we need some space. I like to use a second hdd with only one partition on it. With proxmox you can easy
expand the hdd and with cfdisk you can expand the new space to the partition. So you dont need think about how handle this
while you OS is installed on the same disk.

```loginctl enable-linger podmanusr``` to enable systemd service execution while not logged in.


To use traefik reverse proxy with the docker-like api using labels, you need to start the podman socket manually.
It should pre configured under ```/usr/lib/systemd/user/podman.socket```.

So you can easily enable and start it immediately in your user context you are logged in.

```
systemctl --user enable podman.socket --now
```

## Systemd Socket Activation

This seems to be neccessary if you want to see the real ip-addresses of clients connecting to your services.
If you don't do this you will see any NAT ip-address from pasta network (the default network stack since podman 5.x)

Socket activation for containers. The .socket file has to be placed in ```~/.config/systemd/user/```
then the container can use this sockets and no --publish options are required in the [container] section
of the .container file. At least for this ports in the .socket file. For any further one, you may need it.

### Binding lower ports

It seems that podman can not handle linux capabilities for its own "podman" binary. So there is currently no other option
than lower down the starting point of the unprivileged ports to 80. But this is a general OS config. So any user on this system
can bind lower port since port 80 and upper.
To make this persitent you need to do:

```echo "net.ipv4.ip_unprivileged_port_start=80" > /etc/sysctl.d/10unpriv_ports.conf```

If you dont like to do this. You can do redirect with your OS firewall like nft. You will redirect port 80 and 443 to your
reverse proxy containers port for example. The Container use upper ports >= 1024 and your redirect point to this ports.

```
nft add table nat
nft 'add chain nat prerouting { type nat hook prerouting priority -100; }'

nft add rule nat prerouting tcp dport 80 redirect to 8080
nft add rule nat prerouting tcp dport 443 redirect to 8443
```

For persistent ruleset put this together in ```/etc/nftables.conf``` and enable ```nftables.service```. A oneshot service which runs on OS startup once to load the firewall rules.

```
table ip nat {
        chain prerouting {
                type nat hook prerouting priority dstnat; policy accept;
                tcp dport 80 redirect to :8080
                tcp dport 443 redirect to :8443
        }
}
```

## .env file

In this repo i try to make a kind of template of the container definitions. So i using .env for the base domain
to hide my own domain from git.
Keep in mind. Currently this contains only the basedomain in the systemd context (Service section). So the variable is not accessable within the container.

Create a .env file under ```~/.config/containers/systemd/.env``` from your podman user. Or rename the .env.example file.
I try to list all variables in the example that you won't miss one.
