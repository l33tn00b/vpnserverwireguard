# vpnserverwireguard
VPN sever with wireguard on Ubuntu with ufw

some routing/firewall issue to be resolved. (seems to be specific to my setup. should work for you, though. See: https://community.hetzner.com/tutorials/install-and-configure-wireguard-vpn)

assuming you are root on your server


for transfering config to client
```
apt-get install qrencode
```

set up the wg server to be 10.8.3.1 using subnet 10.8.3.0/24
choose wisely for your MTU (do you have to take into account your 8 bytes for PPPoE?)
```
tee /etc/wireguard/wg0.conf <<END
[Interface]
PrivateKey = $(wg genkey)
Address = 10.8.3.1/24
ListenPort = 51820
MTU = 1400
PostUp = sysctl net.ipv4.ip_forward=1
PostUp = iptables -A FORWARD -i eth0 -o %i -j ACCEPT
PostUp = iptables -A FORWARD -i %i -j ACCEPT
PostUp = iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = sysctl net.ipv4.ip_forward=0
PostDown = iptables -D FORWARD -i eth0 -o %i -j ACCEPT
PostDown = iptables -D FORWARD -i %i -j ACCEPT
PostDown = iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE
END
```

start server
```
systemctl start wg-quick@wg0 && sudo systemctl enable wg-quick@wg0
```

firewall
```
ufw allow 51820/udp
ufw reload
```

dir for client config
```
mkdir ~/wg-clients
```

create client config
replace client01 with client name
```
peers_count="$(sudo grep '\[Peer\]' /etc/wireguard/wg0.conf | wc -l)"
tee ~/wg-clients/client01.conf <<END
[Interface]
PrivateKey = $(wg genkey)
Address = 10.8.3.$(( peers_count + 2 ))/24
DNS = 9.9.9.9
MTU = 1400
END
```

make client known to server
replace client01 by client name chosen above
```
sudo tee -a /etc/wireguard/wg0.conf <<END
[Peer]
PublicKey = $(grep PrivateKey ~/wg-clients/client01.conf | awk '{print $3}' | wg pubkey)
AllowedIPs = $(grep Address ~/wg-clients/client01.conf | awk '{print $3}' | sed 's/\/.*//')/32
END
```
```
sudo systemctl stop wg-quick@wg0 && sudo systemctl start wg-quick@wg0
```

make server known to client
replace client01 by client name chosen above
make sure to heve correct network interface name (e.g. eth0)
```
wgservip="$(ip -4 -br address show dev eth0)" &&
tee -a ~/wg-clients/client01.conf <<END

[Peer]
PublicKey = $(sudo grep PrivateKey /etc/wireguard/wg0.conf | awk '{print $3}' | wg pubkey)
Endpoint = $(awk '{print $3}' <<<"$wgservip" | sed 's/\/.*//'):51820
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
END
```

# Make sure TCP gets told about the MTU
Just in case our clients don't get ICMP messages (maybe because we didn't allow these in our firewall rules)  

insert into `/etc/ufw/after.rules` just before the last `COMMIT`, right after `don't log noisy broadcasts`: 

```
# MSS clamping during TCP 3-Way Handshake
-A ufw-after-forward -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu
```

```
ufw reload
```
