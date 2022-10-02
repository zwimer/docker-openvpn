# Private Internet Access
A Fedora Docker container for connecting to the PIA servers using OpenVPN, based on [ItsDaSpecialK/docker-openvpn](https://bitbucket.org/ItsDaSpecialK/docker-openvpn/src/master/) and [colinherbert/pia-openvpn](https://hub.docker.com/r/colinhebert/pia-openvpn/)
Has a firewall/killswitch configured, so your information cannot be leaked if the vpn connection fails.

## Improvements
* Updated Dockerfile to fetch new configuration directly from PIA on each build
* Changed openvpn.sh so it works with Fedora
* Updated README
* Added firewall/killswitch support
* Automatically creates tunnel device
* Uses PIA NextGen Servers
* Ability to specify network interface to use

# What is Private Internet Access
Private Internet Access VPN Service encrypts your connection and provides you with an anonymous IP to protect your privacy.

# How to use this image
This image provides the configuration file for each region managed by PIA.

The goal is to start this container first then run other container within the PIA VPN via `--net=container:pia`.


## Starting the client
```Shell
docker run --cap-add=NET_ADMIN --name=pia -d \
  --restart=always \
  --dns 209.222.18.222 --dns 209.222.18.218 \
  -e 'REGION=<region>' \
  -e 'USERNAME=<pia_username>' \
  -e 'PASSWORD=<pia_password>' \
  zwimer/pia-openvpn
```

Substitute the environment variables `REGION`, `USERNAME`, and `PASSWORD` as needed. Note that `REGION` is optional, and defaults to `US East`.

Due to the nature of the VPN client, this container must be started with some additional privileges. `--cap-add=NET_ADMIN` makes sure that the tunnel can be created from within the container.

Starting the container in privileged mode would also achieve this, but keeping the privileges to the minimum required is preferable.

*NOTE:* In most cases, a DNS server must be specified using `--dns <ip-address>`. It is recommended to use the PIA DNS servers, whose addresses are listed above, or can be found on their [client support page](https://www.privateinternetaccess.com/pages/client-support/) under `DNS Leak Protection`.

## Creating a container that uses PIA VPN
```Shell
docker run -it --rm --net=container:pia centos
```

The IP address returned after this execution should be different from the IP address you would get without specifying `--net=container:pia`.

# Advanced usage

## Additional arguments for the openvpn client
Every parameter provided to the `docker run` command is directly passed as an argument to the [openvpn executable](https://community.openvpn.net/openvpn/wiki/Openvpn23ManPage).

This will run the openvpn client with the `--pull` option:
```Shell
docker run ... --name=pia \
  zwimer/pia-openvpn \
    --pull
```

## Avoid passing credentials in environment variables
By default this image relies on the variables `USERNAME` and `PASSWORD` to be set in order to successfully connect to the PIA VPN.

It is possible to use instead a pre-existing volume/file containing the credentials.
```Shell
docker run ... --name=pia \
  -v '<path/to/auth.conf>:/pia/auth.conf' \
  zwimer/pia-openvpn \
    --auth-user-pass auth.conf
```

## Specifying the connection strength
You can also specify the connection strength using the environment variable `CONNECTIONSTRENGTH`. If it is not specified, it defaults to `strong`, but you could alternatively also specify `normal`.

## Connection between containers behind PIA
Any container started with `--net=container:...` will use the same network stack as the PIA container, therefore they will share the same local IP address.

[Prior to Docker 1.9](https://docs.docker.com/engine/userguide/networking/default_network/dockerlinks/) `--link=pia:mycontainer` was the recommended way to connect to a specific container.

[Since Docker 1.9](https://docs.docker.com/engine/userguide/networking/dockernetworks/), it is recommended to use a non default network allowing containers to address each other by name.

## Specifying the network interface
You can also specify the network interface using the environment variable `NETWORKINTERFACE`. If it is not specified, it defaults to `eth0`.

### Creation of a network
```Shell
docker network create pia_network
```

This creates a network called `pia_network` in which containers can address each other by name; the `/etc/hosts` is updated automatically for each container added to the network.

### Start the PIA container in the pia_network
```Shell
docker run ... --net=pia_network --name=pia zwimer/pia-openvpn
```

In `pia_network` there is now a resolvable name `pia` that points to that newly created container.

### Create a container behind the PIA VPN
This step is the same as the earlier one
```Shell
# Create an HTTP service that listens on port 80
docker run ... --net=container:pia --name=myservice myservice
```

This container is not addressable by name in `pia_network`, but given that the network stack used by `myservice` is the same as the `pia` container, they have the same IP address and the service running in this container will be accessible at `http://pia:80`.

### Create a container that access the service
```Shell
docker run ... --net=pia_network tutum/curl curl -s http://pia/
```

The container is started within the same network as `pia` but is not behind the VPN.
It can access services started behind the VPN container such as the HTTP service provided by `myservice`.

# Sample Configuration
This is a sample docker-compose file based on the one which I've been using.
```
version: "2.1"
services:
  pia-vpn:
    container_name: pia-vpn
    image: zwimer/pia-openvpn
    restart: always
    cap_add:
      - net_admin
    dns:
      - 209.222.18.222
      - 209.222.18.218
    ports:
      - 9091:9091
    volumes:
      - /path/on/host:/path/in/container:ro
    environment:
      REGION: "US East"
    command: ["--auth-user-pass", "/path/in/container/auth.conf"]
  transmission:
    container_name: transmission
    image: linuxserver/transmission
    restart: always
    network_mode: "service:pia-vpn"
    depends_on:
      - pia-vpn
    volumes:
      - /path/on/host:/config 
      - /path/on/host:/downloads 
  sabnzbd:
    container_name: sabnzbd
    image: linuxserver/sabnzbd
    restart: always
    network_mode: "service:pia-vpn"
    depends_on:
      - pia-vpn
    volumes:
      - /path/on/host:/config
      - /path/on/host:/downloads 
```

