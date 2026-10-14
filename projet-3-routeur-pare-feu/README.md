# Projet 3 — Mise en place d'un routeur/pare-feu pour une petite entreprise

**Cours :** Networking Essentials — Infrastructure informatique et réseau d'une petite entreprise
**Classe :** E1A
**Auteur :** Drilon Nuhija
**Date :** 12 octobre 2026
**Outils :** GNS3 (serveur local), routeur Cisco c3725 (Dynamips), VPCS 0.6.2, réseau NAT VMware (VMnet8)

---

## Contexte

Une petite entreprise veut connecter son réseau local à Internet de façon sécurisée, avec un pare-feu basique. Le réseau du Projet 1 (8 postes, `192.168.10.0/24`) est repris. Le routeur R1 reçoit une interface WAN vers Internet, fait du NAT pour les postes et filtre le trafic avec une liste de contrôle d'accès (ACL).

### Résumé des résultats

| Élément | Résultat |
|---|---|
| Interface LAN | R1 Fa0/0, `192.168.10.1/24`, `up / up` ✅ |
| Interface WAN | R1 Fa0/1, `192.168.0.8/24` reçue par DHCP, route par défaut via `192.168.0.2` ✅ |
| NAT | Le LAN sort sous l'adresse WAN de R1 (NAT overload) ✅ |
| Pare-feu | ACL `PAREFEU-LAN` : HTTP (TCP 80) bloqué, reste du trafic autorisé ✅ |
| Accès Internet depuis PC1 | DNS ✅, HTTPS (TCP 443) ✅ |
| Blocage | TCP 80 : « Communication administratively prohibited » ✅ |

> 📸 Les captures servent d'illustration. Toutes les valeurs utiles sont aussi écrites en texte, dans les tableaux.

---

## 1. Configurer l'interface WAN et LAN du routeur

### 1.1 Plan d'adressage

| Équipement | Interface | Rôle | Adresse | Obtenue par |
|---|---|---|---|---|
| R1 | Fa0/0 | LAN (côté bureau) | `192.168.10.1/24` | Statique |
| R1 | Fa0/1 | WAN (côté Internet) | `192.168.0.8/24` | DHCP |
| Passerelle Internet | — | NAT VMware (VMnet8) | `192.168.0.2` | — |
| PC1 à PC8 | Ethernet0 | Postes | `192.168.10.11` à `.18/24`, passerelle `192.168.10.1` | Statique |
| DNS | — | Résolution des noms | `8.8.8.8` | Configuré sur PC1 |

### 1.2 Maquette

Le nuage GNS3 `INTERNET` est relié à l'adaptateur **VMware Network Adapter VMnet8** du PC. VMnet8 est le réseau NAT de VMware : il fournit une adresse par DHCP et fait sortir le trafic sur Internet par la connexion du PC. Dans cette maquette, il joue le rôle de la box d'un fournisseur d'accès.

Un premier essai sur la carte Wi-Fi du PC n'a pas fonctionné (incident 1).

| Lien | De | Vers |
|---|---|---|
| WAN | R1 Fa0/1 | INTERNET (VMnet8) |
| LAN | R1 Fa0/0 | SW1 e8 |
| Postes | PC1 à PC8 e0 | SW1 e0 à e7 |

📸 *Capture 01 — Maquette : INTERNET (VMnet8) – R1 – SW1 – 8 postes*

![Capture 01 — Maquette : INTERNET (VMnet8) – R1 – SW1 – 8 postes](https://hackmd.io/_uploads/ByRO3B9sMg.png)

### 1.3 Interface LAN

Configurée au Projet 1 et conservée :

```
interface FastEthernet0/0
 description LAN bureau 8 postes
 ip address 192.168.10.1 255.255.255.0
 no shutdown
```

### 1.4 Interface WAN

```
R1#configure terminal
R1(config)#interface fa0/1
R1(config-if)#description WAN vers Internet (box)
R1(config-if)#ip address dhcp
R1(config-if)#no shutdown
R1(config-if)#end
```

`ip address dhcp` : l'interface demande son adresse au fournisseur, comme une box chez un client. Le DHCP donne aussi la passerelle, que R1 installe comme route par défaut.

```
%DHCP-6-ADDRESS_ASSIGN: Interface FastEthernet0/1 assigned DHCP address 192.168.0.8, mask 255.255.255.0, hostname R1
```

📸 *Capture 02 — Configuration de Fa0/1 en client DHCP*

![Capture 02 — Configuration de Fa0/1 en client DHCP](https://hackmd.io/_uploads/Hk0O3B9jfg.png)

📸 *Capture 03 — Fa0/1 reçoit l'adresse 192.168.0.8 par DHCP*

![Capture 03 — Fa0/1 reçoit l'adresse 192.168.0.8 par DHCP](https://hackmd.io/_uploads/HkCdnS9iGx.png)

### 1.5 Vérification

```
R1#show ip interface brief
Interface          IP-Address      OK? Method Status   Protocol
FastEthernet0/0    192.168.10.1    YES NVRAM  up       up
FastEthernet0/1    192.168.0.8     YES DHCP   up       up

R1#show ip route
Gateway of last resort is 192.168.0.2 to network 0.0.0.0

C    192.168.10.0/24 is directly connected, FastEthernet0/0
C    192.168.0.0/24 is directly connected, FastEthernet0/1
S*   0.0.0.0/0 [254/0] via 192.168.0.2
```

| Ligne | Signification |
|---|---|
| `C 192.168.10.0/24` | Réseau du LAN, directement connecté |
| `C 192.168.0.0/24` | Réseau du WAN, directement connecté |
| `S* 0.0.0.0/0 via 192.168.0.2` | Route par défaut apprise par DHCP : tout ce qui n'est pas local part vers Internet |

R1 joint Internet :

```
R1#ping 8.8.8.8
.!!!!
Success rate is 80 percent (4/5), round-trip min/avg/max = 100/112/136 ms
```

Le premier point est la résolution ARP de la passerelle, normale au premier envoi.

📸 *Capture 04 — R1 : LAN et WAN up/up, route par défaut via 192.168.0.2*

![Capture 04 — R1 : LAN et WAN up/up, route par défaut via 192.168.0.2](https://hackmd.io/_uploads/HyA_3Scifx.png)

📸 *Capture 05 — R1 joint 8.8.8.8 (4 réponses sur 5)*

![Capture 05 — R1 joint 8.8.8.8 (4 réponses sur 5)](https://hackmd.io/_uploads/SJR_3HcjGl.png)

---

## 2. Activer le NAT pour l'accès Internet

### 2.1 Pourquoi le NAT

Les adresses `192.168.10.x` sont **privées** : elles ne sont pas routées sur Internet, et le réseau en face ne saurait pas comment répondre aux postes. Le NAT remplace l'adresse source d'un poste par l'adresse WAN de R1 (`192.168.0.8`), puis fait l'opération inverse sur les réponses.

Avec `overload` (PAT), tous les postes partagent la même adresse WAN. Chaque connexion est distinguée par son numéro de port.

### 2.2 Configuration

```
R1(config)#interface fa0/0
R1(config-if)#ip nat inside
R1(config-if)#exit
R1(config)#interface fa0/1
R1(config-if)#ip nat outside
R1(config-if)#exit
R1(config)#access-list 1 remark LAN autorise a sortir sur Internet
R1(config)#access-list 1 permit 192.168.10.0 0.0.0.255
R1(config)#ip nat inside source list 1 interface fa0/1 overload
R1(config)#end
R1#write memory
```

| Commande | Rôle |
|---|---|
| `ip nat inside` | Fa0/0 est le côté interne (LAN) |
| `ip nat outside` | Fa0/1 est le côté externe (Internet) |
| `access-list 1 permit 192.168.10.0 0.0.0.255` | Les adresses qui ont le droit d'être traduites : tout le LAN |
| `ip nat inside source list 1 interface fa0/1 overload` | Traduit ces adresses vers l'adresse de Fa0/1, en partageant cette adresse |

```
R1#show running-config | include nat|access-list
 ip nat inside
 ip nat outside
ip nat inside source list 1 interface FastEthernet0/1 overload
access-list 1 remark LAN autorise a sortir sur Internet
access-list 1 permit 192.168.10.0 0.0.0.255
```

📸 *Capture 06 — Configuration NAT de R1*

![Capture 06 — Configuration NAT de R1](https://hackmd.io/_uploads/BkCu3B5iMx.png)

### 2.3 Vérification : la table de traductions

Après des requêtes de PC1 vers Internet :

```
R1#show ip nat translations
Pro Inside global      Inside local       Outside local       Outside global
udp 192.168.0.8:17163  192.168.10.11:17163 8.8.8.8:53         8.8.8.8:53
icmp 192.168.0.8:43675 192.168.10.11:43675 8.8.8.8:43675      8.8.8.8:43675
icmp 192.168.0.8:48027 192.168.10.11:48027 74.125.29.100:48027 74.125.29.100:48027
```

| Colonne | Valeur | Signification |
|---|---|---|
| Inside local | `192.168.10.11` | Adresse réelle de PC1 |
| Inside global | `192.168.0.8` | Adresse sous laquelle PC1 sort |
| Outside | `8.8.8.8:53` | Serveur DNS de Google, port 53 |

La première ligne est la requête DNS de PC1 (UDP 53), traduite vers l'adresse WAN de R1 ✅.

📸 *Capture 07 — Table NAT : PC1 (192.168.10.11) traduit en 192.168.0.8*

![Capture 07 — Table NAT : PC1 (192.168.10.11) traduit en 192.168.0.8](https://hackmd.io/_uploads/HyCOnH5iGe.png)

---

## 3. Définir des règles de pare-feu basiques (bloquer un port/service donné)

### 3.1 Politique choisie

| Règle | Action | Protocole / port | Source | Destination | Raison |
|---|---|---|---|---|---|
| 10 | **Refuser** | TCP 80 (HTTP) | LAN `192.168.10.0/24` | Toute | Le web non chiffré expose les données (mots de passe, formulaires). Les postes doivent utiliser HTTPS. |
| 30 | **Autoriser** | Tout IP | LAN `192.168.10.0/24` | Toute | Le reste de l'activité (HTTPS, DNS, mail…) doit continuer. |
| implicite | Refuser | Tout | Toute autre source | Toute | Fin automatique de toute ACL Cisco : un paquet venant du LAN avec une adresse source étrangère est rejeté. |

### 3.2 Configuration

ACL étendue nommée, appliquée **en entrée** de l'interface LAN, pour filtrer le trafic dès qu'il arrive sur R1 :

```
R1(config)#ip access-list extended PAREFEU-LAN
R1(config-ext-nacl)#remark Bloque le web non chiffre HTTP vers Internet
R1(config-ext-nacl)#10 deny tcp 192.168.10.0 0.0.0.255 any eq 80
R1(config-ext-nacl)#30 permit ip 192.168.10.0 0.0.0.255 any
R1(config-ext-nacl)#exit
R1(config)#interface fa0/0
R1(config-if)#ip access-group PAREFEU-LAN in
R1(config-if)#end
R1#write memory
```

| Élément | Explication |
|---|---|
| `extended` | ACL étendue : elle filtre selon la source, la destination, le protocole et le port. Une ACL standard ne regarde que la source. |
| `deny tcp … any eq 80` | Refuse le TCP à destination du port 80 |
| `permit ip … any` | Autorise tout le reste venant du LAN |
| Ordre 10 puis 30 | R1 lit l'ACL de haut en bas et s'arrête à la première ligne qui correspond. Le refus doit donc passer avant l'autorisation générale. |
| `ip access-group … in` sur Fa0/0 | Filtre le trafic qui **entre** dans R1 depuis le LAN, avant le routage et le NAT |

```
R1#show access-lists
Standard IP access list 1
    10 permit 192.168.10.0, wildcard bits 0.0.0.255
Extended IP access list PAREFEU-LAN
    10 deny tcp 192.168.10.0 0.0.0.255 any eq www
    30 permit ip 192.168.10.0 0.0.0.255 any
```

IOS affiche `www` à la place du port 80.

📸 *Capture 08 — ACL PAREFEU-LAN : deny tcp eq www, puis permit ip*

![Capture 08 — ACL PAREFEU-LAN : deny tcp eq www, puis permit ip](https://hackmd.io/_uploads/BJeAO2B5jMx.png)

---

## 4. Tester l'accès Internet depuis un poste du LAN

Les postes VPCS ne savent pas ouvrir une page web. Le test d'un service se fait avec un « ping TCP » : `ping <adresse> -P 6 -p <port>` ouvre une connexion TCP sur le port indiqué (Connect), envoie des données (SendData), puis la ferme (Close).

### 4.1 Résolution DNS

```
PC1> ip dns 8.8.8.8
PC1> save
PC1> ping google.com -c 3
google.com resolved to 74.125.29.100
```

Le nom est résolu : la requête DNS (UDP 53) est sortie par le NAT et la réponse est revenue ✅.

📸 *Capture 09 — PC1 : google.com résolu en 74.125.29.100*

![Capture 09 — PC1 : google.com résolu en 74.125.29.100](https://hackmd.io/_uploads/BklAO3S9sMx.png)

### 4.2 Accès HTTPS et HTTP, avant la règle de blocage

```
PC1> ping 1.1.1.1 -P 6 -p 443 -c 3
Connect   443@1.1.1.1 seq=1 ttl=127 time=1176.407 ms
SendData  443@1.1.1.1 seq=1 ttl=127 time=46.244 ms
Close     443@1.1.1.1 seq=1 ttl=127 time=76.220 ms
...
PC1> ping 1.1.1.1 -P 6 -p 80 -c 3
Connect   80@1.1.1.1 seq=1 ttl=127 time=1085.339 ms
SendData  80@1.1.1.1 seq=1 ttl=127 time=45.536 ms
Close     80@1.1.1.1 seq=1 ttl=127 time=74.544 ms
...
```

| Test depuis PC1 (avant l'ACL) | Résultat |
|---|---|
| DNS (UDP 53) vers 8.8.8.8 | ✅ nom résolu |
| HTTPS (TCP 443) vers 1.1.1.1 | ✅ Connect / SendData / Close, 3 sur 3 |
| HTTP (TCP 80) vers 1.1.1.1 | ✅ Connect / SendData / Close |

Le `ttl=127` (128 − 1) montre que les réponses passent par un routeur, R1, avant d'arriver à PC1. La première connexion prend environ une seconde, le temps que le NAT de R1 et celui de VMware créent leurs entrées. Les suivantes prennent 45 à 76 ms.

📸 *Capture 10 — PC1 avant la règle : TCP 443 et TCP 80 passent*

![Capture 10 — PC1 avant la règle : TCP 443 et TCP 80 passent](https://hackmd.io/_uploads/HJkYhH9jMl.png)

---

## 5. Tester qu'une règle de blocage fonctionne

### 5.1 Même test, après la règle

```
PC1> ping 1.1.1.1 -P 6 -p 80 -c 3
*192.168.10.1 tcp_seq=1 ttl=255 time=29.707 ms (ICMP type:3, code:13, Communication administratively prohibited)
*192.168.10.1 tcp_seq=3 ttl=255 time=30.993 ms (ICMP type:3, code:13, Communication administratively prohibited)
```

```
PC1> ping 1.1.1.1 -P 6 -p 443 -c 3
Connect   443@1.1.1.1 seq=1 ttl=127 time=1166.799 ms
SendData  443@1.1.1.1 seq=1 ttl=127 time=45.904 ms
Close     443@1.1.1.1 seq=1 ttl=127 time=135.409 ms
...
Close     443@1.1.1.1 seq=3 ttl=127 time=152.610 ms

PC1> ping google.com -c 1
google.com resolved to 172.217.208.138
```

| Test depuis PC1 | Avant l'ACL | Après l'ACL |
|---|---|---|
| HTTP (TCP 80) vers 1.1.1.1 | ✅ connexion établie | ❌ **refusée par R1** |
| HTTPS (TCP 443) vers 1.1.1.1 | ✅ connexion établie | ✅ connexion établie |
| DNS (UDP 53) | ✅ nom résolu | ✅ nom résolu |

**Lecture du blocage :** la réponse ne vient plus de `1.1.1.1` mais de **`192.168.10.1`**, c'est-à-dire de R1. Le message ICMP type 3, code 13, « Communication administratively prohibited », signifie que le paquet a été refusé par un filtre de l'administrateur : l'ACL. Le `ttl=255` confirme que R1 a répondu directement. Comme le HTTPS et le DNS passent toujours, ce n'est pas une panne : seul le port 80 est bloqué.

📸 *Capture 11 — PC1 : TCP 80 accepté avant la règle, puis « administratively prohibited » après*

![Capture 11 — PC1 : TCP 80 accepté avant la règle, puis « administratively prohibited » après](https://hackmd.io/_uploads/ry1KhBcifl.png)

📸 *Capture 12 — PC1 après la règle : TCP 443 et DNS fonctionnent toujours*

![Capture 12 — PC1 après la règle : TCP 443 et DNS fonctionnent toujours](https://hackmd.io/_uploads/B1JY2Hcsfx.png)

### 5.2 Compteurs de l'ACL

```
R1#show access-lists
Standard IP access list 1
    10 permit 192.168.10.0, wildcard bits 0.0.0.255 (67 matches)
Extended IP access list PAREFEU-LAN
    10 deny tcp 192.168.10.0 0.0.0.255 any eq www (6 matches)
    30 permit ip 192.168.10.0 0.0.0.255 any (11 matches)
```

| Ligne | Compteur | Lecture |
|---|---|---|
| `deny … eq www` | 6 matches | Les paquets HTTP de PC1 ont bien été refusés par cette ligne |
| `permit ip …` | 11 matches | Le trafic HTTPS et DNS a été autorisé |
| `access-list 1` (NAT) | 67 matches | Paquets du LAN traduits par le NAT depuis le début |

📸 *Capture 13 — Compteurs : 6 paquets refusés par la règle HTTP, 11 autorisés*

![Capture 13 — Compteurs : 6 paquets refusés par la règle HTTP, 11 autorisés](https://hackmd.io/_uploads/B1Cd2H5sfl.png)

### 5.3 Schéma

📸 *Capture 14 — Schéma annoté : LAN, R1 (NAT et ACL), accès Internet et résultat des tests*

![Capture 14 — Schéma annoté : LAN, R1 (NAT et ACL), accès Internet et résultat des tests](https://hackmd.io/_uploads/HJgthrcifl.png)

📸 *Capture 15 — Maquette GNS3 annotée : adresses, NAT, ACL et résultats des tests notés à côté des appareils*

![Capture 15 — Maquette GNS3 annotée : adresses, NAT, ACL et résultats des tests notés à côté des appareils](https://hackmd.io/_uploads/Hy1Y3H9jzx.png)

Le même schéma en texte :

```
 LAN 192.168.10.0/24              R1 (c3725)                 Internet
 PC1..PC8 .11-.18                                            (VMnet8)
 passerelle .1      --> Fa0/0  [ ACL PAREFEU-LAN in ]
 SW1                    inside  10 deny  tcp LAN any eq 80
                                30 permit ip  LAN any
                                [ NAT overload ]   Fa0/1 --> 192.168.0.2
                                LAN -> 192.168.0.8 outside    (DHCP)

 TCP 443 : autorisé ✅   DNS : autorisé ✅   TCP 80 : bloqué ❌
```

---

## Limites

- **Ping ICMP depuis le LAN :** les pings de PC1 vers Internet restent en `timeout`, alors que R1 lui-même pingue `8.8.8.8` et que les tests TCP et DNS du LAN passent. Le trafic des postes traverse deux NAT à la suite (celui de R1, puis celui de VMware). La réponse ICMP ne revient pas jusqu'à PC1 (incident 4). Les tests d'accès ont donc été faits en TCP et en DNS.
- **WAN simulé :** VMnet8 remplace la box d'un fournisseur d'accès. Dans un vrai bureau, Fa0/1 serait branchée sur le modem ou la box, avec une adresse publique ou une adresse de la box.
- **Pare-feu sans état :** une ACL regarde chaque paquet séparément. Un vrai pare-feu (ou la fonction Zone-Based Firewall de Cisco) suit l'état des connexions et bloque par défaut ce qui arrive d'Internet sans avoir été demandé.

---

## Incidents rencontrés

> **Incident 1 — Pas d'adresse DHCP sur le WAN avec la carte Wi-Fi**
> **Constat :** avec le nuage relié à la carte Wi-Fi du PC, Fa0/1 reste `unassigned`. `show dhcp lease` affiche `Temp IP addr: 0.0.0.0`, `state: 11 Purging`. `ping 8.8.8.8` : 0 sur 5.
> **Cause :** les cartes et les points d'accès Wi-Fi n'acceptent en général que l'adresse MAC du PC. Les trames envoyées par R1 avec sa propre adresse MAC ne passent pas, et la box ne répond pas.
> **Résolution :** nuage relié à l'adaptateur VMware VMnet8 (NAT VMware avec DHCP). Fa0/1 reçoit `192.168.0.8`. ✅

📸 *Capture 16 — show dhcp lease : « Purging », aucune adresse reçue par le Wi-Fi*

![Capture 16 — show dhcp lease : « Purging », aucune adresse reçue par le Wi-Fi](https://hackmd.io/_uploads/S11FhBqjfe.png)

> **Incident 2 — NAT impossible par manque de mémoire**
> **Constat :** à la configuration du NAT, R1 affiche `%SYS-2-MALLOCFAIL: Memory allocation of 65536 bytes failed`, `No memory available` et `NAT: Addr pool init failed`.
> **Cause :** la mémoire vive donnée au c3725 émulé est trop faible pour les tables du NAT.
> **Résolution :** R1 arrêté, RAM augmentée dans **Configure → Memories and disks**, puis redémarré. La configuration NAT, sauvegardée avant l'arrêt, est rechargée et fonctionne. ✅

📸 *Capture 17 — Configuration du NAT : erreurs de mémoire insuffisante*

![Capture 17 — Configuration du NAT : erreurs de mémoire insuffisante](https://hackmd.io/_uploads/rygt3H5ofe.png)

> **Incident 3 — Règle de l'ACL incomplète**
> **Constat :** `show access-lists` affiche `10 deny tcp 192.168.10.0 0.0.0.255 any`, sans `eq www`, et sans la ligne `permit`. Cette ACL, déjà appliquée sur Fa0/0, bloquait tout le TCP du LAN, puis tout le reste à cause du refus implicite.
> **Cause :** la fin de la ligne (`eq 80`) et la ligne suivante n'ont pas été prises en compte par la console.
> **Résolution :** `no 10` pour supprimer la ligne, puis `10 deny tcp 192.168.10.0 0.0.0.255 any eq 80` et `30 permit ip 192.168.10.0 0.0.0.255 any`, en vérifiant chaque ligne à l'écran avant de la valider. ✅

📸 *Capture 18 — ACL incomplète : « deny tcp … any » sans port*

![Capture 18 — ACL incomplète : « deny tcp … any » sans port](https://hackmd.io/_uploads/Hy1t2B9iMg.png)

> **Incident 4 — Pas de réponse aux pings ICMP depuis le LAN**
> **Constat :** `ping 8.8.8.8` depuis PC1 : `timeout`, même avec 3 secondes d'attente (`-w 3000`). La table NAT de R1 montre pourtant les traductions ICMP de PC1.
> **Cause probable :** double NAT (R1 puis VMware). R1 modifie l'identifiant des pings, et la réponse ne revient pas à travers le NAT de VMware. Les autres protocoles (TCP, UDP/DNS) ne sont pas touchés.
> **Résolution :** contourné. Les tests d'accès sont faits en TCP (443, 80) et par la résolution DNS. ⚠️

### Tableau récapitulatif des incidents

| N° | Machine | Incident | Cause | Résolution | État |
|---|---|---|---|---|---|
| 1 | R1 / nuage | Pas d'adresse DHCP sur le WAN | Pont Wi-Fi refusé | Nuage sur VMnet8 | ✅ |
| 2 | R1 | NAT : mémoire insuffisante | RAM du c3725 émulé trop faible | RAM augmentée | ✅ |
| 3 | R1 | ACL incomplète, tout le TCP bloqué | Fin de ligne non prise en compte | Ligne supprimée et retapée | ✅ |
| 4 | PC1 | Ping ICMP vers Internet en timeout | Double NAT (cause probable) | Tests en TCP et DNS | ⚠️ contourné |

---

## Livrables

| Livrable attendu | Où le trouver dans ce document | État |
|---|---|---|
| Le routeur configuré et fonctionnel | Sections 1 et 2 : WAN, LAN, route par défaut, NAT (captures 01 à 07) | ✅ |
| Les règles de pare-feu documentées | Section 3 : tableau des règles, configuration, explication (capture 08) | ✅ |
| Preuves de test (accès autorisé/bloqué) | Sections 4 et 5 : avant/après, compteurs, schéma et maquette annotée (captures 09 à 15) | ✅ |
