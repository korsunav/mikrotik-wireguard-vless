# mikrotik->wireguard->vless
Selective URL redirection via VLESS in the network on Mikrotik without the ability to use containers at router

Somewhere in internet:
 - 3X-UI on Linux VM with public IP in foreign country

In your intranet: Create VM with V2rayA and WireGuard
 1. Deploy VM on Linux. In this case we will use Oracle Linux 9
 2. Install Docker by this instruction - https://docs.docker.com/engine/install/rhel/
 3. Run container with V2rayA by docker compose:
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
 4. Run container with wg-easy also by docker compose
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
Configure V2rayA

Configure WireGuard

Configure Mikrotik
NAT rule for VPN interface
add action=masquerade chain=srcnat out-interface=wg0
