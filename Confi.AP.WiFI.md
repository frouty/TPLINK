# Config wifi AP TPLINK






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
