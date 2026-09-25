# Netbird on Podman

If you follow the official docs from netbird, you will see that there is a script which will ask
you some questions and start right after that all the docker containers we need.
But we don't want use docker, we want use Podman.

But here is a small impact. the script will create some files for us which are important.
So what i did, is to execute the script, answer the question and interrupt with crtl+c at
the docker pull.

The  script will create a ```.config.yaml, dashboard.env``` and if you hit the recommend section [0]
you get a ```proxy.env and traefik-dynamic.yaml``` as well. I use the [1] section with an exeternal reverse
proxy which i can completely configure by myself.

I let the generated files in this repo, so that you dont need execute netbirds script.

This Netbird server will be installed on a VPS and need to be reachable from the internet via port 80/tcp and 443/tcp.
Port 3478/udp need to be open as well, but it need to directly published from the netbird container. Not over
the reverse proxy.
Because i use my own Traefik reverse proxy, left the .container file in this repo as well.

### Spezial Config:

In my setup, i created a wireguard site to site connection from home to this vps. Pocket ID is hosted at home.
So in my case, i configure a tls-passthrough from the vps-traefik through the wireguard tunnel to the traefik at home. The traefik at home handle the tls certificate for Pocket ID and request Pocket ID for you.
That has a little advantage in my opinion that i do not depend on the running netbird instance to reach my authentication service.

Keep and eye at the ```AddHost=auth.example.com:host-gateway```. In my Setup, Pocket-ID is reachable
over the same external ip address. So podman pasta network will block those traffic. You can not request yourself
over your external ip address. So i need to put this key=value to the container section. It is basically a host
entry like under /etc/hosts on linux.