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

switch port

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

Config du serveur VPN : https://wiki.openwrt.org/doc/howto/vpn.openvpnPour voir les logs: Status /system log.

interet du vpn: Guest network access can easily be granted because you do not need to care about the things your guests are using your Internet for. :)https://blog.ipredator.se/howto/openwrt/configuring-openvpn-on-openwrt.html


# Création des clefs
Le fichier de conf de easy-rsa est dans /etc/easy-rsa/vars.
./clean-all pour tout effacer

Seuls les .key files doivent etre gardé confidentiel.
.crt et .csr peuvent etre transférer par une voie insecure (comme les mails)
On ne doit jamais devoir copier un .key d'un computer à un autre.

-
- 1 Build own root certificat authority / key
   - ./build-ca
   - CA : certificat authority.
   - crée ca.crt et ca.key dans le KEY_DIR definit dans var.
- 2 Buid intermediate certificat authority / key. Optionel
  - ./Build-inter inter
- 3 Build a diffie-helmann parameters necessaire pour la partie server terminale d'une connection SSL/TLS
  - ./build-dh. Au deuxieme essais cette commande a pris un temps fou. Je n'ai pas pu en voir la fin.
ca n'a pas marché avec les commandes ci dessus.


Mais avec :
  
pkitool --initca                      ## equivalent to the 'build-ca' script
pkitool --server my-server            ## equivalent to the 'build-key-server' script
pkitool          my-client            ## equivalent to the 'build-key' script (*not build-key-pkcs12)
openssl dhparam -out dh2048.pem 2048  ## equivalent to the 'build-dh' script

Mais je n'ai pas encore tester le serveur.
logread -f
If everything went fine, the last log line from OpenVPN should contain Initialization Sequence Completed. There are some warnings and errors ... ignore them.

On cherche s'il y a une interface tun avec ifconfig -a
On regarde la table de routage avec netsat -nr. On doit avoir et c'est essentiel une route vers le serveur VPN (xxx.xxx.XXX.XXX), une route 
On fait un traceroute pour voir si on va vers le server vpn.

Quand est ce qu'on donne l'adresse du seveur vpn? dans le fichier de configuration du client /etc/config/openvpn : option remote SERVER_IP_ADRESS 1194
option remote 'pw.openvpn.ipredator.se 1194'


 YOu will need to open port 1194 (TCP) to be forwarding through the firewall to your OpenVPN server.


You can easily find out your OpenVPN server IP address. The syntax is as follows to get tun0 ip address on Unix or Linux:
ifconfig tun0

OR use Linux specific command:
ip a show tun0

dig TXT +short o-o.myaddr.l.google.com @ns1.google.com


# comment tester le server openvpn
- If you don't mind reconfiguring your network, you could also unplug the modem, plug a computer in its place, set the router WAN to a static IP (192.168.64.1) and the computer on 192.168.64.2, and try connecting to the VPN using the .1 IP.
- En utilisant une connection 3G
- En changeant l'option remote pour qu'elle pointe vers l'adresse IP privée du serveur openvpn.
~~~
client
dev tap
proto udp
# remote yourddns.dyndns.org 5712
remote 192.168.1.160 5712
~~~~
Si votre openvpn est votre routeur ce sera l'adresse du routeur 192.168.1.1.

# configuration du client
openvpn client.conf
s'assurer que le lien est up avec `ifconfig tun0`
on doit avoir une ligne du genre : ` inet addr:10.8.0.6  P-t-P:10.8.0.5  Mask:255.255.255.255`
s'assurer que cela fonctionne : `ping 10.8.0.1 -c 2


# configuration du serveur sur une machine qui n'est pas le routeur
fichier /etc/openvpn/server.conf
- directive `local`: to make it listen to an IP adress. C'est l'adress IP vers qui est forwardé UDP port 1194 par le firewall.
- Pour que les remote clients quand ils se connectent sur le server, puissent atteindre autre choses que les que le serveur lui meme il faut activer le ip forwarding avec
- `echo 1 > /proc/sys/net/ipv4/ip_forward`
- ou editer /etc/systcl.conf : `net.ipv4.ip_forward = 1`

On doit pouvoir pinguer n'importe qu'elle machine sur le LAN du server à partir du client. et pouvoir pinguer le client de n'importe qu'elle machine sur le LAN du server

Depuis le client traceroute 192.168.10.65 (ip address LAN du réseau du serveur)
ping 192.168.10.65


# configuration du TPlink en wifi AP.
## Edit the lan interface
### arreter le service DHCP pour la lan interface
### Créer une nouvelle interface wireless
Dans le fichier /etc/config/wireless on a :
- wifi-device qui concerne l'aspect matériel
- wifi-interface qui concerne l'aspect reseau

Pour ce faire network / wireless / add puis :
- device configuration on ne fait rien
- interface configuration:
	- ESSID: un nom qui va bien (ie RADIO_GUEST)
	- Mode:  access point 
	- Network : create : un nom qui va bien (ie IF_GUEST)
	- Save and Apply
On voit que dans le fichier /etc/config/wireless on a juste ajouté:

```
config wifi-iface
	option device 'radio0'
	option mode 'ap'
	option encryption 'none'
	option ssid 'RADIO_GUEST'
	option network 'IF_GUEST'
```

On se retrouve avec deux SSID. On doit donc pouvoir soit à l'un soit à l'autre. 

### configurer cette nouvelle interface
On voit que dans le fichier /etc/config/network ce qui est nouveau c'est :
```
config interface 'IF_GUEST'
	option proto 'none'
```
- network / interface:
- common config
	- protocol : static address . Really switch protocol? switch protocol
	- IPv4 address : une address d'un autre subnet que celle du lan.
	- IPv4 netmask: 255.255.255.0
	- IPv4 gateway : on laisse vide 
- DHCP Server
	- setup DHCP server
	- configurer les addresses ip pour éviter les conflits d'ip
- onglet firewall setting:
	- pour cette nouvelle interface on crée une nouvelle zone. unspecified or create: GUEST_ZONE.  
Ce qui ajoute ceci au fichier /etc/config/firewall

```
config zone
	option name 'GUEST_ZONE'
	option input 'ACCEPT'
	option forward 'REJECT'
	option output 'ACCEPT'
	option network 'IF_GUEST'
```	

### configurer le firewall
network / firewall

