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

# Premiere chose à faire
désactiver le dhcp sur le lan car sinon il y a des conflits avec l'autre serveur dhcp sur l'autre openwrt.

Config du serveur VPN : https://wiki.openwrt.org/doc/howto/vpn.openvpnPour voir les logs: Status /system log.

interet du vpn: Guest network access can easily be granted because you do not need to care about the things your guests are using your Internet for. :)https://blog.ipredator.se/howto/openwrt/configuring-openvpn-on-openwrt.html


# Création des clefs/certificats.
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
  
pkitool --initca                      ## equivalent to the 'build-ca' script --> crée une cler privée ca.key.
pkitool --server my-server            ## equivalent to the 'build-key-server' script
pkitool          my-client            ## equivalent to the 'build-key' script (*not build-key-pkcs12)
openssl dhparam -out dh2048.pem 2048  ## equivalent to the 'build-dh' script

Mais je n'ai pas encore tester le serveur.
logread -f
If everything went fine, the last log line from OpenVPN should contain Initialization Sequence Completed. There are some warnings and errors ... ignore them.

On cherche s'il y a une interface tun avec ifconfig -a
On regarde la table de routage avec netsat -nr. On doit avoir et c'est essentiel une route vers le serveur VPN (xxx.xxx.XXX.XXX), une route 
On fait un traceroute pour voir si on va vers le server vpn.

# pkitool --initca 
je vois que j'ai des nouveaux fichiers dans /etc/easy-rsa/keys  
- ca.crt  
- ca.key  
-index.txt qui est vide
- serial avec 01 dedans.


# pkitool --server my-openvpn_server  
nouveaux fichiers dans /etc/easy-rsa/keys  
  01.pem   
                index.txt.attr  
         my_openvpn_server.key  
ca.crt                 index.txt.old          serial
ca.key                 my_openvpn_server.crt  serial.old
index.txt              my_openvpn_server.csr


# pkitool my-openvpn-client
j'ai des nouveaux fichiers my-openvpn-client;*  

Je vois que cela utilise des parametres qui sont dans `/etc/easy-rsa/vars`, que l'on doit pouvoir customiser.

# openssl  dhparam -out dh2048.pm 2048
C'est long ...  


# Copie  des certificats
cp /etc/easy-rsa/keys/ca.crt /etc/easy-rsa/keys/my-openvpn-server.* /etc/easy-rsa/keys/dh2048.pem /etc/openvpn

# distribution des certificats sur le client openvpn
On fait cela comme on veut.

depuis le ssh du router TPLINk: scp /etc/easy-rsa/keys/ca.crt /etc/easy-rsa/keys/my-openvpn-client.* lof@ipduclient:/home/lof/TPLINK.  
Mais en fait les mettre dans le /etc/openvpn du client.

# Configuration du reseau sur le openwrt router (je ne sais pas si j'ai fait comme cela)

- 1 Create the VPN interface (named vpn0):
```
uci set network.vpn0=interface
uci set network.vpn0.ifname=tun0
uci set network.vpn0.proto=none
uci set network.vpn0.auto=1
```

- 2 Allow incoming client connections by opening the server port (default 1194) in our firewall:
```
uci set firewall.Allow_OpenVPN_Inbound=rule
uci set firewall.Allow_OpenVPN_Inbound.target=ACCEPT
uci set firewall.Allow_OpenVPN_Inbound.src=*
uci set firewall.Allow_OpenVPN_Inbound.proto=udp
uci set firewall.Allow_OpenVPN_Inbound.dest_port=1194
```


- 3 Create firewall zone (named vpn) for the new vpn0 network. By default, it will allow both incoming and outgoing connections being created within the VPN tunnel. Edit the defaults as required. This does not (yet) allow clients to access the LAN or WAN networks, but allows clients to communicate with services on the router and may allow connections between VPN clients if your OpenVPN server configuration allows:
```
uci set firewall.vpn=zone
uci set firewall.vpn.name=vpn
uci set firewall.vpn.network=vpn0
uci set firewall.vpn.input=ACCEPT
uci set firewall.vpn.forward=REJECT
uci set firewall.vpn.output=ACCEPT
uci set firewall.vpn.masq=1
```

- 4 (Optional) If you plan to allow clients to connect to computers within your LAN, you'll need to allow traffic to be forwarded between the vpn firewall zone and the lan firewall zone:
```
uci set firewall.vpn_forwarding_lan_in=forwarding
uci set firewall.vpn_forwarding_lan_in.src=vpn
uci set firewall.vpn_forwarding_lan_in.dest=lan
```

- 5 And you'll probably want to allow your LAN computers to be able to initiate connections with the clients, too.
```
uci set firewall.vpn_forwarding_lan_out=forwarding
uci set firewall.vpn_forwarding_lan_out.src=lan
uci set firewall.vpn_forwarding_lan_out.dest=vpn
```

- 6 (Optional) Similarly, if you plan to allow clients to connect the internet (WAN) through the tunnel, you must allow traffic to be forwarded between the vpn firewall zone and the wan firewall zone:
```
uci set firewall.vpn_forwarding_wan=forwarding
uci set firewall.vpn_forwarding_wan.src=vpn
uci set firewall.vpn_forwarding_wan.dest=wan
```

- 7 Commit the changes:
```
uci commit network
/etc/init.d/network reload
uci commit firewall
/etc/init.d/firewall reload
```




Quand est ce qu'on donne l'adresse du seveur vpn? dans le fichier de configuration du client /etc/config/openvpn : option remote SERVER_IP_ADRESS 1194
option remote 'pw.openvpn.ipredator.se 1194'


 YOu will need to open port 1194 (TCP) to be forwarding through the firewall to your OpenVPN server.


You can easily find out your OpenVPN server IP address. The syntax is as follows to get tun0 ip address on Unix or Linux:
ifconfig tun0

OR use Linux specific command:
ip a show tun0

dig TXT +short o-o.myaddr.l.google.com @ns1.google.com

## clean-all 
me dit que si je fais ./clean-all alors j'efface les directory /etc/easy-rsa/keys  
mais ./clean-all me dit que la commande n'existe.

mais je me retrouve avec un repertoire keys vide. 

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
On va configurer maintenant cette interface:
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
C'est mal expliqué sur le site    
- GUEST_ZONE IF_GUEST Edit  
- General setting je ne change rien  
- Inter-Zone Forwarding : 
	- Allow forward to destination zones : lan
j'ai accept dans zones / input sur le site c'est reject.  
Je le passe à reject.  

### ajouter des traffic rules :
Traffic rules / new forward rule  
une regle pour que les requetes dhcp aboutissent:
	- Name : allow_GUEST_DHCP
	- Add and Edit
		- restrict to address family : IPv4 and Ipv6
		- protocol udp
		- Match icmp type : any
		- Source zone : GUEST_ZONE
		- source mac address : any
		- source address : any
		- source port : any
		- destination zone : device (j'ai pas compris pourquoi. Je crois que le device c'est le router)
		- destination address : any
		- destination port : 67-68
		- extra argument : empty
et cela donne une regle :
```
Any udp
From any host in GUEST_ZONE
To any router IP at ports 67-68 on this device

Accept input
```

une regle pour le DNS: 
	- Name : allow_GUEST_DNS
	- Add and Edit
		- restrict to address family : IPv4 and Ipv6
		- protocol : TCP + UDP
		- Match icmp type : any
		- Source zone : GUEST_ZONE
		- source mac address : any
		- source address : any
		- source port : any
		- destination zone : device (j'ai pas compris pourquoi. Je crois que le device c'est le router)
		- destination address : any
		- destination port : 53
		- extra argument : empty
et cela donne une regle :
```
Any traffic
From any host in GUEST_ZONE
To any router IP at port 53 on this device

Accept input
```

J'ai fait un reset du routeur.  
et pas moyen d'obtenir une ip avec RADIO_GUEST. En fait des elements de config dans network/ interface / IF_GUEST avait fait pshittt.

OK j'ai mon ip en 192.168.2.xxx. Mais j'ai pas internet.

Si j'ai bien compris les paquets sont forwardés de 192.168.2.x vers 192.168.1.x par le routeur AP. Mais je n'ai jamais pu le vérifier. Maintenant il faut configurer le routeur principal. Il faut lui faire connaitre ce nouveau subnet (192.168.2.x). 

Sur le routeur principal si j'essaie de pinger un device connecté au reseau 192.168.4.x je n'y arrive pas 
Sur le routeur principal :  
Network / onglet static route / this section contains no value yet  
Add  
- interface: lan
- target 192.168.4.0 sous réseau de la nouvelle interface wifi
- netmask : 255.255.255.0
- ipV4 gateway qui est l'adresse lan de l'AP : 192.168.1.254
- metric inchange
- mtu inchangé

Et la bingo j'arrive à pinguer depuis le routeur principal l'adresse 192.168.4.x d'un device connecté à l'access point.  Je n'y arrive plus.
Par contre je ne peux pas surfer  

```
# Insert (-I) entries into your public zone's forwarding rule
## Reject all traffic
iptables -I forwarding_ZONE_GUEST_rule -j REJECT
## Reject all TCP traffic with reset flag to avoid unnessecary timeouts
iptables -I forwarding_ZONE_GUEST_rule -p tcp -j REJECT --reject-with tcp-reset

# Allow certain traffic
iptables -I forwarding_ZONE_GUEST_rule -p tcp -m tcp --dport 80 -j ACCEPT -m comment --comment "Allow traffic on port 80"
iptables -I forwarding_ZONE_GUEST_rule -p tcp -m tcp --dport 443 -j ACCEPT -m comment --comment "Allow traffic on port 443"
# Reject traffic with destination LAN
iptables -I forwarding_Z_Wifi_Mitg_rule -d 192.168.2.0/24 -j REJECT
# Add a closing NewLine, otherwise the last command may be not interpreted correctly (e.g. because you did not use vi as editor).
```
depuis le routeur principal je peux pinguer : 192.168.4.1. Je ne peux pas pinguer un device connecté en 192.168.4.x.  
Depuis le device connecté en 192.168.4.x je peux pinguer 8.8.8.8, 139.130.4.5. Je ne peux pas pinguer www.google.com
Depuis l'AP je peux pinguer 8.8.8.8 mais pas www.google.com  
donc j'ajoute  dans network / interface / lan / use custom DNS server : 8.8.8.8
et c'est bon je peux surfer depuis le device.
et aussi network / interface / lan / use custom DNS server : 192.168.1.1 (adresse lan du routeur principal)
