# Atelier Proxmox

On prépare notre serveur pour les saisons à venir ! Au programme, du Proxmox, du NAT, de la config' de pfSense !

O'clock vous met à disposition un **serveur dédié** dans le cadre de votre formation. Proxmox y a été pré-installé, **vous avez dû recevoir un mail** sur votre adresse O'clock contenant l'**adresse IP du serveur** et le **mot de passe** de l'utilisateur `root`.

⚠️ Ce serveur vous est mis à disposition pour des fins pédagogiques. Vous en êtes administrateur, vous pouvez faire les tests et labos de votre choix dessus mais **soyez responsables** !

⚠️ Vous partagez ce serveur avec un collègue de promo, ne supprimez pas les VMs ou ressources de ce collègue 😉

⚠️ Toutes les étapes ci-dessous sont à réaliser à deux, avec votre collègue. Vous devez être en vocal sur Discord pour réaliser cet atelier, et celui qui manipule peut partager son écran pour que l'autre puisse suivre et aider.

## Étape 1 - Accès à l'interface

Rendez-vous sur [https://mail.oclock.school/](https://mail.oclock.school/) pour consulter vos mails O'clock.

Vous avez dû en recevoir un avec l'objet `O'clock - Ton serveur est arrivé !`. Si ce n'est pas le cas, prévenez votre formateur.

Cliquez sur le lien dans ce mail et connectez-vous avec le nom d'utilisateur `user1` ou `user2` (en fonction de ce qui est indiqué dans le mail) et le mot de passe fourni.

💡 Vérifiez que le `royaune/realm` sélectionné est bien `Proxmox VE authentication server`. Vous pouvez passer l'interface en français.

## Étape 2 - Proxmox : configuration réseau

Notre serveur chez OVH n’a qu’**une seule adresse IP publique**, déjà configurée après l’installation.

➡️ Vous pouvez visualiser les interfaces réseau du serveur en cliquant sur son nom à gauche, puis en allant dans la section `Système > Réseau` :

![](./images/proxmox_reseau.png)

⚠️ Vous n'aurez pas nécessairement les mêmes interfaces que ce qu'on peut voir dans la capture ci-dessus, c'est **normal**.

💡 Pour info, notre serveur est dans un sous-réseau en `/24`. La passerelle est la machine en `.254` dans ce sous-réseau.

**Les autres adresses de ce sous-réseau ne sont pas utilisables**, et peuvent potentiellement appartenir à d’autres clients d’OVH. **Ne faites pas n’importe quoi !** ⚠️

Certains de vos serveurs sont d’ailleurs potentiellement dans le même sous-réseau !

> Vu que nous n’avons qu’une seule adresse IP, comment on va faire pour virtualiser plusieurs machines ? Ces machines devront bien avoir accès à Internet !

Pas le choix, on va devoir mettre en place ... du **NAT** ! Plus exactement, on va mettre en place deux NATs consécutifs : un premier directement au niveau de Promox, un deuxième dans une VM pfSense.

Voici le diagramme réseau représentant ce que vous allez devoir mettre en place aujourd'hui (ce sera votre bible, n'hésitez pas à venir y jeter un oeil pendant l'atelier) :

![](./images/network.svg)

### Étape 2.1 - Interfaces bridge

Sur GNU/Linux, on peut créer des interfaces réseaux appelées `bridges` (pont, en français). Ces interfaces virtuelles vont nous permettre d'inter-connecter des interfaces réseau physiques et également les interfaces réseau de nos machines virtuelles.

On peut voir ça comme une sorte de **switch virtuel**, un peu comme les interfaces `réseau interne` sur Virtual Box !

Par défaut, vous n'aurez qu'elle seule interface bridge : `vmbr0`, l'interface réseau physique `eno1` de notre serveur y est connectée. ⚠️ **Ne modifiez pas la configuration de cette interface !**

**Cette interface `vmbr0` sera l'interface de sortie de notre NAT.**

Pour connecter le WAN de la VM pfSense que vous allez devoir créer par la suite, il va nous falloir une nouvelle interface bridge.

Au boulot 💪

➡️ Dans la section `Système > Réseau` de votre serveur Proxmox (après avoir cliqué sur son nom à gauche), cliquez sur le bouton `Créer` puis `Linux Bridge` pour créer cette nouvelle interface.

➡️ Dans le champ `IPv4/CIDR`, saisissez l'adresse IP statique au format CIDR que nous voulons attribuer à notre serveur Proxmox sur cette interface : `192.168.42.1/24`. Vérifiez que la case `Démarrage automatique` est bien cochée, laissez le reste des champs vides et cliquez sur `OK`.

➡️ Refaites la même opération pour créer l'interface `vmbr2`, mais ne mettez pas d'adresse `IPv4/CIDR` ce coup-ci ⚠️

➡️ N'oubliez pas de cliquer sur le bouton `Appliquer la configuration` en haut !

### Étape 2.2 - NAT Proxmox

On peut maintenant mettre en place notre premier NAT, au niveau du serveur Proxmox ! Proxmox est basé sur Debian, donc on va utiliser le pare-feu `iptables` pour faire ça (on en reparlera plus tard dans la formation).

➡️ Cliquez sur le nom de votre serveur à gauche, puis ouvrez le `Shell`.

Pour vous connecter, utilisez le nom d'utilisateur `user` et le mot de passe `rocknroll` 😉

💡 Les commandes tapées dans le `Shell` sont lancées directement sur le serveur dédié sur lequel Proxmox est installé. Attention, vous avez les droits administrateur, vous pouvez facilement tout casser !

➡️ Lancez la commande `ip a` pour vérifier que nos interfaces bridges sont bien détectées et correctement configurées.

Vous devriez avoir la configuration suivante :

- **vmbr0 :** l'adresse IP publique de votre serveur
- **vmbr1 :** `192.168.42.1/24`
- **vmbr2 :** pas d'adresse IPv4

Exemple, sur le serveur formateur :

![](./images/ipa.png)

Plusieurs autres interfaces réseaux sont présentes, vous pouvez les ignorer.

La première étape consiste à éditer le fichier `/etc/sysctl.conf`, par exemple avec l'éditeur de texte `nano`.

➡️ Lancez la commande `sudo nano /etc/sysctl.conf`. Vous voilà dans un éditeur de texte en ligne de commande ! Qui dit ligne de commande, dit... pas de souris 😬 Vous devez déplacer le curseur avec les flèches directionnelles de votre clavier.

Dans ce fichier, vous devez **localiser puis décommenter la ligne `net.ipv4.ip_forward=1`**, en enlevant le caractère `#` au début de la ligne. Quittez et sauvegardez avec le raccourci `Ctrl+X` dans `nano` (il faudra ensuite appuyer sur `Y` pour sauvegarder puis `Entrée` pour valider le nom du fichier).

➡️ Lancez la commande `sudo sysctl -p /etc/sysctl.conf` pour appliquer la modification que nous venons d'effectuer (la commande devrait retourner `net.ipv4.ip_forward = 1`).

💡 Cette modification permet d'**activer l'IP forward**, sorte de "mode routeur" du noyau Linux.

➡️ Pour finir, lancez la commande `sudo iptables -t nat -A POSTROUTING -s 192.168.42.0/24 -o vmbr0 -j MASQUERADE` afin d'activer le NAT.

`-s 192.168.42.0/24` permet de n'autoriser que les paquets en provenance de ce sous-réseau à traverser le NAT.

`-o vmbr0` permet d'indiquer l'interface réseau de sortie.

Vous pouvez lancer la commande `sudo iptables -L -t nat` pour vérifier la configuration du pare-feu `iptables`, vous devriez obtenir ceci :

![](./images/iptables.png)

Ça y est, la configuration réseau sur Proxmox est terminée 🎉

> Déjà ?

Il faut encore qu'on teste, et si tout est OK, on devra revenir sauvegarder la configuration `iptables` dans une prochaine étape de cet atelier.

## Étape 3 - pfSense

💡 **pfSense est un système d'exploitation** permettant de transformer n'importe quel ordinateur en un **routeur** professionnel. Il n'est pas rare d'en rencontrer en entreprise ! Si vous voulez en savoir plus, jetez un oeil par [ici](https://fr.wikipedia.org/wiki/PfSense).

Sur cette étape, nous allons créer une VM pfSense qui servira de passerelle/de routeur pour toutes nos VMs pour le reste de la formation.

### Étape 3.1 - création VM

➡️ Cliquez sur le bouton `Créer une VM`, tout en haut dans l'interface.

Voici les réglages à utiliser :
- ID : 100
- nom : NAT-pfSense
- iso : pfSense 2.7.2 (déjà disponible sur votre serveur proxmox)
- type : Other (à la place de "Linux")
- disque : 20 Gio
- RAM : 2048 MiB
- CPU : 1
- Pont (bridge) : vmbr1 (⚠️ très important)
- Pare-feu : décochez la case (⚠️ très important)

**Vous pouvez laisser tous les autres réglages par défaut.**

➡️ **Avant de démarrer la VM**, ajoutez une seconde interface réseau depuis la section `Matériel`. Sélectionnez le Pont `vmbr2` cette fois-ci, et décochez également le pare-feu.

Voici ce que vous devriez obtenir dans la partie `Matériel` :

![](./images/pfsense_hw.png)

### Étape 3.2 - installation

➡️ Démarrez la VM, puis rendez-vous sur la section `Console` pour réaliser l'installation.

Rien de particulier pendant l'installation, il faut appuyer quasiment tout le temps sur `Entrée` pour valider le choix par défaut ! ⚠️ **Seule exception**, lors du choix du disque dur, il faudra appuyer sur la touche `Espace` pour cocher la case **avant** d'appuyer sur `Entrée`.

![](./images/pfsense_install.png)

💡 Une fois l'installation terminée, n'oubliez pas d'éjecter l'image ISO du lecteur de DVD virtuel via la section `Matériel` (`Éditer > N'utiliser aucun média`), afin d'éviter que l'installation se relance en boucle après le redémarrage.

### Étape 3.3 - configuration et test

Une fois la VM pfSense redémarrée, nous allons pouvoir configurer les adresses IPv4 sur ses interfaces et vérifier qu'elle a bien accès à Internet.

💡 Notre pfSense ne va pas récupérer automatiquement d'adresse sur son interface WAN (comme on peut le voir ci-dessous) : il n'y a pas de serveur DHCP actif sur notre pont `vmbr1` !

![](./images/pfsense_1st-screen.png)

➡️ Sur notre pfSense, sélectionnez le menu `8) Shell` avec votre clavier (attention, en Qwerty de base !), puis lancez la commande `kbdcontrol -l /usr/share/vt/keymaps/fr.kbd` pour passer le clavier en Azerty. Tapez `exit` pour retourner au menu.

➡️ Toujours sur la pfSense, sélectionnez `2) Set interface(s) IP address`, puis `1` pour configurer l'interface WAN. La pfSense va vous poser une série de questions (en anglais !) pour configurer cette interface, suivez les points ci-dessous pour vous aider à répondre à ces questions.

💡 Petit indice : on ne veut pas récupérer une adresse via DHCP, mais attribuer une addresse IP fixe sur le pont `vmbr1` (notre switch virtuel entre la pfSense et Proxmox). Regardez le [diagramme de l'étape 2](./README.md#étape-2---proxmox--configuration-réseau) pour déterminer l'adresse à utiliser sur la patte WAN de la pfSense !

⚠️ Attention, **cette interface WAN doit avoir une passerelle** : l'adresse IP de notre Proxmox sur le pont `vmbr1`. Reportez-vous à nouveau au diagramme pour renseigner la bonne adresse lors de la configuration de l'interface. Cette passerelle sera celle utilisée **par défaut** par la pfSense.

💡 Pas besoin d'adresse IPv6, pas besoin d'activer un serveur DHCP sur le WAN, ni de désactiver HTTPS et repasser en HTTP pour l'interface web de la pfSense.

L'interface LAN a déjà une configuration par défaut. On modifiera cette configuration depuis l'interface graphique, après avoir installé une VM Windows 10 !

➡️ Toujours sur la pfSense, sélectionnez `7) Ping host` puis saisissez `8.8.8.8` pour vérifier que la VM a bien accès à Internet. Si le ping ne passe pas, vérifiez que vous n'avez pas loupé une étape !

![](./images/pfsense_ping.png)

## Étape 4 - Windows 10

➡️ Créez une nouvelle VM Windows 10. Débrouillez-vous pour cette étape, il n'y a pas de piège / rien de très différent de ce qu'on a déjà fait sur Virtual Box.

💡 L'interface réseau doit être connectée sur le pont `vmbr2`, notre "switch virtuel" pour toutes nos VMs ! (décochez également le pare-feu dans la partie `Réseau`)

À la fin de l'installation, la machine Windows devrait obtenir une adresse IP grace au serveur DHCP de la pfSense installée précédemment. Si ce n'est pas le cas, vérifiez que vous n'avez pas loupé une étape.

## Étape 5 - config' pfSense

➡️ Depuis un navigateur web sur la VM Windows 10, rendez-vous sur l'interface de la pfSense à l'adresse [192.168.1.1](https://192.168.1.1).

💡 Votre navigateur vous avertira d'un risque de sécurité, c'est normal. Vous pouvez cliquer sur `Avancé` et pursuivre !

![](./images/pfsense_warn.png)

Connectez-vous avez le nom d'utilisateur `admin` et le mot de passe `pfsense`.

➡️ Suivez les étapes du `Wizard pfSense Setup`, pour effectuer la configuration initiale de notre pfSense.

Voici les réglages à utiliser :

- Première page :
  - Hostname : pfSense
  - Domain : le nom de votre Proxmox + `.lan` (par exemple : `ns500131.lan`)
  - Primary DNS Server : `8.8.8.8`
  - Secondary DNS Server : laisser vide
  - Override DNS : décocher
- Deuxième page :
  - Time server hostname : laisser le serveur renseigné par défaut
  - Timezone : changer pour `Europe/Paris`
- Troisième page :
  - rien à changer, sauf tout en bas ! Décocher la case `Block RFC1918 Private Networks` (⚠️ très important)
- Quatrième page :
  - rien à changer, on laisse l'IP configurée pour l'instant.
- Cinquième page :
  - admin password : choisissez `rocknroll`, ce sera plus pratique si on doit venir vous dépanner.

Cliquez sur `Reload` pour appliquer les changements.

Par défaut, pfSense utilise `192.168.1.1/24` comme adresse sur le LAN, nous allons modifier ça !

➡️ Rendez-vous dans `Interfaces > LAN` (depuis le menu tout en haut) puis attribuez l'adresse `10.0.0.1/16` à la pfSense sur le LAN (c'est la seule chose à modifier ici). N'oubliez pas de cliquer sur `Save` tout en bas puis d'appliquer les changements !

💡 Vous rencontrez des erreurs liées à la configuration IPv6 ? Désactivez le serveur DHCPv6(dans `Services > DHCPv6 Server`) ET le `Router Advertisement` (dans `Services > Router Advertisement`, passez `Router Mode` à `Disabled`) puis sélectionnez `None` comme configuration IPv6 de l'interface LAN et réessayez.

Dès cette configuration validée, nous allons **perdre l'accès à l'interface web de la pfSense** (normal, elle a changée d'adresse).

➡️ **Attribuez une adresse IP statique** à votre VM windows 10 sur le sous-réseau `10.0.0.0/16`, par exemple :

![](./images/win_static.png)

💡 On configure l'adresse IP de la pfSense sur le LAN (`10.0.0.1`) comme passerelle et comme serveur DNS.

➡️ Reconnectez-vous à la pfSense via sa nouvelle adresse sur le LAN (`10.0.0.1`), et rendez-vous dans `Services > DHCP Server` et ajustez les paramètres du serveur DHCP sur le LAN. On veut qu'il distribue des adresses sur l'étendue `10.0.0.50 - 10.0.0.250`.

Vérifiez que votre machine Windows 10 récupère bien une adresse IP en DHCP.

💡 Vous aurez probablement un problème de DNS (vous empêchant d'accéder à Internet depuis la VM Windows). Pour le résoudre, il faut se rendre dans `Services > DNS Resolver` sur la pfSense, puis relancer le service en utilisant le bouton 🔄 en haut à droite.

![](./images/dns_pfsense.png)

Une fois le service relancé, vous devriez avoir accès à Internet depuis la VM Windows 🎉

## Étape 6 - VPN

Pour pouvoir plus facilement bosser sur nos VMs par la suite, on va créer un VPN permettant de directement accéder à notre pfSense depuis le navigateur web de notre PC, et pouvoir prendre la main à distance sur nos VMs en utilisant le protocole RDP ou SSH.

### Étape 6.1 - serveur OpenVPN

Rendez-vous sur la page `VPN > OpenVPN`, puis dans l'onglet `Wizards`.

Laissez `Local User Access` sélectionnée, et cliquez sur `Next`.

Remplissez les différents champs en suivant les instructions ci-dessous :

- Première page :
  - Descriptive name : saisissez `vpn`
  - Common Name : saisissez `vpn`
  - laissez les autres champs à leur valeur par défaut, et cliquez sur `Add new CA`.
- Deuxième page :
  - cliquez sur `Add new Certificate`
  - Descriptive name : saisissez `vpn-cert-server`
  - Common Name : saisissez `vpn-cert-server`
  - laissez les autres champs à leur valeur par défaut, et cliquez sur `Create new Certificate`.
- Troisième page :
  - Description : saisissez `vpn-remote-access`
  - IPv4 Tunnel Network : saisissez `10.42.0.0/24`
  - IPv4 Local Network : saisissez `10.0.0.0/16`
  - laissez les autres champs à leur valeur par défaut, et cliquez sur `Next`.
- Quatrième page :
  - Firewall Rule : cochez la case
  - OpenVPN rule : cochez la case
  - puis cliquez sur `Next`.
- et cliquez enfin sur `Finish` !

Le VPN est presque prêt ! Il faut encore que l'on créé deux utilisateurs, un pour chaque membre de votre binôme !

### Étape 6.2 - utilisateurs & certificats

Rendez-vous sur `System > User Manager`, puis cliquez sur `Add`. Renseignez les différents champs en suivant les instructions ci-dessous :

- Username : `user1`
- Password/Confirm Password : ce que vous voulez, mais de préférence un mot de passe solide.
- Certificate : cochez la case (⚠️ si vous oubliez, ça ne fonctionnera pas)
- Section `Create Certificate for User` :
  - Descriptive name : `user1-vpn-cert`
- laissez les autres champs à leur valeur par défaut, et cliquez sur `Save`.

Reproduisez les mêmes étapes pour le deuxième utilisateur, `user2`.

Vous devriez obtenir ceci :

![](./images/pfsense_users.png)

Vous pouvez également vérifier dans `System > Certificates` puis dans l'onglet `Certificates`, vous devriez avoir 3 nouveaux certificats (un pour le serveur, et un pour chaque utilisateur, en plus du certificat `GUI default` qui est présent de base) :

![](./images/pfsense_certs.png)

### Étape 6.3 - export de la configuration client

Pour pouvoir exporter facilement les configurations pour nos utilisateurs du serveur VPN, on va devoir installer un paquet logiciel sur la pfSense.

Rendez-vous sur `System > Package Manager`, puis sur l'onglet `Available Packages`.

💡 Vous avez une erreur `Unable to retrieve package information` ? Si c'est le cas, rendez-vous sur la page d'accueil de la pfSense (en cliquant sur le logo en haut à gauche), puis cliquez sur le bouton 🔄 dans la section `System Information` / `Version`. Vous devriez voir un message vous indiquant qu'une nouvelle version est disponible, inutile de l'installer maintenant. Retournez dans le `Package Manager` et l'erreur devrait avoir disparue.

Cherchez "openvpn", et installez le premier paquet de la liste : `openvpn-client-export` (en cliquant sur le bouton `+ Install` correspondant) !

![](./images/package.png)

Rendez-vous ensuite dans `VPN > OpenVPN`, puis sur l'onglet `Client Export`.

Vous allez devoir changer le champ `Host Name Resolution` : remplacez `Interface IP Address` par `Other`, **puis saisissez dans le champ `Host Name` l'adresse IPv4 publique de votre serveur Proxmox.**

⚠️ Attention, ne vous trompez pas d'adresse ! On parle bien d'une adresse IP publique, donc pas une adresse de la RFC1918 ! L'adresse IP publique de votre serveur Proxmox apparaît dans la barre d'adresse de votre navigateur (sur votre machine, pas sur la VM Windows).

Cliquez sur le bouton `Save as default` plus bas.

Une fois que c'est fait, vous devrez télécharger le fichier `OpenVPN Connect (iOS/Android)` pour votre utilisateur (`user1` ou `user2`) un peu plus bas.

![](./images/openvpn_config.png)

Récupérez votre fichier (en fonction de votre utilisateur) sur votre PC en vous l'envoyant par [WeTransfer](https://wetransfer.com/) (par exemple) depuis la VM Windows 10.

### Étape 6.4 - redirection de port sur Proxmox

Retournez sur le shell Proxmox (en cliquant sur le nom de votre serveur en haut à gauche puis `Shell`), connectez-vous et lancez la commande suivante :

```
sudo iptables -t nat -A PREROUTING -i vmbr0 -p udp --dport 1194 -j DNAT --to-destination 192.168.42.254
```

💡 Cette commande permet de créer une redirection du port 1194 sur le protocole UDP (utilisé par OpenVPN) vers notre pfSense.

### Étape 6.5 - connexion au VPN

Installez sur votre ordinateur (pas sur une VM !) le logiciel [OpenVPN Connect](https://openvpn.net/client/).

Une fois sur OpenVPN Connect, il faudra cliquer sur le bouton `Upload File` en bas pour charger votre fichier de configuration VPN récupéré à l'étape précédente.

![](./images/ovpn_connect.png)

Une fois le fichier importé, vous devriez pouvoir vous connecter en cliquant sur le bouton `Connect`. Renseignez votre nom d'utilisateur (`user1` ou `user2`) et le mot de passe que vous avez choisi lors de la création de cet utilisateur sur la pfSense. Si tout est OK, vous devriez voir ceci :

![](./images/vpn_co.png)

Ouvrez votre navigateur web (sur votre machine, pas dans une VM !) et essayez d'accéder à [https://10.0.0.1/](https://10.0.0.1/). Vous devriez voir la page de connexion de votre pfSense, ça veut dire que le VPN fonctionne 🎉

## Étape 7 - Sauvegarde iptables

Si tout fonctionne bien (que votre VM Windows 10 a bien accès à Internet et que le VPN fonctionne), on va pouvoir sauvegarder notre configuration `iptables`.

En effet, notre config' `iptables` pour le NAT sur Proxmox ne sera pas conservée après un redémarrage. Comme pour les équipements Cisco, il va falloir qu’on « sauvegarde » cette config.

➡️ **Sur le Shell de votre serveur Proxmox**, exportez la config' dans le fichier `/etc/iptables-rules.save` avec la commande `sudo iptables-save | sudo tee /etc/iptables-rules.save`.

Et on veut que ce fichier soit « appliqué » au démarrage, pour cela on doit rajouter la ligne `post-up iptables-restore < /etc/iptables-rules.save` dans le fichier `/etc/network/interfaces`, sous notre interface `vmbr0` (après une tabulation).

Utilisez la commande `sudo nano /etc/network/interfaces`, et ajoutez la ligne indiquée ci-dessus comme visible dans cette capture d'écran :

![](./images/postup.png)

Vous êtes arrivé jusque-là ? Bravo, c'était vraiment pas facile du tout 💪

C'est déjà vraiment très très bien d'avoir fini l'atelier, mais si vous vous ennuyez... quelques bonus sont dispo ci-dessous 😉

## Bonus : Redirection de ports & Caddy

Démarrez un serveur web Caddy (vous pouvez vous inspirer de ce qu'on a fait sur vos machines en cours) sur la VM Windows 10.

💡 La procédure à suivre est à la fin des slides sur le NAT, disponibles sur le drive.

On va maintenant essayer de rendre ce serveur accessible sur Internet !

Pour cela, on doit mettre en place deux redirections de ports :

- une sur la pfSense
- une sur Proxmox

💡 Il nous faut une redirection de port par NAT à traverser.

➡️ À vous de chercher comment faire ces deux redirections 😱

## Méga-bonus - Un deuxième LAN

Essayez de créer un deuxième LAN connecté à une troisième interface réseau sur la pfSense ! Ce deuxième LAN permettra de cloisonner les VMs que vous créerez par la suite : l'un de vous deux bossera sur le LAN1, l'autre sur le LAN2 !

➡️ Il faudra refaire presque toutes les étapes : créer une nouvelle interface Bridge/Pont sur Proxmox (`vmbr3`), ajouter une interface réseau sur la pfSense, et configurer une adresse IPv4 statique ainsi que le serveur DHCP.

Utilisez le sous-réseau `172.16.0.1/16`.

Vous pourrez ensuite créer une deuxième VM Windows connectée à ce pont `vmbr3`.

💡 Pour qu'elle ait accès à Internet, il faudra regarder du coté des règles de pare-feu de la pfSense (`Firewall > Rules`). Comparez les règles coté LAN et coté LAN2 😉 (indice : il faudra créer une règle coté LAN2)