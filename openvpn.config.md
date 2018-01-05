# Config du server VPN sur le freepbx.

# System admin / VPN server tout en bas du menu déroulant utiliser l'ascenseur
openVPN client  
Enable yes  
description : hello cab VPN client  
use ddns no   
use server remote address no  
Client remote address : goeen.ddns.net  
assigned address no .

mais normalement quand on démarre le server VPN il y a deux onglets que je ne vois plus.

Applications / Extension / Add extension / Add new CHan_SIP extension  
user extension : un chiffre
display name
Link to a default user : Create new user.


Admin / user management / onglet VPN 
define additionnal VPN client : selectionner hello cab VPN client (crée sans system Admin / VPN server )  

