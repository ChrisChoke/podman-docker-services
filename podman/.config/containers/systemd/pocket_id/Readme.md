# Pocket-ID

### Some specials in my setup.

In my setup pocket-id run in a dedicated vlan on a lxc container with rootless podman.
So its is a single machine for pocket-id. The reverse proxy for pocket-id is in a dedicated vlan as well. And this reverse proxy (Traefik) is the only machine that can communicate to pocket-id's service port 1411.
That is the reason why the other markdown doc contains a traefik config for it's file provider.
But i left the Labels in the .container file to give a example.

The Setup for the the LXC container with their specials is described in a other markdown file beside of this one.

My container use a enryption.key file instead of the plain text encryption key inside the .container. If you want to do so as well, you need to move this enryption.key into the volume bind of the container. Otherwise the application can not see this file and wont start up.