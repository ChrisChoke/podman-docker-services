this traefik config will just used to set up an ingress controller for a docker host.
traefik is the only entrypoint for the docker services on this server.
if you need https with vaild letsenctypt certs you can do this on a other reverse proxy
which handle the certifitcates and refresh these.

labels example:

Only https when traefik has wildcard certificate as example. Maybe or not. Have to read more deep the docs.
```yaml
labels:
    - "traefik.enable=true"
    - "traefik.http.routers.jellyfin.entrypoints=https"
    - "traefik.http.routers.jellyfin.rule=Host(`jellyfin.example.com`)"
    - "traefik.http.routers.jellyfin.tls=true"
    - "traefik.http.services.jellyfin.loadbalancer.server.port=8096"
```