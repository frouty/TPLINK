TPlink Archer C7 V2.0 (c'est marqué sur l'etiquette de la boite)

# First install
login : admin / password : admin
image à installer openwrt-15-05-ar71xx-generic-archer-c7-v2-squashfs-factory.bin


https://wiki.openwrt.org/toh/hwdata/tp-link/tp-link_archer_c7_ac1750_v2.0

https://lede-project.org/toh/hwdata/tp-link/tp-link_archer_c7_ac1750_v2.0

initialement on a le factory firmware : 3.15.1 Build 160616 Rel.44182n 




metre dans /etc/profile:
```
 export PS1='\[\033[35;1m\]\u\[\033[0m\]@\[\033[31;1m\]\h \[\033[32;1m\]$PWD\[\033[0m\] [\[\033[35m\]\#\[\033[0m\]]\[\033[31m\]\$\[\033[0m\] '
 ```
c'est plus jolie.

# switch port

0  eth1  
1  wan  
2  lan1  
3  lan2  
4  lan3  
5  lan4   
6  eth0  

le port wan (je pense que c'est le connecteur rj45 bleu). Il est connecté à eth0

https://wiki.openwrt.org/doc/howto/secure.access


Dans la configuration du routeur a aucun moment je n'ai besoin de donner le ipv4 gateway adresse ni l'adresse du dns server.Je ne comprends pas pourquoi.

pour connaitre les paquets disponibles `opkg search *openvpn*`

`logread -f`

# Premiere chose à faire
désactiver le dhcp sur le lan car sinon il y a des conflits avec l'autre serveur dhcp sur l'autre openwrt.

Config du serveur VPN : https://wiki.openwrt.org/doc/howto/vpn.openvpnPour voir les logs: Status /system log.

interet du vpn: Guest network access can easily be granted because you do not need to care about the things your guests are using your Internet for. :)https://blog.ipredator.se/howto/openwrt/configuring-openvpn-on-openwrt.html

https://wiki.openwrt.org/doc/howto/vpn.openvpn




# installer ohmyzsh en root
sudo su
sh -c "$(wget https://raw.github.com/robbyrussell/oh-my-zsh/master/tools/install.sh -O -)"

# OPENVPN


on a :
```
Wed Jan  3 02:26:14 2018 OpenVPN 2.3.4 x86_64-pc-linux-gnu [SSL (OpenSSL)] [LZO] [EPOLL] [PKCS11] [MH] [IPv6] built on Jun 26 2017
Wed Jan  3 02:26:14 2018 library versions: OpenSSL 1.0.1t  3 May 2016, LZO 2.08
Wed Jan  3 02:26:14 2018 WARNING: file '/etc/openvpn/my-openvpn-client.key' is group or others accessible
Wed Jan  3 02:26:14 2018 Socket Buffers: R=[212992->131072] S=[212992->131072]
Wed Jan  3 02:26:14 2018 UDPv4 link local (bound): [undef]
Wed Jan  3 02:26:14 2018 UDPv4 link remote: [AF_INET]103.17.45.190:1194
Wed Jan  3 02:27:14 2018 TLS Error: TLS key negotiation failed to occur within 60 seconds (check your network connectivity)
Wed Jan  3 02:27:14 2018 TLS Error: TLS handshake failed
Wed Jan  3 02:27:14 2018 SIGUSR1[soft,tls-error] received, process restarting
Wed Jan  3 02:27:14 2018 Restart pause, 2 second(s)
Wed Jan  3 02:27:16 2018 WARNING: file '/etc/openvpn/my-openvpn-client.key' is group or others accessible
Wed Jan  3 02:27:16 2018 Socket Buffers: R=[212992->131072] S=[212992->131072]
Wed Jan  3 02:27:16 2018 UDPv4 link local (bound): [undef]
Wed Jan  3 02:27:16 2018 UDPv4 link remote: [AF_INET]103.17.45.190:1194
Wed Jan  3 02:28:16 2018 TLS Error: TLS key negotiation failed to occur within 60 seconds (check your network connectivity)
Wed Jan  3 02:28:16 2018 TLS Error: TLS handshake failed
Wed Jan  3 02:28:16 2018 SIGUSR1[soft,tls-error] received, process restarting
Wed Jan  3 02:28:16 2018 Restart pause, 2 second(s)
Wed Jan  3 02:28:18 2018 WARNING: file '/etc/openvpn/my-openvpn-client.key' is group or others accessible
Wed Jan  3 02:28:18 2018 Socket Buffers: R=[212992->131072] S=[212992->131072]
Wed Jan  3 02:28:18 2018 UDPv4 link local (bound): [undef]
Wed Jan  3 02:28:18 2018 UDPv4 link remote: [AF_INET]103.17.45.190:1194
Wed Jan  3 02:29:18 2018 TLS Error: TLS key negotiation failed to occur within 60 seconds (check your network connectivity)
Wed Jan  3 02:29:18 2018 TLS Error: TLS handshake failed
Wed Jan  3 02:29:18 2018 SIGUSR1[soft,tls-error] received, process restarting
Wed Jan  3 02:29:18 2018 Restart pause, 2 second(s)
Wed Jan  3 02:29:20 2018 WARNING: file '/etc/openvpn/my-openvpn-client.key' is group or others accessible
Wed Jan  3 02:29:20 2018 Socket Buffers: R=[212992->131072] S=[212992->131072]
Wed Jan  3 02:29:21 2018 UDPv4 link local (bound): [undef]
Wed Jan  3 02:29:21 2018 UDPv4 link remote: [AF_INET]103.17.45.190:1194
```

Si votre openvpn est votre routeur ce sera l'adresse du routeur 192.168.1.1.



# /etc/config/openvpn serveur c'est à dire sur le routeur openwrt.

```
config openvpn 'myvpn'
        option enable '1'
        option verb '3'
        option port '1194'  <== on peut le changer mais il faudra le renseigner dans client.conf.
        option proto 'udp'
        option dev 'tun'
        option server '10.10.0.0 255.255.255.0'
        option keepalive '10 120'
        option ca '/etc/openvpn/ca.crt'
        option dh '/etc/openvpn/dh2048.pem'
        option cert '/etc/openvpn/my_openvpn_server.crt'
        option key '/etc/openvpn/my_openvpn_server.key'
        option log '/tmp/openvpn.log'
        option ifconfig_pool_persit '/tmp/ipp.txt'
        option status '/tmp/openvpn-status.log'
        list push 'route 10.66.0.0 255.255.255.0'
        list push 'dhcp-option DNS 10.66.0.1'
```
```
ls /etc/openvpn  
dh2048.pem
my_openvpn_server.crt
my_openvpn_server.csr
my_openvpn_server.key
```

## Comment redémarrer le service openvpn sur le routeur TPLINK
Il semble que sur le webgui du TPLINK on arrive pas a le redémarrer
### Comment savoir si le service tourne?
- ` ps | grep openvpn` 
- ` ifconfig tun0`  ou ifconfig et on cherche une interface `tun`.
- les fichiers de log :
 - sur les server : /tmp/openvpn.log il est défini dans le fichier de config /etc/easy-rsa/keys
## les fichiers de config est :
/etc/config/openvpn. Il n'est pas dans /etc/openvpn.

## panne
Le fichier /etc/openvpn/ca.crt a disparu sur le router tpLink que faire?  
`cp /etc/easy-rsa/keys/ca.crt /etc/openvp/` 




# /etc/openvpn sur le client linux.
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


remote-cert-tls server
remote goeen.ddns.net 1200  <== on retrouve le changement de port
```
tree /etc/openvpn  sur le client linux.  
/etc/openvpn  
├── ca.crt  
├── client.conf  
├── client.trash  
├── my-openvpn-client.crt  
├── my-openvpn-client.key  
├── openvpn.log  
├── server.trash  
└── update-resolv-conf  


# pour windows 
il faut récupérer les fichiers .crt, .key, les mettre dans C:/Program/openvpn/config avec aussi un client.conf

```
client
dev tun
proto udp

log openvpn.log
verb 3

resolv-retry infinite

ca ca.crt
cert my-openvpn-client.crt
key my-openvpn-client.key


remote-cert-tls server
remote goeen.ddns.net 1200  <== on retrouve le changement de port
```













