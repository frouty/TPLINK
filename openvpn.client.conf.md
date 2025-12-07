# Comment configurer le client sur linux
- 1 installation d'un fichier de config dans /etc/openvpn/
client.conf :
```
client
dev tun
proto udp

log openvpn.log
verb 3

resolv-retry infinite

ca /etc/openvpn/ca.crt
cert /etc/openvpn/my-openvpn-client.crt
key /etc/openvpn/my-openvpn-client.key
```
TODO pour le reste


# comment configurer le client sur windows
- 1 installer l'installer.




# Comment installer un client openvpn sur le router.

https://openwrt.org/docs/guide-user/services/vpn/openvpn/client