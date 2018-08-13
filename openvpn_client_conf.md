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
- Récuperer les fichiers du routeur openwrt dans /etc/openvpn/:  
- ca.crt, 
- my_openvp_server.crt,
- my_openvpn_server.key
sur clef usb et les copier dans /C:/program/openvpn/config/

Cela ne marche pas.   j'ai un message TLS key negociation failed to occur within 60 seconds


https://support.eapps.com/index.php?/Knowledgebase/Article/View/451/55/user-guide---openvpn-client-configuration#client-file-configuration

https://opengear.zendesk.com/hc/en-us/articles/216374963-Configuring-a-Windows-OpenVPN-client-or-server

https://support.safervpn.com/hc/en-us/articles/115003743129-OpenVPN-Setup-on-OpenWRT-Router : tres detaille pour la config de l'openvpn server sur openwrt. Oui mais c'est pour configurer en client le routeur.



# Comment installer un client openvpn sur le router.

https://openwrt.org/docs/guide-user/services/vpn/openvpn/client
