# mikrotik->wireguard->vless
Selective URL redirection via VLESS in the network on Mikrotik without the ability to use containers

3X-UI on VM in foreign country

In your intranet
Create VM with V2rayA in Redirect mode:
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
 4. 

NAT rule for VPN interface
add action=masquerade chain=srcnat out-interface=wg0
