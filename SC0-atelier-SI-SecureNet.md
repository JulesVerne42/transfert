# Atelier - Infrastructure virtualisée d'entreprise

## Contexte professionnel

Vous venez d'être recruté(e) en tant qu'administrateur système chez **SecureNet**, une PME de 50 employés spécialisée dans le conseil en cybersécurité. L'entreprise possède actuellement une infrastructure physique vieillissante et souhaite migrer vers une infrastructure virtualisée moderne.

Votre manager vous confie la mission de concevoir et déployer une **infrastructure virtualisée complète** qui servira d'environnement de test et de développement pour l'équipe. Cette infrastructure doit être sécurisée, segmentée et facilement reproductible.

Vous disposez d'un serveur Proxmox dédié sur lequel vous allez construire cette infrastructure de A à Z.

## Objectifs de l'atelier

À l'issue de cet atelier, vous serez capable de :
- Concevoir une architecture réseau segmentée et sécurisée
- Créer et configurer des machines virtuelles sous Proxmox
- Mettre en place différents systèmes d'exploitation (Windows et Linux)
- Configurer la communication entre VMs selon des règles de sécurité
- Déployer des services essentiels (AD, serveur web, base de données)
- Documenter une infrastructure complète
- Réaliser des snapshots et sauvegardes

---

## Prérequis

### Matériel et accès

- ✅ Accès à votre serveur Proxmox dédié OVH
- ✅ pfSense déjà configuré (atelier précédent)
- ✅ ISOs disponibles sur Proxmox :
  - Windows Server 2022
  - Debian 12
  - Ubuntu Server 22.04 LTS
- ✅ Connexion Internet stable
- ✅ Client RDP (Bureau à distance Windows)

---

## Architecture cible

### Vue d'ensemble

Vous allez déployer l'infrastructure suivante :

```
Internet
   |
[pfSense] (déjà configuré)
   |
   ├─── VLAN 10 - DMZ (10.10.10.0/24)
   |      └─── Web-SRV (10.10.10.10) - Debian
   |
   ├─── VLAN 20 - LAN Interne (10.20.20.0/24)
   |      ├─── DC-SRV (10.20.20.10) - Windows Server (AD)
   |      └─── DB-SRV (10.20.20.20) - Ubuntu (MySQL)
   |
   └─── VLAN 30 - Administration (10.30.30.0/24)
          └─── ADMIN-WKS (10.30.30.10) - Windows 10/11
```

### Segmentation réseau

| VLAN | Nom | Réseau | Usage | Niveau sécurité |
|------|-----|--------|-------|-----------------|
| 10 | DMZ | 10.10.10.0/24 | Services publics (web) | Élevé |
| 20 | LAN | 10.20.20.0/24 | Services internes (AD, DB) | Moyen |
| 30 | ADMIN | 10.30.30.0/24 | Postes administration | Élevé |

---

## PARTIE 1 : Planification et préparation (30 min)

### Objectif
Comprendre l'architecture et préparer l'environnement Proxmox.

### Travail à réaliser

**1.1 - Analyse de l'architecture**

Prenez le temps de comprendre l'architecture cible :
- Identifiez les différents VLANs et leur rôle
- Notez les adresses IP de chaque machine
- Comprenez les flux réseau nécessaires

**1.2 - Documentation initiale**

Créez un document (fichier texte, Excel, OneNote, etc.) qui contiendra :
- Tableau récapitulatif des VMs à créer
- Plan d'adressage IP
- Informations de connexion (à remplir au fur et à mesure)
- Journal des actions effectuées

Modèle suggéré :
```
VM : WEB-SRV
OS : Debian 12
VLAN : 10 (DMZ)
IP : 10.10.10.10/24
Gateway : 10.10.10.1
RAM : 2 GB
Disk : 20 GB
CPU : 2 cores
Rôle : Serveur web Apache
Statut : [ ] Créée [ ] OS installé [ ] Configurée [ ] Testée
Identifiants : root / [mot_de_passe]
Notes : 
```

**1.3 - Vérification des ressources Proxmox**

Connectez-vous à votre Proxmox et vérifiez :
- Espace disque disponible
- RAM disponible
- ISOs présentes dans le stockage
- État de pfSense (doit être démarré)

**1.4 - Configuration des bridges réseau**

Vérifiez que les bridges pour vos VLANs sont créés dans Proxmox :
- vmbr10 pour VLAN 10 (DMZ)
- vmbr20 pour VLAN 20 (LAN)
- vmbr30 pour VLAN 30 (ADMIN)

Si absents, créez-les :
```
Datacenter → [Votre nœud] → System → Network → Create → Linux Bridge
```

---

## PARTIE 2 : Déploiement de la DMZ (1h30)

### Objectif
Créer et configurer le serveur web dans la zone DMZ.

### Travail à réaliser

**2.1 - Création de la VM WEB-SRV**

Créez une nouvelle VM avec ces paramètres :
- **Nom** : WEB-SRV
- **OS** : Debian 12 (ISO)
- **Disque** : 20 GB
- **CPU** : 2 cores
- **RAM** : 2048 MB
- **Réseau** : vmbr10 (VLAN 10 - DMZ)

Configuration détaillée :
- Démarrage automatique : OUI
- Options → QEMU Agent : activer après installation

**2.2 - Installation de Debian**

Démarrez la VM et installez Debian :
- Partitionnement : automatique (tout sur une partition)
- Nom d'hôte : `web-srv`
- Domaine : `dmz.securenet.local`
- Compte root : définir un mot de passe fort
- Créer un utilisateur : `admin`
- Logiciels à installer :
  - [x] SSH server
  - [x] Standard system utilities
  - [ ] Desktop environment (NON, serveur sans GUI)

**2.3 - Configuration réseau statique**

Après l'installation, configurez l'IP statique :

```bash
# Se connecter en root
su -

# Éditer la configuration réseau
nano /etc/network/interfaces
```

Contenu :
```
auto ens18
iface ens18 inet static
    address 10.10.10.10
    netmask 255.255.255.0
    gateway 10.10.10.1
    dns-nameservers 1.1.1.1 8.8.8.8
```

Redémarrer le réseau :
```bash
systemctl restart networking
```

Vérifier :
```bash
ip addr show
ping -c 3 10.10.10.1  # Ping pfSense
ping -c 3 1.1.1.1      # Ping Internet
```

**2.4 - Mise à jour et installation des outils de base**

```bash
# Mettre à jour le système
apt update && apt upgrade -y

# Installer outils utiles
apt install -y curl wget net-tools sudo vim htop

# Ajouter l'utilisateur admin au groupe sudo
usermod -aG sudo admin
```

**2.5 - Installation et configuration du serveur web Apache**

```bash
# Installer Apache
apt install -y apache2

# Vérifier le statut
systemctl status apache2

# Activer au démarrage
systemctl enable apache2
```

Créer une page d'accueil personnalisée :
```bash
nano /var/www/html/index.html
```

Contenu :
```html
<!DOCTYPE html>
<html>
<head>
    <title>SecureNet - DMZ Web Server</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            margin: 50px;
            background-color: #f0f0f0;
        }
        .container {
            background: white;
            padding: 30px;
            border-radius: 10px;
            box-shadow: 0 0 20px rgba(0,0,0,0.1);
        }
        h1 { color: #2c3e50; }
        .info { 
            background: #ecf0f1;
            padding: 15px;
            margin: 20px 0;
            border-left: 4px solid #3498db;
        }
    </style>
</head>
<body>
    <div class="container">
        <h1>🌐 SecureNet - Serveur Web</h1>
        <div class="info">
            <p><strong>Environnement :</strong> DMZ (VLAN 10)</p>
            <p><strong>Serveur :</strong> WEB-SRV</p>
            <p><strong>IP :</strong> 10.10.10.10</p>
            <p><strong>Date :</strong> <script>document.write(new Date().toLocaleString());</script></p>
        </div>
        <p>✅ Le serveur web est opérationnel</p>
    </div>
</body>
</html>
```

**2.6 - Configuration pfSense pour accès DMZ**

Depuis l'interface pfSense, créez une règle pour autoriser l'accès HTTP depuis votre réseau ADMIN vers la DMZ :

```
Firewall → Rules → ADMIN (VLAN 30)
Add rule :
- Action : Pass
- Protocol : TCP
- Source : ADMIN net (10.30.30.0/24)
- Destination : 10.10.10.10
- Destination Port : 80 (HTTP)
- Description : "Allow ADMIN to DMZ Web Server"
```

**2.7 - Tests et validation**

- [ ] Vérifier que la VM démarre correctement
- [ ] Vérifier la connectivité réseau (`ping 10.10.10.1`, `ping 1.1.1.1`)
- [ ] Vérifier qu'Apache est démarré (`systemctl status apache2`)
- [ ] Accéder au serveur web depuis votre navigateur : `http://10.10.10.10`
- [ ] Vérifier les logs Apache : `tail -f /var/log/apache2/access.log`

**2.8 - Snapshot de sécurité**

Créez un snapshot de la VM dans Proxmox :
```
VM WEB-SRV → Snapshots → Take Snapshot
Nom : "web-srv-base-configured"
Description : "Debian + Apache configuré et fonctionnel"
```

---

## PARTIE 3 : Déploiement du LAN interne - Active Directory (2h)

### Objectif
Créer et configurer un contrôleur de domaine Windows Server.

### Travail à réaliser

**3.1 - Création de la VM DC-SRV**

Créez une nouvelle VM :
- **Nom** : DC-SRV
- **OS** : Windows Server 2022 (ISO)
- **Disque** : 60 GB
- **CPU** : 2 cores
- **RAM** : 4096 MB
- **Réseau** : vmbr20 (VLAN 20 - LAN)

Options importantes :
- Type de machine : q35
- BIOS : OVMF (UEFI)
- Ajouter un TPM (pour Windows 11/Server 2022)

**3.2 - Installation de Windows Server 2022**

Installez Windows Server :
- Édition : Windows Server 2022 Standard (Desktop Experience)
- Type d'installation : Personnalisée
- Partitionnement : utiliser tout l'espace disque
- Définir le mot de passe Administrateur (fort et noté !)

Après l'installation :
- Installer les Guest Tools Proxmox (QEMU Agent)
- Redémarrer

**3.3 - Configuration réseau statique**

Dans Windows Server :
```
Paramètres → Réseau et Internet → Ethernet → Propriétés
IPv4 :
- Adresse IP : 10.20.20.10
- Masque : 255.255.255.0
- Passerelle : 10.20.20.1
- DNS préféré : 127.0.0.1 (lui-même, car il sera DC)
- DNS auxiliaire : 1.1.1.1
```

Renommer l'ordinateur :
```
Gestionnaire de serveur → Serveur local → Nom de l'ordinateur
Nouveau nom : DC-SRV
Redémarrer
```

**3.4 - Installation du rôle Active Directory Domain Services**

Ouvrir PowerShell en tant qu'Administrateur :

```powershell
# Installer le rôle AD DS
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools

# Promouvoir en contrôleur de domaine
Install-ADDSForest `
    -DomainName "securenet.local" `
    -DomainNetbiosName "SECURENET" `
    -ForestMode "WinThreshold" `
    -DomainMode "WinThreshold" `
    -InstallDns `
    -SafeModeAdministratorPassword (ConvertTo-SecureString "VotreMotDePasseDSRM!" -AsPlainText -Force) `
    -Force
```

Le serveur redémarrera automatiquement.

**3.5 - Vérification de l'Active Directory**

Après redémarrage, vérifier :

```powershell
# Vérifier le domaine
Get-ADDomain

# Vérifier le contrôleur de domaine
Get-ADDomainController

# Vérifier le DNS
nslookup securenet.local
```

**3.6 - Création de l'arborescence AD**

Créer une structure organisationnelle :

```powershell
# Créer les OUs principales
New-ADOrganizationalUnit -Name "SecureNet" -Path "DC=securenet,DC=local"
New-ADOrganizationalUnit -Name "Utilisateurs" -Path "OU=SecureNet,DC=securenet,DC=local"
New-ADOrganizationalUnit -Name "Ordinateurs" -Path "OU=SecureNet,DC=securenet,DC=local"
New-ADOrganizationalUnit -Name "Serveurs" -Path "OU=SecureNet,DC=securenet,DC=local"
New-ADOrganizationalUnit -Name "Groupes" -Path "OU=SecureNet,DC=securenet,DC=local"
```

**3.7 - Création d'utilisateurs de test**

```powershell
# Créer des utilisateurs
$Password = ConvertTo-SecureString "P@ssw0rd123!" -AsPlainText -Force

New-ADUser -Name "Alice Martin" `
    -GivenName "Alice" `
    -Surname "Martin" `
    -SamAccountName "amartin" `
    -UserPrincipalName "amartin@securenet.local" `
    -Path "OU=Utilisateurs,OU=SecureNet,DC=securenet,DC=local" `
    -AccountPassword $Password `
    -Enabled $true `
    -ChangePasswordAtLogon $false

New-ADUser -Name "Bob Dupont" `
    -GivenName "Bob" `
    -Surname "Dupont" `
    -SamAccountName "bdupont" `
    -UserPrincipalName "bdupont@securenet.local" `
    -Path "OU=Utilisateurs,OU=SecureNet,DC=securenet,DC=local" `
    -AccountPassword $Password `
    -Enabled $true `
    -ChangePasswordAtLogon $false
```

**3.8 - Création de groupes**

```powershell
# Créer des groupes de sécurité
New-ADGroup -Name "GRP_Admins" `
    -GroupScope Global `
    -GroupCategory Security `
    -Path "OU=Groupes,OU=SecureNet,DC=securenet,DC=local"

New-ADGroup -Name "GRP_Utilisateurs" `
    -GroupScope Global `
    -GroupCategory Security `
    -Path "OU=Groupes,OU=SecureNet,DC=securenet,DC=local"

# Ajouter des membres
Add-ADGroupMember -Identity "GRP_Admins" -Members "amartin"
Add-ADGroupMember -Identity "GRP_Utilisateurs" -Members "amartin","bdupont"
```

**3.9 - Tests et validation**

- [ ] Vérifier que le domaine `securenet.local` est accessible
- [ ] Vérifier les services AD DS et DNS (Services.msc)
- [ ] Lister les utilisateurs créés : `Get-ADUser -Filter *`
- [ ] Vérifier la réplication : `repadmin /replsummary`

**3.10 - Snapshot**

```
Proxmox : DC-SRV → Snapshots → Take Snapshot
Nom : "dc-srv-ad-configured"
Description : "Windows Server 2022 + AD DS + utilisateurs de test"
```

---

## PARTIE 4 : Déploiement du LAN interne - Base de données (1h)

### Objectif
Créer un serveur de base de données MySQL sur Ubuntu.

### Travail à réaliser

**4.1 - Création de la VM DB-SRV**

Créez une nouvelle VM :
- **Nom** : DB-SRV
- **OS** : Ubuntu Server 22.04 LTS (ISO)
- **Disque** : 30 GB
- **CPU** : 2 cores
- **RAM** : 2048 MB
- **Réseau** : vmbr20 (VLAN 20 - LAN)

**4.2 - Installation d'Ubuntu Server**

Installez Ubuntu avec ces paramètres :
- Profil : Nom complet : Administrator / Nom serveur : db-srv / Utilisateur : admin
- Installation OpenSSH : OUI
- Pas de snaps supplémentaires

**4.3 - Configuration réseau statique**

Éditer la configuration Netplan :

```bash
sudo nano /etc/netplan/00-installer-config.yaml
```

Contenu :
```yaml
network:
  version: 2
  ethernets:
    ens18:
      addresses:
        - 10.20.20.20/24
      routes:
        - to: default
          via: 10.20.20.1
      nameservers:
        addresses:
          - 10.20.20.10  # DC-SRV (DNS)
          - 1.1.1.1
        search:
          - securenet.local
```

Appliquer :
```bash
sudo netplan apply
```

Vérifier :
```bash
ip addr show
ping -c 3 10.20.20.10  # Ping DC-SRV
ping -c 3 10.20.20.1   # Ping pfSense
```

**4.4 - Mise à jour et installation MySQL**

```bash
# Mise à jour
sudo apt update && sudo apt upgrade -y

# Installer MySQL
sudo apt install -y mysql-server

# Vérifier le statut
sudo systemctl status mysql
```

**4.5 - Sécurisation de MySQL**

```bash
sudo mysql_secure_installation
```

Répondre :
- VALIDATE PASSWORD COMPONENT : Yes (niveau Medium)
- Mot de passe root : définir un mot de passe fort
- Remove anonymous users : Yes
- Disallow root login remotely : No (on veut accéder depuis le LAN)
- Remove test database : Yes
- Reload privilege tables : Yes

**4.6 - Configuration de MySQL pour accès réseau**

```bash
# Éditer la configuration
sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
```

Modifier la ligne :
```
bind-address = 0.0.0.0
```

Redémarrer MySQL :
```bash
sudo systemctl restart mysql
```

**4.7 - Création d'une base de données de test**

```bash
# Se connecter à MySQL
sudo mysql
```

Dans MySQL :
```sql
-- Créer une base de données
CREATE DATABASE securenet_app;

-- Créer un utilisateur avec accès depuis le réseau LAN
CREATE USER 'appuser'@'10.20.20.%' IDENTIFIED BY 'AppPassword123!';

-- Donner tous les droits sur la base
GRANT ALL PRIVILEGES ON securenet_app.* TO 'appuser'@'10.20.20.%';

-- Appliquer les changements
FLUSH PRIVILEGES;

-- Créer une table de test
USE securenet_app;

CREATE TABLE users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    username VARCHAR(50) NOT NULL,
    email VARCHAR(100) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Insérer des données de test
INSERT INTO users (username, email) VALUES 
    ('alice', 'alice@securenet.local'),
    ('bob', 'bob@securenet.local');

-- Vérifier
SELECT * FROM users;

-- Quitter
EXIT;
```

**4.8 - Tests et validation**

Depuis DB-SRV :
```bash
# Tester la connexion locale
mysql -u appuser -p securenet_app

# Vérifier l'écoute réseau
sudo netstat -tlnp | grep mysql
```

- [ ] MySQL écoute sur 0.0.0.0:3306
- [ ] Base de données `securenet_app` créée
- [ ] Utilisateur `appuser` peut se connecter
- [ ] Données de test présentes

**4.9 - Snapshot**

```
Proxmox : DB-SRV → Snapshots → Take Snapshot
Nom : "db-srv-mysql-configured"
Description : "Ubuntu Server + MySQL + base de test"
```

---

## PARTIE 5 : Poste d'administration (1h)

### Objectif
Créer un poste Windows pour administrer l'infrastructure.

### Travail à réaliser

**5.1 - Création de la VM ADMIN-WKS**

Créez une nouvelle VM :
- **Nom** : ADMIN-WKS
- **OS** : Windows 10 ou 11 (ISO)
- **Disque** : 50 GB
- **CPU** : 2 cores
- **RAM** : 4096 MB
- **Réseau** : vmbr30 (VLAN 30 - ADMIN)

**5.2 - Installation de Windows**

Installez Windows normalement :
- Compte local : Administrateur
- Personnalisation : désactiver tout ce qui n'est pas nécessaire

**5.3 - Configuration réseau statique**

```
Paramètres → Réseau → Ethernet → Propriétés
IPv4 :
- Adresse IP : 10.30.30.10
- Masque : 255.255.255.0
- Passerelle : 10.30.30.1
- DNS préféré : 10.20.20.10 (DC-SRV)
- DNS auxiliaire : 1.1.1.1
```

Renommer le PC : `ADMIN-WKS`

**5.4 - Jonction au domaine Active Directory**

```
Paramètres → Comptes → Accès Professionnel ou Scolaire
→ Se connecter → Joindre ce PC à un domaine Active Directory local

Domaine : securenet.local
Utilisateur : amartin
Mot de passe : P@ssw0rd123!

Type de compte : Administrateur
Redémarrer
```

**5.5 - Installation des outils d'administration**

Après redémarrage, se connecter avec `SECURENET\amartin` :

Installer RSAT (Remote Server Administration Tools) :
```
Paramètres → Applications → Fonctionnalités facultatives
→ Ajouter une fonctionnalité

Chercher et installer :
- RSAT: Active Directory Domain Services Tools
- RSAT: DNS Server Tools
- RSAT: DHCP Server Tools
```

**5.6 - Installation d'outils complémentaires**

Télécharger et installer :
- **Navigateur** : Firefox ou Chrome
- **MySQL Workbench** : pour gérer la base de données
- **Putty** ou **Windows Terminal** : pour SSH vers serveurs Linux
- **Notepad++** : éditeur de texte avancé

**5.7 - Tests de connectivité et d'administration**

Depuis ADMIN-WKS, tester :

**Test 1 : Active Directory**
```
Menu Démarrer → Outils d'administration Windows
→ Utilisateurs et ordinateurs Active Directory

Vérifier que vous voyez :
- Le domaine securenet.local
- Les OUs créées
- Les utilisateurs amartin et bdupont
```

**Test 2 : Serveur Web (DMZ)**
```
Ouvrir navigateur → http://10.10.10.10
Vérifier que la page d'accueil s'affiche
```

**Test 3 : Base de données (LAN)**
```
Ouvrir MySQL Workbench
→ Nouvelle connexion :
   Hostname : 10.20.20.20
   Port : 3306
   Username : appuser
   Password : AppPassword123!

Se connecter et vérifier la base securenet_app
```

**Test 4 : SSH vers serveurs Linux**
```
Ouvrir PowerShell ou Windows Terminal

# Vers WEB-SRV
ssh admin@10.10.10.10

# Vers DB-SRV
ssh admin@10.20.20.20
```

**5.8 - Validation**

- [ ] Poste joint au domaine securenet.local
- [ ] Connexion possible avec amartin
- [ ] Outils RSAT installés et fonctionnels
- [ ] Accès au serveur web DMZ
- [ ] Connexion MySQL réussie
- [ ] SSH fonctionnel vers serveurs Linux

**5.9 - Snapshot**

```
Proxmox : ADMIN-WKS → Snapshots → Take Snapshot
Nom : "admin-wks-domain-joined"
Description : "Windows 10/11 + domaine + outils admin"
```

---

## PARTIE 6 : Sécurisation et règles de firewall (1h)

### Objectif
Implémenter les règles de sécurité entre les différents VLANs.

### Travail à réaliser

**6.1 - Matrice de flux réseau**

Documenter les flux autorisés :

| Source | Destination | Service | Port | Justification |
|--------|-------------|---------|------|---------------|
| ADMIN | DMZ | HTTP | 80 | Administration web |
| ADMIN | DMZ | SSH | 22 | Administration serveur |
| ADMIN | LAN | RDP | 3389 | Administration Windows |
| ADMIN | LAN | SSH | 22 | Administration serveurs |
| ADMIN | LAN | MySQL | 3306 | Gestion BDD |
| ADMIN | LAN | DNS | 53 | Résolution noms |
| ADMIN | LAN | LDAP | 389/636 | Requêtes AD |
| LAN | DMZ | HTTP | 80 | Accès web interne |
| LAN | Internet | HTTP/HTTPS | 80/443 | Mises à jour |
| DMZ | Internet | HTTP/HTTPS | 80/443 | Mises à jour |
| DMZ | LAN | ❌ INTERDIT | - | Segmentation sécurité |
| LAN → ADMIN | ❌ INTERDIT | - | Protection admin |

**6.2 - Configuration des règles pfSense**

Connectez-vous à pfSense et configurez les règles pour chaque VLAN.

**Règles VLAN ADMIN (10.30.30.0/24)** :

```
Firewall → Rules → ADMIN

1. Allow ADMIN to DMZ Web
   - Action : Pass
   - Protocol : TCP
   - Source : ADMIN net
   - Destination : 10.10.10.10
   - Dest Port : HTTP (80)

2. Allow ADMIN to DMZ SSH
   - Action : Pass
   - Protocol : TCP
   - Source : ADMIN net
   - Destination : 10.10.10.10
   - Dest Port : SSH (22)

3. Allow ADMIN to LAN Services
   - Action : Pass
   - Protocol : TCP
   - Source : ADMIN net
   - Destination : LAN net (10.20.20.0/24)
   - Dest Port : Multiple (22, 3389, 3306, 53, 389, 636)

4. Allow ADMIN to Internet
   - Action : Pass
   - Protocol : TCP/UDP
   - Source : ADMIN net
   - Destination : any
   - Dest Port : any
```

**Règles VLAN LAN (10.20.20.0/24)** :

```
Firewall → Rules → LAN

1. Allow LAN to DMZ Web
   - Action : Pass
   - Protocol : TCP
   - Source : LAN net
   - Destination : 10.10.10.10
   - Dest Port : HTTP (80)

2. Allow LAN DNS to Internet
   - Action : Pass
   - Protocol : UDP
   - Source : 10.20.20.10 (DC-SRV)
   - Destination : any
   - Dest Port : DNS (53)

3. Allow LAN Updates
   - Action : Pass
   - Protocol : TCP
   - Source : LAN net
   - Destination : any
   - Dest Port : HTTP/HTTPS (80, 443)

4. Block LAN to ADMIN
   - Action : Block
   - Protocol : any
   - Source : LAN net
   - Destination : ADMIN net
   - Description : "Protect admin network"
```

**Règles VLAN DMZ (10.10.10.0/24)** :

```
Firewall → Rules → DMZ

1. Allow DMZ Updates
   - Action : Pass
   - Protocol : TCP
   - Source : DMZ net
   - Destination : any
   - Dest Port : HTTP/HTTPS (80, 443)

2. Block DMZ to LAN
   - Action : Block
   - Protocol : any
   - Source : DMZ net
   - Destination : LAN net
   - Description : "DMZ isolation"

3. Block DMZ to ADMIN
   - Action : Block
   - Protocol : any
   - Source : DMZ net
   - Destination : ADMIN net
   - Description : "DMZ isolation"
```

**6.3 - Tests de sécurité**

Depuis ADMIN-WKS, tester tous les flux :

**Tests positifs (doivent fonctionner)** :
```powershell
# Vers DMZ
Test-NetConnection -ComputerName 10.10.10.10 -Port 80
Test-NetConnection -ComputerName 10.10.10.10 -Port 22

# Vers LAN
Test-NetConnection -ComputerName 10.20.20.10 -Port 3389
Test-NetConnection -ComputerName 10.20.20.20 -Port 3306
Test-NetConnection -ComputerName 10.20.20.20 -Port 22
```

**Tests négatifs (doivent échouer)** :

Depuis WEB-SRV (DMZ), essayer de joindre le LAN :
```bash
# Doit échouer
ping 10.20.20.10
telnet 10.20.20.20 3306
```

Depuis DB-SRV (LAN), essayer de joindre ADMIN :
```bash
# Doit échouer
ping 10.30.30.10
```

**6.4 - Validation de la segmentation**

Créer un tableau de validation :

| Test | Source | Destination | Résultat attendu | Résultat obtenu | Statut |
|------|--------|-------------|------------------|-----------------|--------|
| HTTP DMZ | ADMIN | 10.10.10.10:80 | ✅ Succès | | |
| SSH DMZ | ADMIN | 10.10.10.10:22 | ✅ Succès | | |
| MySQL | ADMIN | 10.20.20.20:3306 | ✅ Succès | | |
| Ping LAN | DMZ | 10.20.20.10 | ❌ Échec | | |
| SSH LAN | DMZ | 10.20.20.20:22 | ❌ Échec | | |
| Ping ADMIN | LAN | 10.30.30.10 | ❌ Échec | | |

---

## PARTIE 7 : Documentation et sauvegarde (30 min)

### Objectif
Documenter l'infrastructure et mettre en place une stratégie de sauvegarde.

### Travail à réaliser

**7.1 - Documentation complète de l'infrastructure**

Créez un document Word/PDF contenant :

**Page 1 : Vue d'ensemble**
- Schéma réseau (dessiné à la main ou avec draw.io)
- Liste des VMs avec rôles
- Plan d'adressage IP complet

**Page 2 : Inventaire des VMs**

Pour chaque VM :
- Nom et rôle
- Système d'exploitation
- Ressources (CPU, RAM, Disque)
- Configuration réseau
- Services installés
- Identifiants de connexion
- Date de création
- Snapshots disponibles

**Page 3 : Configuration réseau et sécurité**
- VLANs et leurs rôles
- Matrice de flux réseau
- Règles de firewall implémentées
- Points de vigilance sécurité

**Page 4 : Procédures**
- Procédure de connexion à chaque serveur
- Procédure de restauration depuis snapshot
- Procédure d'ajout d'utilisateur AD
- Procédure de sauvegarde MySQL

**7.2 - Export de la configuration pfSense**

Dans pfSense :
```
Diagnostics → Backup & Restore → Backup Configuration
→ Download configuration as XML
```

Sauvegarder ce fichier dans votre documentation.

**7.3 - Stratégie de snapshots**

Documenter votre stratégie de snapshots :

| VM | Fréquence | Rétention | Déclencheur |
|------|-----------|-----------|-------------|
| WEB-SRV | Avant chaque modification majeure | 3 snapshots | Manuel |
| DC-SRV | Hebdomadaire + avant modifs AD | 4 snapshots | Manuel |
| DB-SRV | Quotidien + avant modifs BDD | 7 snapshots | Manuel |
| ADMIN-WKS | Avant installation logiciels | 2 snapshots | Manuel |

**7.4 - Sauvegarde des données MySQL**

Sur DB-SRV, créer un script de sauvegarde :

```bash
sudo nano /usr/local/bin/backup-mysql.sh
```

Contenu :
```bash
#!/bin/bash

# Configuration
BACKUP_DIR="/backup/mysql"
DATE=$(date +%Y%m%d_%H%M%S)
MYSQL_USER="root"
MYSQL_PASSWORD="VotreMotDePasseRoot"

# Créer le dossier si inexistant
mkdir -p $BACKUP_DIR

# Sauvegarde
mysqldump -u $MYSQL_USER -p$MYSQL_PASSWORD --all-databases > $BACKUP_DIR/all_databases_$DATE.sql

# Compression
gzip $BACKUP_DIR/all_databases_$DATE.sql

# Nettoyage (garder 7 jours)
find $BACKUP_DIR -name "*.sql.gz" -mtime +7 -delete

echo "[$(date)] Sauvegarde MySQL effectuée : all_databases_$DATE.sql.gz"
```

Rendre exécutable :
```bash
sudo chmod +x /usr/local/bin/backup-mysql.sh
```

Tester :
```bash
sudo /usr/local/bin/backup-mysql.sh
```

**7.5 - Tests de restauration**

**Test 1 : Restauration snapshot**

Sur une VM de test (WEB-SRV par exemple) :
1. Faire une modification (supprimer le fichier index.html)
2. Vérifier que le site ne fonctionne plus
3. Restaurer le snapshot dans Proxmox
4. Vérifier que le site fonctionne à nouveau

**Test 2 : Restauration MySQL**

```bash
# Supprimer la base de test
sudo mysql -e "DROP DATABASE securenet_app;"

# Vérifier qu'elle n'existe plus
sudo mysql -e "SHOW DATABASES;"

# Restaurer depuis la sauvegarde
sudo gunzip < /backup/mysql/all_databases_[DATE].sql.gz | sudo mysql

# Vérifier la restauration
sudo mysql -e "USE securenet_app; SELECT * FROM users;"
```

**7.6 - Checklist finale de validation**

- [ ] Documentation complète rédigée
- [ ] Schéma réseau créé
- [ ] Configuration pfSense exportée
- [ ] Snapshots créés pour toutes les VMs
- [ ] Script de sauvegarde MySQL testé
- [ ] Procédure de restauration testée et validée
- [ ] Identifiants documentés de manière sécurisée

---

## PARTIE 8 : Scenarios avancés (Bonus - si temps restant)

### Objectif
Aller plus loin dans l'administration de l'infrastructure.

### Travail à réaliser

**8.1 - Configuration HTTPS sur WEB-SRV**

Installer un certificat auto-signé :

```bash
# Installer OpenSSL
sudo apt install -y openssl

# Générer certificat auto-signé
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/ssl/private/apache-selfsigned.key \
  -out /etc/ssl/certs/apache-selfsigned.crt

# Activer SSL
sudo a2enmod ssl
sudo a2ensite default-ssl

# Redémarrer Apache
sudo systemctl restart apache2
```

Tester : `https://10.10.10.10`

**8.2 - Configuration de partage de fichiers sur DC-SRV**

```powershell
# Créer un dossier partagé
New-Item -Path "C:\Partages\Commun" -ItemType Directory

# Partager le dossier
New-SmbShare -Name "Commun" -Path "C:\Partages\Commun" -FullAccess "SECURENET\GRP_Utilisateurs"

# Définir les permissions NTFS
$acl = Get-Acl "C:\Partages\Commun"
$permission = "SECURENET\GRP_Utilisateurs","Modify","Allow"
$accessRule = New-Object System.Security.AccessControl.FileSystemAccessRule $permission
$acl.SetAccessRule($accessRule)
Set-Acl "C:\Partages\Commun" $acl
```

Tester depuis ADMIN-WKS : `\\DC-SRV\Commun`

**8.3 - Monitoring avec un dashboard simple**

Sur ADMIN-WKS, créer un dashboard HTML qui affiche l'état des services :

```html
<!DOCTYPE html>
<html>
<head>
    <title>SecureNet - Dashboard Infrastructure</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            background: #2c3e50;
            color: white;
            padding: 20px;
        }
        .container {
            max-width: 1200px;
            margin: 0 auto;
        }
        .service {
            background: #34495e;
            padding: 20px;
            margin: 10px 0;
            border-radius: 8px;
            display: flex;
            justify-content: space-between;
            align-items: center;
        }
        .status {
            padding: 5px 15px;
            border-radius: 4px;
            font-weight: bold;
        }
        .online { background: #27ae60; }
        .offline { background: #e74c3c; }
    </style>
</head>
<body>
    <div class="container">
        <h1>🖥️ Dashboard Infrastructure SecureNet</h1>
        <div class="service">
            <div>
                <h3>WEB-SRV (DMZ)</h3>
                <p>Apache Web Server - 10.10.10.10</p>
            </div>
            <div class="status online">ONLINE</div>
        </div>
        <div class="service">
            <div>
                <h3>DC-SRV (LAN)</h3>
                <p>Active Directory - 10.20.20.10</p>
            </div>
            <div class="status online">ONLINE</div>
        </div>
        <div class="service">
            <div>
                <h3>DB-SRV (LAN)</h3>
                <p>MySQL Database - 10.20.20.20</p>
            </div>
            <div class="status online">ONLINE</div>
        </div>
    </div>
</body>
</html>
```

**8.4 - Automatisation avec scripts**

Créer un script PowerShell qui vérifie l'état de toutes les VMs :

```powershell
# check-infrastructure.ps1

$services = @(
    @{Name="WEB-SRV"; IP="10.10.10.10"; Port=80},
    @{Name="DC-SRV"; IP="10.20.20.10"; Port=3389},
    @{Name="DB-SRV"; IP="10.20.20.20"; Port=3306}
)

Write-Host "=== Vérification Infrastructure SecureNet ===" -ForegroundColor Cyan
Write-Host ""

foreach ($service in $services) {
    Write-Host "Test de $($service.Name) ($($service.IP):$($service.Port))..." -NoNewline
    
    $result = Test-NetConnection -ComputerName $service.IP -Port $service.Port -WarningAction SilentlyContinue
    
    if ($result.TcpTestSucceeded) {
        Write-Host " ✅ OK" -ForegroundColor Green
    } else {
        Write-Host " ❌ KO" -ForegroundColor Red
    }
}
```

---

## Ressources complémentaires

### Documentation

- **Proxmox** : https://pve.proxmox.com/wiki/
- **pfSense** : https://docs.netgate.com/pfsense/
- **Active Directory** : https://docs.microsoft.com/windows-server/identity/
- **MySQL** : https://dev.mysql.com/doc/



