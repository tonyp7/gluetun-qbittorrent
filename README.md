# Containerized qBittorrent with Gluetun VPN Client

This repository demonstrates how you can combine two popular Docker images together:
 - [Gluetun VPN client](https://github.com/qdm12/gluetun) Lightweight swiss-knife-like VPN client to multiple VPN service providers
 - [Qbittorrent](https://github.com/linuxserver/docker-qbittorrent) Docker image for Qbittorrent maintained by linuxserver.io

# How to use it

You will need to configure the gluetun client so that it can connect to the VPN provider of your choice. Glutun has a great [Wiki](https://github.com/qdm12/gluetun-wiki) on how to do that.

This mean you will need to edit the `docker-compose.yml` file to fit your needs.

Then, you can simply run:

```console
docker compose up -d
```

As per the notes from the linuxserver.io team, a random generator for the admin panel of Qbittorrent will be generated at launch, for you to change.

![image](https://tonypottier.com/wp-content/uploads/2024/01/qbittorrentlog.png)

### Example: AirVPN

A typical docker-compose to AirVPN, using OpenVPN and a connection to a Netherlands server should like below. Note that the gluetun configuration would be place in a subfolder where the docker-compose file resides.

```yaml
version: "3"
services:
  gluetun:
    image: qmcgaw/gluetun
    container_name: gluetun
    cap_add:
      - NET_ADMIN
    devices:
      - /dev/net/tun:/dev/net/tun
    ports:
      - 8080:8080 # Qbittorent
    volumes:
      - ./gluetun:/gluetun
    environment:
      - VPN_SERVICE_PROVIDER=airvpn
      - VPN_TYPE=openvpn
      - SERVER_COUNTRIES=Netherlands
      - UPDATER_PERIOD=
  qbittorrent:
    image: lscr.io/linuxserver/qbittorrent:latest
    container_name: qbittorrent
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Etc/UTC
      - WEBUI_PORT=8080
    volumes:
      - ./qbittorrent/config:/config
      - ./qbittorrent/downloads:/downloads
    restart: unless-stopped
    network_mode: "service:gluetun"
```

In the `./gluetun` folder, the private key `./gluetun/client.key` and client certificate `./gluetun/client.crt` should be populated by the values obtained from your VPN provider. In the case of AirVPN, this would be under Client Area/Config Generator. Pick OpenVPN UDP/Linux and it'll generate a ovpn file containing both keys that should be copy/pasted in their respective Gluetun `client.key` and `client.crt` files. As this is very provider specific, you should refer to the excellent [Gluetun documentation](https://github.com/qdm12/gluetun-wiki) and to your VPN provider for more details.

# Note: configure Qbittorent to bind to VPN

In the Qbittorent client, you should make sure that you are only allowing connection from the `tun` network interface. To do this, you need to bind Qbittorent to the `tun` network interface. Effectively, Qbittorent will not send network packets should the VPN die. 

![image](https://tonypottier.com/wp-content/uploads/2024/01/qbittorrentbindtonetworkinterface.png)

# Note: confirm you are connect to a VPN

From within the qBittorent container, you can run the following command:

```console
curl -L https://ipleak.net/json
```

This should output a json similar to this:

```json
{
    "as_number": 49453,
    "isp_name": "Global Layer B.V.",
    "country_code": "NL",
    "country_name": "Netherlands",
    "continent_code": "EU",
    "continent_name": "Europe",
    "city_name": "Haarlem",
    "postal_code": null,
    "postal_confidence": null,
    "latitude": "51.866729736328125",
    "longitude": "4.662680149078369",
    "accuracy_radius": 1,
    "time_zone": "Europe\/Amsterdam",
    "metro_code": null,
    "level": "min",
    "country_confidence": 100,
    "city_confidence": 100,
    "region_confidence": 100,
    "cache": 1705033492,
    "ip": "213.152.161.15",
    "type": "AirVPN Server (Exit, Situla)",
    "reverse": "",
    "query_text": "213.152.161.15",
    "query_type": "myip",
    "query_date": 1705033492
}
```
If it returns the VPN public IP, that means the configuration is successful.

