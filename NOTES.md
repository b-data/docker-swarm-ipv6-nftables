# Notes

## Træfik Proxy

In this project, traefik's ports are published using the `mode=host` and
`published=<PORT>` options. This ensures the following:

1.  Listening to both `tcp` (IPv4) and `tcp6` (IPv6).
1.  Getting the client IP (`X-Forwarded-For`/`X-Real-IP`).

### Publish ports directly on the swarm node

> If you need to read the client IP in your applications/stacks using the
> `X-Forwarded-For` or `X-Real-IP` headers provided by Traefik, you need to make
> Traefik listen directly, not through Docker Swarm mode, even while being
> deployed with Docker Swarm mode.
> 
> For that, you need to publish the ports using "host" mode.

— [Traefik Proxy with HTTPS - Docker Swarm Rocks > Getting the client IP](https://dockerswarm.rocks/traefik/#getting-the-client-ip)

Using the `mode=host` option, the service's ports are published directly on the
node where it is running.

> :exclamation: **Note: If you publish a service’s ports directly on the swarm
> node using `mode=host` and also set `published=<PORT>` this creates an
> implicit limitation that you can only run one task for that service on a given
> swarm node. You can work around this by specifying `published` without a port
> definition, which causes Docker to assign a random port for each task.**
> 
> **In addition, if you use `mode=host` and you do not use the `--mode=global`
> flag on `docker service create`, it is difficult to know which nodes are
> running the service to route work to them.**

— [Deploy services to a swarm | Docker Documentation > Publish ports](https://docs.docker.com/engine/swarm/services/#publish-ports)

### Alternative: Publish ports using the routing mesh

If you don't need the client IP in your applications/stacks, you can publish
traefik's ports using the routing mesh.

```bash
sudo nano ~/docker/webproxy/docker-compose.yml
```

```diff
diff --git a/docker-compose.yml b/docker-compose.yml
index d3516bd..543b2d3 100644
--- a/docker-compose.yml
+++ b/docker-compose.yml
@@ -4,15 +4,9 @@ services:
   traefik:
     image: traefik:${TF_VERSION}
     ports:
-      - target: 80
-        published: 80
-        mode: host
-      - target: 443
-        published: 443
-        mode: host
-      #- target: 8080
-      #  published: 8080
-      #  mode: host
+      - '80:80'
+      - '443:443'
+      #- '8080:8080'
     networks:
       - webproxy
     volumes:
```

Apply the changes to the service:

```bash
cd ~/docker/webproxy
env $(cat .env | grep "^[A-Z]" | xargs) docker stack deploy -c docker-compose.yml webproxy
```

Somehow, this setup only allows requests on `tcp` (IPv4). Solution: Set up two
services to listen on `tcp6` (IPv6) ports 80 and 443 and redirect the requests
to `127.0.0.1` (localhost) on `tcp` (IPv4).

Set up service:

```bash
sudo nano /etc/systemd/system/swarm-ipv6@.service
```

```bash
sudo nano /etc/systemd/system/swarm-ipv6@.socket
```

Enable services for ports 80 and 443:

```bash
sudo systemctl daemon-reload
sudo systemctl enable swarm-ipv6@80.service
sudo systemctl enable swarm-ipv6@443.service
```

Reboot:

```bash
sudo reboot
```

## IPv6 connectivity

### Internet to container

Services can be reached via AAAA records for the public IPv6 address, while
traefik forwards the requests to the containers based on their labels.

### Container to internet

Containers can reach services using IPv6 - both on `bridge` and `overlay`
networks.

Test using `bridge` network:

```bash
docker run --rm -ti busybox ping6 -c 4 google.com
```

Test using `overlay` network:

```bash
mkdir -p ~/docker/busybox
nano ~/docker/busybox/docker-compose.yml
cd ~/docker/busybox
docker stack deploy -c docker-compose.yml busybox
docker exec -ti $(docker ps -q -f name=busybox_ipv6) ping6 -c 4 google.com
```

**Note**: Setting `net.ipv6.conf.eth0.accept_ra=2` in
`/etc/sysctl.d/99-ipv6.conf` is required for this to work.

### Container to container

Containers can reach each other using IPv6 - both in `bridge` and `overlay`
networks.

Test using `overlay` network:

```bash
docker exec -ti $(docker ps -q -f name=busybox_ipv6) ping6 $(docker container inspect -f '{{ .NetworkSettings.Networks.webproxy.GlobalIPv6Address }}' $(docker ps -q -f name=webproxy_traefik))
```

However - and in contrast to IPv4 - containers can **not** reach each other
using IPv6 by service name in `overlay` networks.

Test using `overlay` network:

```bash
docker exec -ti $(docker ps -q -f name=busybox_ipv6) ping6 traefik
```

**Note**: IPv6 service discovery works perfectly well in `bridge` networks,
though.

#### Consequence

This means that docker containers in an `overlay` network communicate via IPv4.

**Note**: [Alpine](https://hub.docker.com/_/alpine)-based containers seem to
have a problem with dual-stack `overlay` networks.  
Use the _default_ version (e.g. `nginx` instead of `nginx:alpine`) to avoid any
problems.

## IPv6 address range

The address block `2001:db8:1::/64` mentioned on [Enable IPv6 support | Docker Documentation](https://docs.docker.com/config/daemon/ipv6/)
is a subnet of `2001:db8::/32`, which is

> a reserved prefix for use in documentation.

— [RFC 3849: IPv6 Address Prefix Reserved for Documentation](https://www.rfc-editor.org/rfc/rfc3849)

Therefore, private `fd00:[0, ffff]::/80` subnets are used in this project.

## References

*  Publish ports directly on the swarm node:
   [Traefik Proxy with HTTPS - Docker Swarm Rocks](https://dockerswarm.rocks/traefik/),
   [Deploy services to a swarm | Docker Documentation](https://docs.docker.com/engine/swarm/services/)
*  Alternative: Publish ports using the routing mesh: [moby/moby#24379 (comment)](https://github.com/moby/moby/issues/24379#issuecomment-569603301)
*  IPv6 address range: [Docker - ArchWiki > Configuration > IPv6](https://wiki.archlinux.org/title/Docker#IPv6)
    *  [RFC 3849: IPv6 Address Prefix Reserved for Documentation](https://www.rfc-editor.org/rfc/rfc3849)
    *  [RFC 4193: Unique Local IPv6 Unicast Addresses](https://www.rfc-editor.org/rfc/rfc4193)
