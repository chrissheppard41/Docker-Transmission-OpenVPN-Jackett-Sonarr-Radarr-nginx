# Docker-Transmission-OpenVPN-Jackett-Sonarr-Radarr-Lidarr-LazyLibrarian-nginx

## Requirements

* Docker (Current build for this Docker version 18.03.0-ce, build 0520e24)
* Docker-compose (Current build for this docker-compose version 1.20.1, build 5d8c71b)
* An active PIA account

## Introduction

A simple network setup that has Jackett and Transmission route their traffic through openVPN while exposing Radarr (http://<localhost-ip>:7878), Sonarr (http://<localhost-ip>:8989), Lidarr (http://<localhost-ip>:8686), LazyLibrarian (http://<localhost-ip>:5299) and nginx to your local network. Nginx acts like a proxy for both Jackett (http://<localhost-ip>:8081) and Transmission (http://<localhost-ip>:8080).


## Install

To run this here is what you need to do (Note that you need to configure your path directories to your servers path, i've included what they should look like, feel free to update):

1. Create folders in `/opt`: <br>
 A) `mkdir /opt/transmission` <br>
 B) `mkdir /opt/jackett` <br>
 C) `mkdir /opt/sonarr` <br>
 D) `mkdir /opt/radarr` <br>
 E) `mkdir /opt/lidarr` <br>
 F) `mkdir /opt/lazylibrarian`


2. Chown the folders above to the user id that corresponds to the data in your .env file: <br>
 A) `chown -R <user>:<group> /opt/transmission` <br>
 B) `chown -R <user>:<group> /opt/jackett` <br>
 C) `chown -R <user>:<group> /opt/sonarr` <br>
 D) `chown -R <user>:<group> /opt/radarr` <br>
 E) `chown -R <user>:<group> /opt/lidarr` <br>
 F) `chown -R <user>:<group> /opt/lazylibrarian`

3. Clone this information out into a folder of your choice then CD into it

4. Copy the .env.example and rename it to .env `cp .env.example .evn`. You can change your user and group ID in this file to whatever user you wish. 1000 is the default. 0 is root. Docker will create this user ID when the containers build. Make sure that user and group id has access to the folders:
* /opt/transmission
* /opt/jackett
* /opt/sonarr
* /opt/radarr
* /opt/lidarr
* /opt/lazylibrarian
* <path/to/your/downloads folder/>
* <path/to/your/watch folder/>
* <path/to/your/TV folder/>
* <path/to/your/Movies folder/>
* <path/to/your/Music folder/>
* <path/to/your/Books folder/>

5. Copy the .evn_openvpn.example and rename it to .evn_openvpn `cp .evn_openvpn.example .evn_openvpn`.

6. Update service `vpn` env file (.env_openvpn) by adding in your user details by replacing <Username> and <Password> with the details PIA provide

```
REGION='Sweden'
USERNAME='<Username>'
PASSWORD='<Password>'
```

7. Update the service `Transmission` volume path directories

```
volumes:
  - /opt/transmission/:/config
  - <path/to/your/downloads folder/>:/downloads
  - <path/to/your/watch folder/>:/watch
```
By changing these folders you will find that the other containers folders must change, it's easy enough but just be careful because any disconnects will cause errors

8. Update the service `Jackett` volume path directories

```
volumes:
  - /opt/jackett/:/config
  - <path/to/your/watch folder/>:/downloads
  - /etc/localtime:/etc/localtime:ro
```

9. Update the service `Sonarr` volume path directories
```
volumes:
  - /opt/sonarr/:/config
  - <path/to/your/TV folder/>:/tv
  - <path/to/your/downloads/complete folder/>:/downloads/complete/
  - /etc/localtime:/etc/localtime:ro
```

10. Update the service `Radarr` volume path directories
```
volumes:
  - /opt/radarr/:/config
  - <path/to/your/Movies folder/>:/movies
  - <path/to/your/downloads/complete folder/>:/downloads/complete/
  - /etc/localtime:/etc/localtime:ro
```

11. Update the service `Lidarr` volume path directories
```
volumes:
  - /opt/lidarr/:/config
  - <path/to/your/Music folder/>:/movies
  - <path/to/your/downloads/complete folder/>:/downloads/complete/
  - /etc/localtime:/etc/localtime:ro
```

12. Update the service `LazyLibrarian` volume path directories
```
volumes:
  - /opt/lazylibrarian/:/config
  - <path/to/your/Books folder/>:/movies
  - <path/to/your/downloads/complete folder/>:/downloads/complete/
```

For steps 9 10 11 and 12, Transmission has a downloads folder, and inside that there are incomplete and complete folders, you will want to point your download complete folder at that folder structure, also mirror your downloads folder like above `/downloads/complete/` else Sonarr Radarr Lidarr and lazylibrarian wont be able to see the downloaded files.


So the Tranmission's container will have a folder structure like this <br>
`/downloads` points to hosts <path/to/your/downloads/> It will contain a `complete` and `incomplete` folder <br>
`/watch` points to hosts <path/to/your/watch/>

Radarr's, Sonarr's, Lidarr's and LazyLibrarian's download folder structure should be <br>
`/downloads/complete` points to hosts <path/to/your/downloads/complete folder/>

Jackett's folder structure <br>
`/downloads` points to hosts <path/to/your/watch/>


11. Boot `docker-compose up -d`

Hopefully you wont see any failures

12. If you check your hosts child folders of this directory you will see populated config files, take a look at them. Transmission has a full series of configurations you do, you can also update parts of your configuration via the web GUI but read up first on the Transmission page first before making any changes (remember updating your download folders will result in you having to update Sonarr, Radarr and Lidarr path to downloaded files). Side note, don't mess around with settings you are not familiar with, you might damage your configuration so much that you can't repair and you will have to start again with that container.

## How it works

Sonarr, Radarr, Lidarr, LazyLibrarian and nginx will be exposed to your hosts on the following ports 8080, 8081, 7878 and 8989.

If you see in each service
```
networks:
  - vpn-network
```

This is the network these containers will belong to. Pretty simple.

The tricky part is the OpenVPN connection.

Firstly the vpn service will have these variables:

```
cap_add:
  - NET_ADMIN # Giving this container privilege access on this network
network_mode: server_default # Telling this container that it's main network is the server default (can be any name)
```
(network_node is also known as --net) <br>
https://docs.docker.com/compose/compose-file/#cap_add-cap_drop <br>
https://docs.docker.com/compose/compose-file/#network_mode

Transmission and Jackett will have these variables:

```
depends_on:
  - vpn # Tells these containers that the vpn container must first be active before adding these networks
network_mode: service:vpn # Important: Telling the network traffic to route through the service vpn
```

Docker magic

https://docs.docker.com/compose/compose-file/#depends_on <br>
https://docs.docker.com/network/ <br>
https://docs.docker.com/network/bridge/ <br>
https://docs.docker.com/compose/compose-file/ <br>
https://docs.docker.com/compose/networking/


## Health check

Make sure that your Radarr, Sonarr, Lidarr, LazyLibrarian are running and their ports 7878, 8989 and 8686 are running respectively

Nginx should be running on ports 8080 and 8081 (test those addresses in the browser http://<localhost-ip>:8080 (transmission) and http://<localhost-ip>:8080 (jackett))

Run the network inspect on the server_default `docker network inspect server_default`

It should look like this (please note that your variables like subnet and gateways may differ but should be consistent on your build):

```
[
    {
        "Name": "server_default",
        "Id": "c48b991a4c73c63641919846fb76db5075883c72f1bacc71ac2191ba99fb0288",
        "Created": "2018-03-28T20:25:46.588330755+01:00",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "172.18.0.0/16",
                    "Gateway": "172.18.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {
            "52dba1e01e7f44bee405ec2d33c1a3f475aef94849e0e09979da12851e9b2ad9": {
                "Name": "nginx",
                "EndpointID": "ff9befd51ce004a9cc536afcdfeaa02d511d825c446124357df778cfdd34d22f",
                "MacAddress": "02:42:ac:12:00:02",
                "IPv4Address": "172.18.0.2/16",
                "IPv6Address": ""
            },
            "abcc9d68dc62243a29d56f196ebd28181e6f1179d3f3b6d0addc8d58c095f8ff": {
                "Name": "sonarr",
                "EndpointID": "55d0d81870f6a82a5acf92cd6ab88a6551fd86203a6cb4fb3064f231e22ca95f",
                "MacAddress": "02:42:ac:12:00:03",
                "IPv4Address": "172.18.0.3/16",
                "IPv6Address": ""
            },
            "bc0ea23dfb344afe4d86f7ad7b503191cbe9927c99ec4de236a6ec532f18c640": {
                "Name": "pia",
                "EndpointID": "5ab221f9db2637beeaff2a5430ad962ad8c50ce2fcfd5b1eb803c97b0b833511",
                "MacAddress": "02:42:ac:12:00:04",
                "IPv4Address": "172.18.0.4/16",
                "IPv6Address": ""
            },
            "f21830ad65692b96511cbd2cea53983a650804d7026106823df7d7641a55493b": {
                "Name": "radarr",
                "EndpointID": "a0af86086cbbba6a5f11eccce85cea94517055eb4b7e3be0e8569aaa1a3986fd",
                "MacAddress": "02:42:ac:12:00:05",
                "IPv4Address": "172.18.0.5/16",
                "IPv6Address": ""
            }
        },
        "Options": {},
        "Labels": {}
    }
]
```
With Nginx, Sonarr, Pia and Radarr exposed. You should not see Jackett and Transmission in this list.

You can ssh into the Jackett, Transmission and PIA containers, Their IPs should be the same:

1. Execute `docker exec -it pia bash`
2. Execute `ifconfig`
3. Check the eth0 inet IP (your ip should match the network inspection for PIA)
4. Execute `exit`

Then

1. Execute `docker exec -it transmission bash`
2. Execute `ifconfig`
3. Check the eth0 inet IP
4. Execute `exit`

The IPs both should match


Jackett, Transmissions and OpenVpn shouldn't have any ports assigned to them.

## Configure

Sonarr:<br>
https://github.com/Sonarr/Sonarr/wiki <br>
https://sonarr.tv/

Radarr:<br>
https://github.com/Radarr/Radarr/wiki <br>
https://radarr.video/

Lidarr:<br>
https://github.com/lidarr/Lidarr/wiki <br>
http://lidarr.audio/

LazyLibrarian:<br>
https://lazylibrarian.gitlab.io/

Transmission:<br>
https://transmissionbt.com/

Jackett:<br>
https://github.com/Jackett/Jackett/wiki

PIA:<br>
https://www.privateinternetaccess.com/

## Nginx proxy

The nginx acts like a proxy to give access to the Transmission and Jackett on the network. It points to the PIA container, if you look in the nginx/nginx.conf, you will see a very simple conf file for nginx that listens on ports 8080 and 8081 and uses nginx's proxy_pass pointing at `http://pia:9091` (Transmissions native port) or `http://pia:9117` (Jacketts native port).

https://docs.nginx.com/nginx/admin-guide/web-server/reverse-proxy/

## Useful commands

`docker exec -it <container name> bash` To access each of the containers

`docker network ls`

`docker network inspect <name of the network>` (typically "server_default")

`docker ps` (you should see 6 containers running)

`curl ipinfo.io/ip` (to get your IP of your host or ssh into the containers then execute command to check that you are on the VPN)

Transmission sometimes doesn't play ball the first time you boot up as you might see error with permissions so to solve this you can either set the PGID, PUID of transmission to 0 (root) or keep it to 1000 (default user):
1. Execute `docker exec -it transmission bash`
2. Run `chown -R abc:users /config`
3. Execute `exit`
4. Reboot the transmission container by stopping it `docker stop transmission` then `docker-compose up -d`

### Other reading material to expand on this further

itsdaspecialk/pia-openvpn: https://hub.docker.com/r/itsdaspecialk/pia-openvpn/ <br>
linuxserver/transmission: https://hub.docker.com/r/linuxserver/transmission/ <br>
linuxserver/jackett: https://hub.docker.com/r/linuxserver/jackett/ <br>
nginx: https://hub.docker.com/_/nginx/ <br>
linuxserver/sonarr: https://hub.docker.com/r/linuxserver/sonarr/ <br>
linuxserver/radarr: https://hub.docker.com/r/linuxserver/radarr/ <br>
linuxserver/lidarr: https://hub.docker.com/r/linuxserver/lidarr/ <br>
linuxserver/lazylibrarian: https://hub.docker.com/r/linuxserver/lazylibrarian/ <br>
