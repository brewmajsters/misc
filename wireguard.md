# Wireguard setup

**Problem to solve**
- connect **server** from private net segment with **clients** in other private net segments via **proxy** server with public IP

**Prerequisites**
- this guide assumes you have [wireguard installed](https://www.wireguard.com/install/)
- generate 2 + NUMBER OF CLIENTS pairs of asymentric keys using this command
    ```bash
    wg genkey | tee privatekey | wg pubkey > publickey
    ```

## Configuration

Assume this example for configuration:
```
All in 10.12.13.0/24 subnet
                                  __________________________                       
                                 |                          |                     
                                 |           Proxy          |                      ____________
                                 | (with public IP x.x.x.x) |                     |            |
 ________________                |__________________________|      /___________.2_|  client 1  |
|                |                           |.1                  /               |____________|
|  Raspberry Pi  |_.100______________________|___________________/                              
|________________|                                               \                 ____________ 
                                                                  \               |            |
                                                                   \___________.3_|  client 2  |
                                                                                  |____________|
```
- NOTE: only the Proxy is required to have a public IP
- using these set of keys:

| host | private key | public key |
| :--: | :---------: | :--------: |
| Raspberry Pi | PI_PRIVATE_KEY | PI_PUBLIC_KEY |
| Proxy | PROXY_PRIVATE_KEY | PROXY_PUBLIC_KEY |
| client 1 | C1_PRIVATE_KEY | C1_PUBLIC_KEY |
| client 2 | C2_PRIVATE_KEY | C2_PUBLIC_KEY |


### Raspberry Pi

**/etc/wireguard/wg1.conf**
```ini
[Interface]
Address = 10.12.13.100/24
PrivateKey = PI_PRIVATE_KEY

[Peer]
PublicKey = PROXY_PUBLIC_KEY
AllowedIPs = 10.12.13.0/24
Endpoint = x.x.x.x:30001
PersistentKeepalive = 25
```

### Proxy

**/etc/wireguard/postup.sh**
```bash
iptables -A FORWARD -i wg1 -j ACCEPT
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE

ip6tables -A FORWARD -i wg1 -j ACCEPT
ip6tables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```

**/etc/wireguard/postdown.sh**
```bash
iptables -D FORWARD -i wg1 -j ACCEPT
iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

ip6tables -D FORWARD -i wg1 -j ACCEPT
ip6tables -t nat -D POSTROUTING -o eth0 -j MASQUERADE
```

**/etc/wireguard/wg1.conf**
```ini
[Interface]
Address = 10.12.13.1/24
SaveConfig = true
PostUp = bash /etc/wireguard/postup.sh
PostDown = bash /etc/wireguard/postdown.sh
ListenPort = 30001
PrivateKey = PROXY_PRIVATE_KEY

[Peer]
PublicKey = PI_PUBLIC_KEY
AllowedIPs = 10.12.13.100/32

[Peer]
PublicKey = C1_PUBLIC_KEY
AllowedIPs = 10.12.13.2/32

[Peer]
PublicKey = C2_PUBLIC_KEY
AllowedIPs = 10.12.13.3/32
```

**IMPORTANT:** additional add this iptables rule - `iptables -A FORWARD -i wgX -o wgX -j ACCEPT`

### Client 1/2

**/etc/wireguard/wg1.conf**
```ini
[Interface]
Address = 10.77.10.{2,3}/24
PrivateKey = C{1,2}_PRIVATE_KEY

[Peer]
PublicKey = PROXY_PUBLIC_KEY
AllowedIPs = 10.12.13.0/24
Endpoint = x.x.x.x:30001
PersistentKeepalive = 25
```

## Running

- wireguard interface UP:
    ```bash
    wg-quick up wg1
    ```

- wireguard interface DOWN:
    ```bash
    wg-quick down wg1
    ```
