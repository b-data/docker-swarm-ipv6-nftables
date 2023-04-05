[![minimal-readme compliant](https://img.shields.io/badge/readme%20style-minimal-brightgreen.svg)](https://github.com/RichardLitt/standard-readme/blob/master/example-readmes/minimal-readme.md) [![Project Status: Active – The project has reached a stable, usable state and is being actively developed.](https://www.repostatus.org/badges/latest/active.svg)](https://www.repostatus.org/#active) <a rel="license" href="http://creativecommons.org/licenses/by/4.0/"><img alt="Creative Commons License" style="border-width:0" src="https://i.creativecommons.org/l/by/4.0/88x31.png" height = 20 /></a>

| :exclamation: Users of Swarm overlay networks should review [GHSA-vwm3-crmr-xfxw](https://github.com/moby/moby/security/advisories/GHSA-vwm3-crmr-xfxw) to ensure that unintentional exposure has not occurred. |
|:----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|

# Docker swarm + IPv6 + nftables

b-data has been running a single-node docker swarm on a Debian host for years.
Since our ISP has also been providing IPv6 addresses for some time, I wanted to
know whether docker can handle a dual-stack network.

What [this project](https://gitlab.com/b-data/docker/docker-swarm-ipv6-nftables)
is about:

*  nftables
*  Single-node docker swarm
*  Docker host: Public IPv4/IPv6 address
*  Docker networks: Private IPv4/IPv6 addresses only
*  Docker containers: Ability to connect to the internet using IPv6

What this project is **not** about:

*  IPv6 only
*  Multi-node docker swarm
*  Docker host: IPv6 Prefix Delegation (PD)
*  Docker networks: Using delegated public IPv6 addresses
*  Docker containers: Service deployment using delegated public IPv6 adresses

## Table of Contents

*  [Prerequisites](#prerequisites)
*  [Install](#install)
*  [Usage](#usage)
*  [Contributing](#contributing)
*  [References](#references)
*  [License](#license)

## Prerequisites

This projects requires a
[DynDNS](https://oci.dyn.com/dynamic-dns-hostname-search/) account and
the installation of ddclient, docker and docker compose (v2) on a Debian-based
Linux distribution.

### ddclient

*  [Installation](https://ddclient.net/#installation)

See [ddclient/ddclient#75 (comment) ff](https://github.com/ddclient/ddclient/issues/75#issuecomment-977732035)
on how to set up a second _ddclient_ service to update both IPv4 and IPv6
addresses.

### Docker

To install docker, follow the instructions for your platform:

*  [Install Docker Engine | Docker Documentation > Supported platforms](https://docs.docker.com/engine/install/#supported-platforms)
*  [Post-installation steps for Linux | Docker Documentation](https://docs.docker.com/engine/install/linux-postinstall/)

### Docker Compose

*  [Compose V2 | Docker Documentation > Installing Compose V2](https://docs.docker.com/compose/cli-command/#installing-compose-v2)

## Install

### Get started

*  All required files are provided in this project.  
    → The subdirectories are relative to path `/` on the server.
*  If there is a `diff`, modify the lines of the file accordingly.
*  If there is no `diff`, use the entire file provided in this project.

### Configure kernel parameters at boot

```bash
sudo nano /etc/sysctl.d/99-ipv6.conf
```

> **Note**: IPv6 forwarding may interfere with your existing IPv6 configuration:
> If you are using Router Advertisements to get IPv6 settings for your host's
> interfaces, set `accept_ra` to `2` using the following command.  
> Otherwise IPv6 enabled forwarding will result in rejecting Router
> Advertisements.
> 
>     $ sysctl net.ipv6.conf.eth0.accept_ra=2

— [docker.github.io/ipv6.md at c0eb65aabe4de94d56bbc20249179f626df5e8c3 · docker/docker.github.io](https://github.com/docker/docker.github.io/blob/c0eb65aabe4de94d56bbc20249179f626df5e8c3/engine/userguide/networking/default_network/ipv6.md)

**Note**: Replace `eth0` with the name of your server's primary network interface.

```bash
sudo reboot
```

### Set up nftables

```bash
sudo apt update
sudo apt install -y nftables
sudo mv /etc/nftables.conf /etc/nftables.conf.orig
sudo nano /etc/nftables.conf
```

> Because **docker** does not know about the different firewall that is used we
> need to adhere to a few things, to make our ruleset backwards compatible to
> tools using iptables:
> 
> *  use an **ip** and **ip6** table instead of [inet](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=1d49144c0aaa61be4e3ccbef9cc5c40b0ec5f2fe)
> *  name all chains exactly as in **iptables**: **INPUT**, **OUTPUT** &
>    **FORWARD**
> 
> Any change in this will cause errors, since the rules from **docker** might
> end up in the wrong place.

— [debian, docker and nftables](https://ehlers.berlin/blog/nftables-and-docker/)

```bash
sudo systemctl enable nftables
sudo reboot
```

### Modify docker daemon

```bash
sudo nano /etc/docker/daemon.json
```

> **Note**: IPv6 networking is only supported on Docker daemons running on Linux
> hosts.

— [Enable IPv6 support | Docker Documentation](https://docs.docker.com/config/daemon/ipv6/)

> With docker 20.10.5 and docker-compose 1.28.6 you don't need to use the
> ip6tables commands manually and docker can take care of doing the NAT properly
> (which is MUCH better!).
> 
> For that you need to have `"experimental": true` in `/etc/docker/daemon.json`,
> along with `"ip6tables": true`. Then restart docker service and check it has
> an ipv6 on docker0 bridge.

— [Docker IPV6 Guide - DEV Community (comment)](https://dev.to/elabftw/comment/1d0cp)

```bash
sudo reboot
```

**Note**: In my case, the `docker0` bridge only got an IPv6 Gateway assigned after a
reboot.

#### Inspection

```bash
docker network inspect bridge
```

```bash
sudo nft list ruleset
```

## Usage

### Set up a single-node swarm

#### [Use overlay networks | Docker Documentation > Customize the docker_gwbridge interface](https://docs.docker.com/network/overlay/#customize-the-docker_gwbridge-interface)

```bash
docker network create \
  --ipv6 \
  -o com.docker.network.bridge.name=docker_gwbridge \
  -o com.docker.network.bridge.enable_icc=false \
  -o com.docker.network.bridge.enable_ip_masquerade=true \
  --subnet 172.18.0.0/16 \
  --gateway 172.18.0.1 \
  --subnet fd00:1::/80 \
  --gateway fd00:1::1 \
  docker_gwbridge
```

#### [Create a swarm | Docker Documentation](https://docs.docker.com/engine/swarm/swarm-tutorial/create-swarm/)

```bash
docker swarm init
```

#### Inspection

```bash
docker network inspect docker_gwbridge
```

### Serve webservices with IPv6

#### Create an IPv6 enabled overlay network

```bash
docker network create \
  -d overlay \
  --ipv6 \
  -o encrypted \
  --subnet "10.0.2.0/24" \
  --subnet "fd00:2::/80" \
  webproxy
```

#### Install Træfik + whoami

1.  Clone repository https://gitlab.b-data.ch/docker/deployments/traefik.git  
    ```bash
    mkdir ~/docker
    cd ~/docker
    git clone https://gitlab.b-data.ch/docker/deployments/traefik.git webproxy
    ```
1.  Copy the following files:
    *  [home/user/docker/webproxy/.env](home/user/docker/webproxy/.env)
       to `~/docker/webproxy/.env`
    *  [home/user/docker/webproxy/docker-compose.yml](home/user/docker/webproxy/docker-compose.yml)
       to `~/docker/webproxy/docker-compose.yml`
1.  Update environment variables `TF_ACME_EMAIL` in '.env':
    *  Replace `postmaster@mydomain.com` with a valid email address of yours.
1.  Update labels for service whoami in 'docker-compose.yml':
    *  Replace `hostname.dyndns.org` with a valid DynDNS hostname of yours.
1.  Deploy the docker stack:  
    ```bash
    cd ~/docker/webproxy
    env $(cat .env | grep "^[A-Z]" | xargs) docker stack deploy -c docker-compose.yml webproxy
    ```

#### Inspection

Wait a few minutes and visit http://hostname.dyndns.org.

This project is used to serve https://whoami.b-data.ch. You may `curl` this
website until further notice.

Resolving to IPv4 address only:

```bash
curl -4 https://whoami.b-data.ch
```

Resolving to IPv6 address only:

```bash
curl -6 https://whoami.b-data.ch
```

## Contributing

PRs accepted.

This project follows the
[Contributor Covenant](https://www.contributor-covenant.org)
[Code of Conduct](CODE_OF_CONDUCT.md).

## References

*  Configure kernel parameters at boot:
   [wido/docker-ipv6](https://github.com/wido/docker-ipv6)
    *  [docker.github.io/ipv6.md at c0eb65aabe4de94d56bbc20249179f626df5e8c3 · docker/docker.github.io](https://github.com/docker/docker.github.io/blob/c0eb65aabe4de94d56bbc20249179f626df5e8c3/engine/userguide/networking/default_network/ipv6.md)
*  Set up nftables: [debian, docker and nftables](https://ehlers.berlin/blog/nftables-and-docker/)
*  Modify docker daemon: [Docker IPV6 Guide - DEV Community](https://dev.to/csgeek/docker-ipv6-guide-235d)
    *  [dockerd | Docker Documentation > Daemon configuration file](https://docs.docker.com/engine/reference/commandline/dockerd/#daemon-configuration-file)
    *  [Enable IPv6 support | Docker Documentation](https://docs.docker.com/config/daemon/ipv6/)

See also [notes](NOTES.md) for further information.

## [License](LICENSE.md)

This work is licensed under a <a rel="license" href="http://creativecommons.org/licenses/by/4.0/">Creative Commons Attribution 4.0 International License</a>.
