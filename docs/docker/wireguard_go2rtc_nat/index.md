# Stream distant camera using Go2RTC and WireGuard

## Project Overview

### Introduction

The aim of this project is to enable real-time streaming of a remote outdoor camera, viewable on an iPhone within the Home app.

The solution was developed through discussions with colleagues and insights from Reddit, leading to the selection of specific tools.

### Approach

This setup involves two sites:

- **Site A (Remote Location):** Includes a Hikvision Camera and a Raspberry Pi 3.
- **Site B (Home):** Consists of an Ubuntu 22.04 VM with Docker and Home Assistant VM hosted on Proxmox server.

The connection between these sites will be established through a VPN, for which Wireguard is the chosen solution.

### System Architecture

[![System Architecture](Schema.svg){: style="width:1000px"}](Schema.svg)

## Implementation Steps

### Setting Up Site A

1. **Install and Configure WireGuard on Raspberry Pi:**
    - Use [PiVPN](https://pivpn.io/) for easy setup.
    - Follow the provided documentation for a smooth installation.

2. **WireGuard Server Configuration:**
    - The configuration file (`/etc/wireguard/wg0.conf`) should be set up as follows:

    ```title="/etc/wireguard/wg0.conf"
    [Interface]
    Address = 10.83.153.1/24
    MTU = 1480
    SaveConfig = true
    PostUp = iptables -A FORWARD -i %i -j ACCEPT; iptables -A FORWARD -o %i -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
    PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -D FORWARD -o %i -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE
    ListenPort = 51879
    PrivateKey = XXX

    [Peer]
    PublicKey = XXX
    AllowedIPs = 10.83.153.3/32
    PersistentKeepalive = 25
    ```

!!! warning

    Before updating your wireguard configuration ensure that server is stopped
    `wg-quick down wg0`

### Setting Up Site B

1. **Deploy Docker-Compose Stack:**
    - The stack includes Wireguard, Go2RTC, and Autoheal containers.
    - The `docker-compose.yml` file should be configured as shown:

    ```yaml title="docker-compose.yml"
    version: "3"
    services:
        wireguard:
          image: linuxserver/wireguard
          container_name: wireguard
          cap_add:
            - NET_ADMIN
            - SYS_MODULE
          sysctls:
            - net.ipv4.conf.all.src_valid_mark=1
          environment:
            - PUID=1000
            - PGID=1000
            - TZ=Europe/Paris
          volumes:
            - /path/to/wireguard/config:/config
          restart: always
          healthcheck:
            test: ["CMD", "curl", "--fail", "http://192.168.1.201"]
            interval: 5s
            retries: 10
            start_period: 30s
            timeout: 5s
          labels:
            - autoheal=true
          networks:
            - vpn

        go2rtc:
          image: alexxit/go2rtc
          restart: unless-stopped
          environment:
            - TZ=Europe/Paris
          ports:
            - "1984:1984"
            - "8554:8554"
          depends_on:
            - wireguard
          networks:
            - vpn
          volumes:
            - "/path/to/go2rtc/config:/config"

        autoheal:
          restart: always
          image: willfarrell/autoheal
          environment:
            - AUTOHEAL_CONTAINER_LABEL=all
          volumes:
            - /var/run/docker.sock:/var/run/docker.sock:ro

    networks:
        vpn:
            name: vpn
            driver: bridge
    ```

2. **WireGuard Client Configuration:**
    - Configure the tunnel to facilitate communication between Frigate and the camera through Wireguard.
    - Example `wg0.conf`:

    ``` title="wg0.conf"
    [Interface]
    PrivateKey = XXX
    Address = 10.83.153.3/24
    MTU = 1480

    PostUp = iptables -t nat -A PREROUTING -p tcp --dport 554 -j DNAT \
    --to-destination 192.168.1.201:554
    PostUp = iptables -t nat -A POSTROUTING -j MASQUERADE

    PostDown = iptables -t nat -D PREROUTING -p tcp --dport 554 -j DNAT \
    --to-destination 192.168.1.201:554
    PostDown = iptables -t nat -D POSTROUTING -j MASQUERADE

    [Peer]
    PublicKey = XXX
    Endpoint = IP:PORT or DNS:PORT
    AllowedIPs = 192.168.1.201/32, 10.83.153.1/32
    PersistentKeepalive = 25
    ```

    !!! info

        I've set MTU value to 1480 because distant connection is slow (~ 8 mbps)


3. **Deploy the Stack:**
    - Run `docker compose up -d` in the directory containing `docker-compose.yml`.

4. **Configure GO2RTC on Ubuntu VM:**
    - Access the Go2RTC configuration page (`http://serverip:1984`).
    - Click on `Config` tab and write something like this :

    ```yaml title="GO2RTC Configuration"
    log:
      rtsp: info

    streams:
      mystream: rtsp://camera:password@wireguard/Streaming/Channels/101

    rtsp:
      listen: ":8554"       # RTSP Server TCP port, default - 8554
      username: "admin"     # optional, default - disabled
      password: "pass"      # optional, default - disabled
      default_query: "mp4"  # optional, default codecs filters
    ```

5. **Integrating with Home Assistant:**
    - Ensure the go2rtc addon is installed: [![](https://my.home-assistant.io/badges/supervisor_addon.svg)](https://my.home-assistant.io/redirect/supervisor_addon/?addon=a889bffc_go2rtc&repository_url=https%3A%2F%2Fgithub.com%2FAlexxIT%2Fhassio-addons)
    - Go to the go2rtc config and write something like this:
    ```yaml
    streams:
      mystream: rtsp://admin:pass@192.168.0.203:8554/mamie

    homekit:
      mamie:
        pin: 12345678
        name: MY CAMERA
        device_id: my_camera_1
        device_private: my_camera_1
    ```
    - Now add the camera to the HomeKit Bridge configuration: [![HomeKit Integration.](https://my.home-assistant.io/badges/integration.svg)](https://my.home-assistant.io/redirect/integration/?domain=homekit)

## Conclusion

With these steps, you should be able to stream your remote camera in real-time and view it in your Home app.
