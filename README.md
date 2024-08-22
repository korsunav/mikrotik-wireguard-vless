# mikrotik->wireguard->vless
Selective URL redirection via VLESS in the network on Mikrotik without the ability to use containers at router

![screenshot](img/mikrotik_v2raya.drawio.svg)

Somewhere in internet:
 - 3X-UI on Linux VM with public IP in foreign country

In your intranet: Create VM with V2rayA and WireGuard
 1. Deploy VM on Linux. In this case we will use Oracle Linux 9
 2. Install Docker by this instruction - https://docs.docker.com/engine/install/rhel/
 3. Run container with V2rayA by docker compose:
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
      - V2RAYA_V2RAY_BIN=/usr/local/bin/xray #use same core as on server side (3x-ui in this case)
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

 4. Run container with wg-easy also by docker compose
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

Configure V2rayA
 1. Go to `<Your VM ip address>:2017`
 2. Import configuration from 3X-UI and run proxy
 3. Go to Settings and set next options:
    - `Transparent Proxy/System Proxy: On do not split traffic`
    - IP Forward: `active`
    - Port Sharing: `active`
    - `Transparent Proxy/System Proxy Implementation: redirect`
    - `Traffic Splitting Mode of Rule Port: RoutingA`
      - click `Configure` and leave just 1 rule: `default: proxy`
    - everything else is by default
 4. click `Save and Apply`

Configure WireGuard
 1. Go to `<Your VM ip address>:51821`
 2. Click `+ New` input name and click `create`
 3. Download configuration file of created connection

Configure Mikrotik
 1. Upgrade RouterOS to 7.5+ firmware. Last stable is prefered
 2. Click on `WireGuard` button
 3. Click on `WG Import` and choose downloaded configuration file from WireGuard
 4. Go to `Peers` section and double click on just added connection. Enter `<Your VM ip address>` to `Endpoint` and `51820` to `Endpoint port`
 5. Create NAT rule for VPN interface through terminal: `/ip firewall nat add action=masquerade chain=srcnat out-interface=wg0`
 6. Add new address of WireGuard to adress list: `/ip address add address=<Your WG ip address> interface=wg0 network=<Your WG network>`
    e.g `/ip address add address=10.8.0.2/24 interface=wg0 network=10.8.0.0`
 7. Add domains to Address list
    1. Create Routing table: `/routing table add disabled=no fib name=to-proxy`
    2. Add route: `/ip route add comment=vpn disabled=no distance=1 dst-address=0.0.0.0/0 gateway=wg0 pref-src="" routing-table=to-proxy`
    3. Create marking: `/ip firewall mangle add action=mark-routing chain=prerouting disabled=no dst-address-list=vpn-domains new-routing-mark=to-proxy passthrough=yes`
    4. Add nessesary domains to DNS static list for forwarding through proxy:
       - one by one: `/ip dns static add name=terraform.io type=FWD forward-to=8.8.8.8 address-list=vpn-domains match-subdomain=yes`
       - or generate commands from domain list:
         
         `wget -qO- https://raw.githubusercontent.com/itdoginfo/allow-domains/main/Russia/inside-raw.lst | sed "s/.*/\/ip dns static add name=& type=FWD forward-to=8.8.8.8 address-list=vpn-domains match-subdomain=yes/"`

         and paste all of them in to the terminal

That's all 
