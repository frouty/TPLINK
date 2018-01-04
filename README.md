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

https://wiki.openwrt.org/doc/howto/vpn.openvpn

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

choisir un nom my-openvpn_server court et ayant du sens car on va le réutiliser dans d'autres fichiers de config et s'affiche dans le webGUI.

nouveaux fichiers dans /etc/easy-rsa/keys  
  01.pem   
                index.txt.attr  
         my_openvpn_server.key  
ca.crt                 index.txt.old          serial
ca.key                 my_openvpn_server.crt  serial.old
index.txt              my_openvpn_server.csr


# pkitool my-openvpn-client
Choisir un nom court et ayant du sens...


j'ai des nouveaux fichiers my-openvpn-client;*  

Je vois que cela utilise des parametres qui sont dans `/etc/easy-rsa/vars`, que l'on doit pouvoir customiser.

# openssl  dhparam -out dh2048.pem 2048
C'est long ...  
Trop long ... on peut diner.
apres tout ce temps je ne vois pas le fichier dh2048.pem dans /etc/easy-rsa/keys  


# Copie  des certificats
cp /etc/easy-rsa/keys/ca.crt /etc/easy-rsa/keys/my-openvpn-server.* /etc/easy-rsa/keys/dh2048.pem /etc/openvpn

On copie le ca.crt et tous ce qui concerne le serveur et le dh2048.pem  

Si on change le chemin il faudra adapter au fichier de config de openvpn.


# distribution des certificats sur le client openvpn
On fait cela comme on veut. clef usb, mail....etc...   

depuis le ssh du router TPLINk: scp /etc/easy-rsa/keys/ca.crt /etc/easy-rsa/keys/my-openvpn-client.* lfs@ipduclient:/home/lof/TPLINK.  
Mais en fait les mettre dans le /etc/openvpn du client.


On diffuse le ca.crt et tout ce qui concerne le client. On a un .csr .crt .key  
 Il sont dans mon bitbucket maintenant.

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

# configuration de OpenVPN 
en tun server

```
echo > /etc/config/openvpn # clear the openvpn uci config
uci set openvpn.myvpn=openvpn
uci set openvpn.myvpn.enabled=1
uci set openvpn.myvpn.verb=3
uci set openvpn.myvpn.port=1194
uci set openvpn.myvpn.proto=udp
uci set openvpn.myvpn.dev=tun
uci set openvpn.myvpn.server='10.8.0.0 255.255.255.0'
uci set openvpn.myvpn.keepalive='10 120'
uci set openvpn.myvpn.ca=/etc/openvpn/ca.crt 
uci set openvpn.myvpn.cert=/etc/openvpn/my-server.crt  # hackme
uci set openvpn.myvpn.key=/etc/openvpn/my-server.key # hackme
uci set openvpn.myvpn.dh=/etc/openvpn/dh2048.pem
uci commit openvpn
```

# demarrage du server
```
/etc/init.d/openvpn enable
/etc/init.d/openvpn start
```

Parfois rien ne se passe. J'ai du faire un reboot pour voir dans services  / openvpn une instance de nom `myvpn` qui correspond au nom dans le fichier /etc/config/openvpn.

cat /tmp/openvpn.log:
```
Tue Jan  2 23:02:33 2018 OpenVPN 2.3.6 mips-openwrt-linux-gnu [SSL (OpenSSL)] [LZO] [EPOLL] [MH] [IPv6] built on Jul 25 2015
Tue Jan  2 23:02:33 2018 library versions: OpenSSL 1.0.2g  1 Mar 2016, LZO 2.08
Tue Jan  2 23:02:33 2018 Diffie-Hellman initialized with 2048 bit key
Tue Jan  2 23:02:33 2018 Socket Buffers: R=[163840->131072] S=[163840->131072]
Tue Jan  2 23:02:33 2018 TUN/TAP device tun0 opened
Tue Jan  2 23:02:33 2018 TUN/TAP TX queue length set to 100
Tue Jan  2 23:02:33 2018 do_ifconfig, tt->ipv6=0, tt->did_ifconfig_ipv6_setup=0
Tue Jan  2 23:02:33 2018 /sbin/ifconfig tun0 10.8.0.1 pointopoint 10.8.0.2 mtu 1500
Tue Jan  2 23:02:33 2018 /sbin/route add -net 10.8.0.0 netmask 255.255.255.0 gw 10.8.0.2
Tue Jan  2 23:02:33 2018 UDPv4 link local (bound): [undef]
Tue Jan  2 23:02:33 2018 UDPv4 link remote: [undef]
Tue Jan  2 23:02:33 2018 MULTI: multi_init called, r=256 v=256
Tue Jan  2 23:02:33 2018 IFCONFIG POOL: base=10.8.0.4 size=62, ipv6=0
Tue Jan  2 23:02:33 2018 Initialization Sequence Completed
```

et je vois apparaitre une nouvelle interface en faisant ifconfig sur le router:

tun0 avec comme ip 10.8.0.1 qui est configurée dans le /etc/config/openvpn.



`/etc/init.d/openvpn enable or disable` se retrouve dans le webgui system / startup 



Mais plus de connection extranet. Je reboot le reouter pas de connection internet. Le serveur openerp a l'air d'etre ok. 
Je le stop et disable dans startup. Je le reboot. et a nouveau pas de connection internet. donc je reboot le modem adsl.
J'essaie de redémarrer le serveur openvpn pour voir si c'est lui qui fait tomber la connection internet. 
- 1 enable dans startup 
- 2 start dans service.
et c'est bon j'ai la connection extranet.

IP 103.17.47.187




Quand est ce qu'on donne l'adresse du serveur vpn? dans le fichier de configuration du client /etc/config/openvpn : option remote SERVER_IP_ADRESS 1194
option remote `pw.openvpn.ipredator.se 1194`

You will need to open port 1194 (TCP) to be forwarding through the firewall to your OpenVPN server.  

You can easily find out your OpenVPN server IP address. The syntax is as follows to get tun0 ip address on Unix or Linux:
`ifconfig tun0`

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
```
client
dev tap
proto udp
# remote yourddns.dyndns.org 5712
remote 192.168.1.160 5712
```

# Le client sur une debian:
cd /etc/openvpn  
touch client.conf  
y mettre
```
dev tun
proto udp

verb 3

ca /etc/openvpn/ca.crt
cert /etc/openvpn/my-openvpn-client.crt
key /etc/openvpn/my-openvpn-client.key

client 
remote-cert-tls server
remote kuendu.ddns.net 1194
```

On démarre le client avec la commande `service openvpn stop service openvpn start`  
Pas besoin de mettre le chemin du fichier de config le script init va chercher automatiquement les .conf dans /etc/openvp.  

Ou est le fichier de log?

Dans le script /etc/init.d/openvpn on des lignes comme:
log_daemon_msg  
log_progress_msg  
log_end_msg  
log_action_msg  
log_success_msg  
log_failure_msg  
log_warning_msg  

Il semblerait que les msg s'affiche en console.

On trouve le log dans /etc/openvpn/openvpn.log.

- 1 `service openvpn stop service openvpn start`  
rien en console  
ps aux | grep openvpn rien
rien ne bouge dans etc/openvpn/openvpn.log.
dans /var/log rien juste dans syslog started openvpn service.
- 2 `openvpn --config /etc/openvpn/openvpn.log`  en user ca ne marche pas 
rien dans le log. 
j'ai un process avec ps aux | grep openvpn
- 3  `openvpn --config /etc/openvpn/openvpn.log`  (en root)
-4  si je rm le openvpn.log il n'est pas recrée lors de service openvpn start et aussi /etC/init.d/openvpn 



# installer ohmyzsh en root
sudo su
sh -c "$(wget https://raw.github.com/robbyrussell/oh-my-zsh/master/tools/install.sh -O -)"



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

Je n'ai plus cela ni avec openvpn --config ... ou service openvpn restart

oui mais je n'ai pas de tun dans ifconfig.


Si votre openvpn est votre routeur ce sera l'adresse du routeur 192.168.1.1.

# configuration du client
cd /etc/openvpn
touch client.conf

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


# /etc/config/openvpn serveur

```
config openvpn 'myvpn'
        option enable '1'
        option verb '3'
        option port '1194'
        option proto 'udp'
        option dev 'tun'
        option server '10.8.0.0 255.255.255.0'
        option keepalive '10 120'
        option ca '/etc/openvpn/ca.crt'
        option dh '/etc/openvpn/dh2048.pem'
        option cert '/etc/openvpn/my.server.crt'
        option key '/etc/openvpn/my.server.key'
        option log '/tmp/openvpn.log'
        option ifconfig_pool_persit '/tmp/ipp.txt'
        option status '/tmp/openvpn-status.log'
        list push 'route 10.66.0.0 255.255.255.0'
        list push 'dhcp-option DNS 10.66.0.1'
```
mais je ne sais pas si il marche.






# Dans le web gui on peut configurer openvpn
avec des choix : 
- client configuration for a routed multi-client vpn0
- ....

et cela utilise le fichier /etc/config/openvpn_recipes.
 

# Le client
Dans /etc/openvpn  
touch client.conf  
## pour lancer le client 
### `service openvpn start`
n'a pas besoin de savoir ou le .conf. Utilise tous le .conf de /etc/openvpn  


## le fichier de log
dans /etc/openvpn
nom : openvpn.log  
Si je le rm il n'est pas recrée par `service openvpn start` ni par `openvpn --config /etc/openvpn/client.conf`
Un reboot et le fichier est recrée.  il est récrée avec des lignes alors que je n'ai pas lancer openvpn !!!!
ps aux | grep openvpn J'ai bien openvpn qui tourne. pourquoi openvpn demarre au reboot? TODO

## debugging
Je vois que [AF_INET]103.17.45.190:1194 qui est bien l'adresse IP du cabinet. C'est pas sur.

TLS Error: TLS key negotiation failed to occur within 60 seconds  
Grossiere erreur dans le client.conf ce n'est pas kuendu.ddns.net qu'il faut mettre mais goeen.ddns.net.  
`service openvpn restart`  
C'est nettement mieux :

```
Thu Jan  4 06:26:02 2018 WARNING: file '/etc/openvpn/my-openvpn-client.key' is group or others accessible
Thu Jan  4 06:26:02 2018 Socket Buffers: R=[212992->131072] S=[212992->131072]
Thu Jan  4 06:26:02 2018 UDPv4 link local (bound): [undef]
Thu Jan  4 06:26:02 2018 UDPv4 link remote: [AF_INET]103.17.47.187:1194
Thu Jan  4 06:26:02 2018 TLS: Initial packet from [AF_INET]103.17.47.187:1194, sid=04ed319e 76602813
Thu Jan  4 06:26:02 2018 VERIFY OK: depth=1, C=US, ST=CA, L=SanFrancisco, O=Fort-Funston, OU=MyOrganizationalUnit, CN=Fort-Funston CA, name=EasyRSA, emailAddress=me@myhost.mydomain
Thu Jan  4 06:26:02 2018 Validating certificate key usage
Thu Jan  4 06:26:02 2018 ++ Certificate has key usage  00a0, expects 00a0
Thu Jan  4 06:26:02 2018 VERIFY KU OK
Thu Jan  4 06:26:02 2018 Validating certificate extended key usage
Thu Jan  4 06:26:02 2018 ++ Certificate has EKU (str) TLS Web Server Authentication, expects TLS Web Server Authentication
Thu Jan  4 06:26:02 2018 VERIFY EKU OK
Thu Jan  4 06:26:02 2018 VERIFY OK: depth=0, C=US, ST=CA, L=SanFrancisco, O=Fort-Funston, OU=MyOrganizationalUnit, CN=my_openvpn_server, name=EasyRSA, emailAddress=me@myhost.mydomain
Thu Jan  4 06:26:03 2018 Data Channel Encrypt: Cipher 'BF-CBC' initialized with 128 bit key
Thu Jan  4 06:26:03 2018 Data Channel Encrypt: Using 160 bit message hash 'SHA1' for HMAC authentication
Thu Jan  4 06:26:03 2018 Data Channel Decrypt: Cipher 'BF-CBC' initialized with 128 bit key
Thu Jan  4 06:26:03 2018 Data Channel Decrypt: Using 160 bit message hash 'SHA1' for HMAC authentication
Thu Jan  4 06:26:03 2018 Control Channel: TLSv1, cipher TLSv1/SSLv3 DHE-RSA-AES256-SHA, 2048 bit RSA
Thu Jan  4 06:26:03 2018 [my_openvpn_server] Peer Connection Initiated with [AF_INET]103.17.47.187:1194
Thu Jan  4 06:26:06 2018 SENT CONTROL [my_openvpn_server]: 'PUSH_REQUEST' (status=1)
Thu Jan  4 06:26:06 2018 PUSH: Received control message: 'PUSH_REPLY,route 10.66.0.0 255.255.255.0,dhcp-option DNS 10.66.0.1,route 10.8.0.1,topology net30,ping 10,ping-restart 120,ifconfig 10.8.0.6 10.8.0.5'
Thu Jan  4 06:26:06 2018 OPTIONS IMPORT: timers and/or timeouts modified
Thu Jan  4 06:26:06 2018 OPTIONS IMPORT: --ifconfig/up options modified
Thu Jan  4 06:26:06 2018 OPTIONS IMPORT: route options modified
Thu Jan  4 06:26:06 2018 OPTIONS IMPORT: --ip-win32 and/or --dhcp-option options modified
Thu Jan  4 06:26:06 2018 ROUTE_GATEWAY 192.168.1.1/255.255.255.0 IFACE=eth0 HWADDR=00:25:22:16:3b:e5
Thu Jan  4 06:26:06 2018 TUN/TAP device tun0 opened
Thu Jan  4 06:26:06 2018 TUN/TAP TX queue length set to 100
Thu Jan  4 06:26:06 2018 do_ifconfig, tt->ipv6=0, tt->did_ifconfig_ipv6_setup=0
Thu Jan  4 06:26:06 2018 /sbin/ip link set dev tun0 up mtu 1500
Thu Jan  4 06:26:06 2018 /sbin/ip addr add dev tun0 local 10.8.0.6 peer 10.8.0.5
Thu Jan  4 06:26:06 2018 /sbin/ip route add 10.66.0.0/24 via 10.8.0.5
Thu Jan  4 06:26:06 2018 /sbin/ip route add 10.8.0.1/32 via 10.8.0.5
Thu Jan  4 06:26:06 2018 Initialization Sequence Completed
```

# comment on test depuis le client?
## ping IP du server 
OK
## On vérifie la création de l'interface tun
- 1 `cat /etc/openvpn/openvpn.log | grep tun`
- 2 `ifconfig `  
inet addr:10.8.0.6  P-t-P:10.8.0.5  Mask:255.255.255.255  


## traceroute
- `traceroute 10.8.0.1`  
pourquoi 1 ? TODO  
retourne :
traceroute to 10.8.0.1 (10.8.0.1), 30 hops max, 60 byte packets
 1  10.8.0.1 (10.8.0.1)  49.363 ms  51.015 ms  53.385 ms
- `traceroute 8.8.8.8`  
traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets  
 1  OpenWrt.lan (192.168.1.1)  12.743 ms  15.067 ms  15.075 ms  
 2  202.22.235.253 (202.22.235.253)  29.926 ms  29.927 ms  31.662 ms  
 3  202.22.224.41 (202.22.224.41)  36.408 ms  36.419 ms  38.701 ms  
 4  202.87.128.65 (202.87.128.65)  38.732 ms  40.234 ms  42.989 ms  
 5  202.87.128.77 (202.87.128.77)  66.858 ms  66.867 ms  69.123 ms  
 6  202.87.128.78 (202.87.128.78)  69.085 ms  41.048 ms  54.076 ms  
 7  108.170.247.33 (108.170.247.33)  55.660 ms 108.170.247.49 (108.170.247.49)  55.667 ms 108.170.247.33 (108.170.247.33)  55.652 ms  
 8  209.85.251.53 (209.85.251.53)  57.124 ms 216.239.41.191 (216.239.41.191)  53.426 ms 216.239.41.83 (216.239.41.83)  53.411 ms  
 9  google-public-dns-a.google.com (8.8.8.8)  59.310 ms  59.315 ms  59.299 ms  
 On voit que l'on passe par le router du client et pas par l'openvpn server.

 - `nmap -sP 10.8.0.0/24` retourne que seul l'hote 10.8.0.6 est up. 
 
 # j'ai ma conncetion avec le server. Mais ensuite.
 Once the VPN is operational in a point-to-point capacity between client and server, it may be desirable to expand the scope of the VPN so that clients can reach multiple machines on the server network, rather than only the server machine itself.  
 For the purpose of this example, we will assume that the server-side LAN uses a subnet of 10.66.0.0/24 and the VPN IP address pool uses 10.8.0.0/24 as cited in the server directive in the OpenVPN server configuration file. 
  First, you must advertise the 10.66.0.0/24 subnet to VPN clients as being accessible through the VPN. This can easily be done with the following server-side config file directive:
`push "route 10.66.0.0 255.255.255.0"`  
Ce qui pour openwrt se traduit en : `uci add_list openvpn.myvpn.push='route 10.66.0.0 255.255.255.0'`  
Cela devrait pouvoir nous permettre de pinguer les ip du reseau du cabinet depuis le client.

# ping 10.66.0.200 bingo
# j'éteins et je ralume le pc et là pas moyen de pinguer sur l'autre réseau local.   
openvpn tourne  
on fait un restart. `service openvpn restart`. Et là j'ai a nouveau la bonne séquence dans le openvpn.log_action_msg
```
Thu Jan  4 21:08:27 2018 do_ifconfig, tt->ipv6=0, tt->did_ifconfig_ipv6_setup=0
Thu Jan  4 21:08:27 2018 /sbin/ip link set dev tun0 up mtu 1500
Thu Jan  4 21:08:27 2018 /sbin/ip addr add dev tun0 local 10.8.0.6 peer 10.8.0.5
Thu Jan  4 21:08:27 2018 /sbin/ip route add 10.66.0.0/24 via 10.8.0.5
Thu Jan  4 21:08:27 2018 /sbin/ip route add 10.8.0.1/32 via 10.8.0.5
Thu Jan  4 21:08:27 2018 Initialization Sequence Completed
```

# Config du tel
J'installe le tel sur son switch à la maison  
obtient son adress ip depuis le dhcp du main router de la maison. je pense que c'est normal je n'ai pas configuré le client de l'ipphone. 
je le vois apparaitre dans les dhcp lease du routeur.   
http ipduphone 
Network / advanced  / vpn  
active Yes   
upload VPN Config / browse / je prends le /etc/openvpn/client.conf / import 
me dit qu'il veut un ovpn ou un tar. 
J'ai essayé de renomer le client,conf en client.ovpn ne marche pas.

Je ne trouve pas comment configurer l'ip phone avec un server openvpn sur le main router. Donc j'utile la méthode de freepbx avec un server VPN sur le freepx
# Comment configurer un VPN server sur le Freepbx. 


# adresse ip de l'ip phone. elle est obtenue par le dhcp du réseau sur lequel est branché le téléphone. 
elle sert à aller sur le webgui.
l'account est not registered. je ne comprends pas pourquoi.
Dans le web gui du telephone : Account / Basic / Connect Mode je coche VPN. Cela ne change rien  
Dans le EPM / sangome Template / provisionning / HTTP au lieu de TFTP  
Je vois que  dans l'ucp je download un fichier .zip vide.
Et dans user management / onglet VPN / autocreate a link yes au lieu de inherit
Dans l'ucp je vois apparaitre un nouveau lien hellocab-5 en plus de Hello Cab VPN client.
Mais toujours 0 bite. 
Settings / EPM / VPN client je passe de Hello Cab vPN client à hellocab-5 / Save and Rebuild  
UCP non toujours client zip O bytes
System admin / vpn / onglet settings / Enabled no to yes
Je ne peux plus pinguer 10.66.0.2. mais les autres ip du reseau : OK. 
