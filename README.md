# Segmentation et sécurisation d'un reseau LAN multi-VLAN
### pfSense - Snort IDS - pfBlockerNG - GNS3 - Cisco IOU

Un laboratoire reseau complet ou chaque brique a ete configuree, puis testee. Pas juste supposee fonctionnelle.

---

## L'histoire derriere ce projet

Dans une petite entreprise, on ne met pas tout le monde sur le meme reseau plat. Les postes clients, les serveurs, les services critiques... chacun doit vivre dans son propre espace, avec des regles claires sur qui peut parler a qui.

C'est exactement ce que ce laboratoire reproduit a petite echelle.

L'idee etait simple sur le papier : construire trois VLANs distincts, les relier via un pare-feu pfSense qui fait aussi office de routeur, ajouter une sonde de detection d'intrusion (Snort) pour observer ce qui se passe, et mettre en place un filtrage DNS (pfBlockerNG) pour couper l'acces a certains sites avant meme qu'une connexion soit etablie.

Rien de theorique ici. Chaque regle a ete ecrite avec une intention precise. Chaque fonctionnalite a ete verifiee par un test concret.

---

## Architecture generale

```
                    +-------------------------------------------+
                    |         PFSENSE (Pare-feu)                 |
                    |  WAN: 192.168.130.129                      |
                    |  OPT1: 10.0.10.1 (VLAN 10)                |
                    |  OPT2: 10.0.20.1 (VLAN 20)                |
                    |  OPT3: 10.0.30.1 (VLAN 30)                |
                    +--------------------+----------------------+
                                         | Trunk 802.1Q (em1 vers Et0/0)
                                         | VLANs 1, 10, 20, 30
                    +--------------------+----------------------+
                    |    COMMUTATEUR CISCO IOU (IOU1)            |
                    |  Ports acces par VLAN :                    |
                    |  VLAN 10 -> Et1/0, Et1/1, Et1/2            |
                    |  VLAN 20 -> Et2/0, Et2/1, Et2/2            |
                    |  VLAN 30 -> Et3/0, Et3/1, Et3/2            |
                    +----+----------------+---------------------+
                         |                |
                   +-----+----+     +-----+----+     +----------+
                   | VLAN 10  |     | VLAN 20  |     | VLAN 30  |
                   | Clients  |     | Clients  |     | Serveur  |
                   | Segment1 |     | Segment2 |     | Web      |
                   | /24      |     | /24      |     | /24      |
                   +----------+     +----------+     +----------+
```

Le principe : un seul lien trunk entre le commutateur et le pare-feu transporte les trois VLANs. Cote pfSense, trois sous-interfaces taguess (em1.10, em1.20, em1.30) permettent d'appliquer des regles de filtrage differentes a chaque segment.

---

## Ce que ce projet demontre concretement

### Segmentation VLAN fonctionnelle
- 3 VLANs : clients (10), clients bis (20), serveur (30)
- Trunk 802.1Q propre entre le commutateur Cisco IOU et pfSense
- Routage inter-VLAN assure par pfSense

### Politique de filtrage reflechie (pas une config par defaut)
- VLAN 10 et VLAN 20 : isolation totale. Les deux segments clients ne peuvent pas communiquer entre eux.
- VLAN 10 vers VLAN 30 : acces autorise. Les clients peuvent atteindre le serveur.
- VLAN 20 vers VLAN 30 : acces autorise. Meme chose.
- Acces Internet : HTTP et HTTPS autorises pour les clients.
- DNS : resolution via la passerelle pfSense uniquement.

### Detection d'intrusion avec Snort (mode IDS)
- Mode observation uniquement (pas de blocage automatique).
- Regles Emerging Threats activees : ICMP, ICMP info, detection de scan.
- Un scan nmap est correctement detecte et journalise.

### Filtrage DNS avec pfBlockerNG
- Blocage des publicites (liste ADs_Basic).
- Blocage des reseaux sociaux (liste personnalisee : Facebook, Instagram, TikTok, X, etc.).
- La liste inclut les domaines CDN (fbcdn.net, tiktokcdn.com...) parce que bloquer facebook.com seul ne suffit pas face aux applis mobiles.
- Page de blocage DNSBL personnalisee affichee dans le navigateur.

### Tests valides - chaque brique a ete verifiee

| Numero | Test | Resultat |
|--------|------|----------|
| 1 | Ping intra-VLAN (deb-1 vers deb-2) | Reussi |
| 2 | Ping inter-VLAN bloque (VLAN 10 vers VLAN 20) | Bloque |
| 3 | Acces serveur preserve (VLAN 20 vers VLAN 30) | Reussi |
| 4 | Resolution DNS via pfSense | Fonctionnelle |
| 5 | Acces site web par IP | Page "Sentinel" affichee |
| 6 | Acces site web par nom de domaine | Resolution OK |
| 7 | Scan nmap sur interface WAN | Ports 22 et 80 ouverts |
| 8 | Detection du scan par Snort | Alertes ET SCAN generees |
| 9 | Blocage Instagram par pfBlockerNG | Page de blocage affichee |

---

## Galerie

### Topologie du laboratoire
![Topologie GNS3](images/topologie-gns3.png)

### Interfaces et VLANs pfSense
![Interfaces pfSense](images/interfaces-pfsense.png)
![VLANs pfSense](images/vlans-pfsense.png)

### Politique de filtrage

**WAN**
![Regles WAN](images/rules-WAN-pfsense.png)

**OPT1 (VLAN 10)**
![Regles OPT1](images/rules-OPT1-pfsense.png)

**OPT2 (VLAN 20)**
![Regles OPT2](images/rules-OPT2-pfsense.png)

**OPT3 (VLAN 30)**
![Regles OPT3](images/rules-OPT3-pfsense.png)

### Commutation Cisco IOU

**show vlan brief**
![VLAN brief](images/show-vlan-brief-gns3.png)

**show interface trunk**
![Interface trunk](images/show-interface-trunk.png)

**show mac address-table**
![MAC table](images/show-mac-address-table.png)

### Snort - Detection d'intrusion
![Snort configure en mode IDS](images/snort-configure-en-mode-IDS.png)
![Regles Snort installees](images/regles-snort-installes.png)
![Categories Snort activees](images/regles-snort-activees.png)

### pfBlockerNG - Filtrage DNS
![Groupes de domaines](images/groupes-de-domaines-pfblockerng.png)
![Domaines bloques](images/domaines-bloques-par-pfblockerng.png)

### Dashboard pfSense
![Dashboard](images/pfesense-dashboard.png)

### Tests en action

| Test | Capture |
|------|---------|
| Adressage et ping (deb-1) | ![Test 1](images/tests/deb-1-ip-et-test-de-ping.png) |
| Isolation VLAN (Windows) | ![Test 2](images/tests/win-vlan10-ip-et-isolation-vlan.png) |
| Isolation + acces serveur (deb-3) | ![Test 3](images/tests/deb3-ip-et-isolation-intervlan-mais-access-serveur.png) |
| pfSense DNS + routeur | ![Test 4](images/tests/pfsense-servant-de-dns-et-routeur.png) |
| Acces serveur par IP | ![Test 5](images/tests/visite-du-site-du-serveur-deb-5-depuis-windows.png) |
| Acces serveur par domaine | ![Test 6](images/tests/visite-du-site-de-deb-5-mais-avec-son-nom-de-domaine.png) |
| Scan nmap sur WAN | ![Test 7](images/tests/scan-nmap-sur-interface-wan-de-pfsense.png) |
| Snort detecte le scan | ![Test 8](images/tests/snort-en-mode-ids-detecte-le-scan.png) |
| Blocage Instagram | ![Test 9](images/tests/affichage-de-la-page-custom-pfblockerng(instagram.com).png) |

---

## Technologies utilisees

| Technologie | Role |
|-------------|------|
| pfSense CE | Pare-feu / routeur inter-VLAN |
| GNS3 | Plateforme de simulation reseau |
| Cisco IOU L2 | Commutateur virtuel (distribution VLAN) |
| Snort | Detection d'intrusion (IDS) |
| pfBlockerNG | Filtrage DNS (DNSBL) |
| VMware Workstation | Machines virtuelles clientes |
| Debian | Postes clients (deb-1 a deb-4) |
| Debian | Serveur web (deb-5) |
| Windows 10 | Postes clients pour tests applicatifs |

---

## Plan d'adressage

| Hote | VLAN | Adresse | Role |
|------|------|---------|------|
| pfsense-1 (WAN) | - | 192.168.130.129 | Interface externe |
| pfsense-1 (OPT1) | 10 | 10.0.10.1 | Passerelle VLAN 10 |
| pfsense-1 (OPT2) | 20 | 10.0.20.1 | Passerelle VLAN 20 |
| pfsense-1 (OPT3) | 30 | 10.0.30.1 | Passerelle VLAN 30 |
| deb-1 | 10 | 10.0.10.2 | Poste client |
| deb-2 | 10 | 10.0.10.3 | Poste client |
| Windows10x64-1 | 10 | 10.0.10.4 | Poste client (tests) |
| deb-4 | 20 | 10.0.20.2 | Poste client |
| deb-3 | 20 | 10.0.20.3 | Poste client |
| Windows10x64-2 | 20 | 10.0.20.4 | Poste client (tests) |
| deb-5 | 30 | 10.0.30.2 | Serveur web |
| IOU1 | Trunk | Et0/0 vers em1 | Commutateur distribution |

---

## Ce que ce projet n'est pas

Un laboratoire, ce n'est pas une production. Voici les limites identifiees, qui constituent des pistes d'amelioration naturelles :

- Administration exposee sur le WAN : SSH et HTTPS accessibles sans restriction de source. C'est acceptable dans un lab ferme, mais a ne pas reproduire en production.
- Couverture Snort restreinte : seulement 3 categories de regles activees. C'est suffisant pour la demonstration, pas pour de la production.
- Aucune redondance : un seul pare-feu, c'est un point de defaillance unique. Une configuration CARP avec un second pfSense serait la suite logique.
- Journalisation non centralisee : tout est consulte dans l'interface pfSense. Un outil comme Graylog ou ELK serait le bienvenu.

---

## Pistes d'amelioration

- Restreindre l'acces a l'administration du pare-feu (source IP specifique ou VPN)
- Activer des categories supplementaires de regles Snort (malware, exploit, policy...)
- Mettre en place une configuration CARP avec un second pfSense en haute disponibilite
- Centraliser les journaux (Snort + pfBlockerNG + pfSense) vers un serveur Graylog ou ELK
- Passer Snort en mode IPS (blocage automatique) apres analyse des alertes
- Automatiser le deploiement avec des scripts ou Ansible

---

## Lecon retenue

Configurer une regle ne suffit pas a prouver qu'elle produit l'effet attendu.

Ce qui distingue une configuration qui a l'air correcte d'une configuration dont on a verifie le comportement reel, c'est le test.

Le couple scan nmap et alerte Snort en est l'illustration parfaite : sans ce test, on aurait suppose que Snort fonctionnait. Avec ce test, on en a la preuve.

---

## Auteur

BOUMBIEGOU-LARDJA Tiyab Donald
Administration Systemes et Reseaux - 2e annee


Lome, Togo - Juillet 2026

---

## Licence

Ce projet est un travail academique librement inspirant. Vous pouvez reproduire, modifier, ameliorer. Tant que vous testez ce que vous configurez.

---

*Un reseau bien segmente, c'est comme un appartement bien range : vous savez exactement ce qui se trouve dans chaque piece, et qui a les cles de quelle porte.*
