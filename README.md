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


```













