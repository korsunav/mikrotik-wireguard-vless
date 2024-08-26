# Mikrotik -> WireGuard -> VLESS

**Selective URL redirection via VLESS on Mikrotik without container support.**

![Network Diagram](img/mikrotik_v2raya.drawio.svg)

## Requirements

- **3X-UI** on a Linux VM with a public IP address in a foreign country.

## Setup within Your Intranet

### 1. Create a Virtual Machine with V2rayA and WireGuard

#### 1.1. Deploy the VM on Linux

For this guide, Oracle Linux 9 will be used. Please turn off SELinux and Firewalld during setup, it will probably save your time.

#### 1.2. Install Docker

Follow this guide to install Docker: [Docker Installation Guide for RHEL](https://docs.docker.com/engine/install/rhel/).

#### 1.3. Run the V2rayA Container using Docker Compose

<details>
<summary>v2raya_docker-compose.yaml</summary>
 
```yaml
 services:
  v2raya:
    restart: always
    privileged: true
    network_mode: host
    container_name: v2raya
    environment:
      - V2RAYA_V2RAY_BIN=/usr/local/bin/xray
      - V2RAYA_LOG_FILE=/tmp/v2raya.log
      - V2RAYA_NFTABLES_SUPPORT=off
      - IPTABLES_MODE=legacy
      - V2RAYA_VERBOSE=true
    volumes:
      - '/etc/v2raya:/etc/v2raya'
      - '/etc/resolv.conf:/etc/resolv.conf'
      - '/lib/modules:/lib/modules:ro'
    image: 'mzz2017/v2raya:latest'
```

</details>

#### 1.4. Run the wg-easy Container using Docker Compose

<details>
<summary>wg-easy_docker-compose.yaml</summary>

```yaml
volumes:
  etc_wireguard:

services:
  wg-easy:
    environment:
      # Change Language:
      # (Supports: en, ua, ru, tr, no, pl, fr, de, ca, es, ko, vi, nl, is, pt, chs, cht, it, th, hi)
      - LANG=en
      # ⚠️ Required:
      # Change this to your host's public address
      - WG_HOST=192.168.88.112

      # Optional:
      # - PASSWORD_HASH=$$2y$$10$$hBCoykrB95WSzuV4fafBzOHWKu9sbyVa34GJr8VV5R/pIelfEMYyG (needs double $$, hash of 'foobar123'; see "How_to_generate_an_bcrypt_hash.md" for generate the hash)
      # - PORT=51821
      # - WG_PORT=51820
      # - WG_CONFIG_PORT=92820
      # - WG_DEFAULT_ADDRESS=10.8.0.x
      # - WG_DEFAULT_DNS=1.1.1.1
      # - WG_MTU=1420
      # - WG_ALLOWED_IPS=192.168.88.0/24, 10.0.8.0/24
      # - WG_PERSISTENT_KEEPALIVE=25
      # - WG_PRE_UP=echo "Pre Up" > /etc/wireguard/pre-up.txt
      # - WG_POST_UP=echo "Post Up" > /etc/wireguard/post-up.txt
      # - WG_PRE_DOWN=echo "Pre Down" > /etc/wireguard/pre-down.txt
      # - WG_POST_DOWN=echo "Post Down" > /etc/wireguard/post-down.txt
      # - UI_TRAFFIC_STATS=true
      # - UI_CHART_TYPE=0 # (0 Charts disabled, 1 # Line chart, 2 # Area chart, 3 # Bar chart)

    image: ghcr.io/wg-easy/wg-easy
    container_name: wg-easy
    volumes:
      - etc_wireguard:/etc/wireguard
    ports:
      - "51820:51820/udp"
      - "51821:51821/tcp"
    restart: unless-stopped
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
      # - NET_RAW # ⚠️ Uncomment if using Podman 
    sysctls:
      - net.ipv4.ip_forward=1
      - net.ipv4.conf.all.src_valid_mark=1
```

</details>

### 2. Configure V2rayA

1. Go to `<Your VM IP address>:2017`.
2. Import the configuration from 3X-UI and start the proxy.
3. In the settings, configure the following options:
    - `Transparent Proxy/System Proxy`: On, do not split traffic.
    - `IP Forward`: Active.
    - `Port Sharing`: Active.
    - `Transparent Proxy/System Proxy Implementation`: Redirect.
    - `Traffic Splitting Mode of Rule Port`: RoutingA.
        - Click `Configure` and keep only one rule: `default: proxy`.
    - Leave everything else as default.
4. Click `Save and Apply`.

### 3. Configure WireGuard

1. Go to `<Your VM IP address>:51821`.
2. Click `+ New`, input a name, and click `Create`.
3. Download the configuration file for the created connection.

### 4. Configure Mikrotik

1. Upgrade RouterOS to firmware version 7.5+ (the latest stable version is preferred).
2. Click the `WireGuard` button.
3. Click `WG Import` and select the downloaded configuration file from WireGuard.
4. Go to the `Peers` section and double-click the newly added connection. Enter `<Your VM IP address>` in `Endpoint` and `51820` in `Endpoint port`.
5. Create a NAT rule for the VPN interface via the terminal:

    ```bash
    /ip firewall nat add action=masquerade chain=srcnat out-interface=wg0
    ```

6. Add a new WireGuard address to the address list:

    ```bash
    /ip address add address=<Your client WG IP with CIDR> interface=<Your WG interface> network=<Your WG network>
    ```

    Example:

    ```bash
    /ip address add address=10.8.0.2/24 interface=wg0 network=10.8.0.0
    ```

7. Add domains to the Address list:

    1. Create a routing table:

        ```bash
        /routing table add disabled=no fib name=to-proxy
        ```

    2. Add a route:

        ```bash
        /ip route add comment=vpn disabled=no distance=1 dst-address=0.0.0.0/0 gateway=wg0 pref-src="" routing-table=to-proxy
        ```

    3. Create a marking rule:

        ```bash
        /ip firewall mangle add action=mark-routing chain=prerouting disabled=no dst-address-list=vpn-domains new-routing-mark=to-proxy passthrough=yes
        ```

    4. Add necessary domains to the DNS static list for forwarding through the proxy:
        - One by one:

            ```bash
            /ip dns static add name=terraform.io type=FWD forward-to=8.8.8.8 address-list=vpn-domains match-subdomain=yes
            ```

        - Or generate commands from a domain list:

            ```bash
            wget -qO- https://raw.githubusercontent.com/itdoginfo/allow-domains/main/Russia/inside-raw.lst | sed "s/.*/\/ip dns static add name=& type=FWD forward-to=8.8.8.8 address-list=vpn-domains match-subdomain=yes/"
            ```

            And paste them all into the terminal.

---

That's all!
