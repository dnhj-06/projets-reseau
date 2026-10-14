# Projet 5 — Conception réseau complète pour une PME multi-sites

**Cours :** Networking Essentials — Infrastructure informatique et réseau d'une petite entreprise
**Classe :** E1A
**Auteur :** Drilon Nuhija
**Date :** 13 octobre 2026
**Outils :** GNS3 (serveur local), routeurs Cisco c3725 (IOS 12.4(15)T14 adventerprisek9), switchs Ethernet GNS3, VPCS 0.6.2

---

## Contexte

La PME possède deux sites distants, Genève (siège) et Lausanne (antenne). Les deux sites doivent communiquer de façon sécurisée, surtout pour que Lausanne puisse utiliser les serveurs du siège. Ce document présente l'architecture cible (besoins, plan d'adressage, interconnexion, schéma) et la justifie. L'architecture a été montée dans une maquette GNS3 pour tester les deux méthodes d'interconnexion demandées : la ligne dédiée, puis le VPN IPsec site-à-site.

### Résumé

| Élément | Choix | Résultat dans la maquette |
|---|---|---|
| Sites | Genève (~20 postes + serveurs), Lausanne (~10 postes) | 2 routeurs, 2 switchs, 5 postes de test |
| Adressage | `10.<site>.<VLAN>.0/24` : Genève `10.1.0.0/16`, Lausanne `10.2.0.0/16` | Toutes les interfaces `up / up` ✅ |
| VLAN | 10 Admin, 20 Employés, 30 Serveurs (Genève), 99 Gestion | Routage entre VLAN par le routeur de chaque site ✅ |
| Ligne dédiée | Liaison série `10.255.0.0/30` + routes statiques | Lausanne joint le serveur de Genève ✅ |
| VPN IPsec | Tunnel site-à-site par Internet, AES-256 | Sans VPN : échec ❌ · avec VPN : tunnel `QM_IDLE`, paquets chiffrés ✅ |
| Méthode retenue | **VPN IPsec**, ligne dédiée en option | — |

> 📸 Les captures servent d'illustration. Toutes les valeurs utiles sont aussi écrites en texte, dans les tableaux et les blocs de sortie.

---

## 1. Recueillir les besoins (nombre de postes par site, services partagés)

L'entreprise reprend la PME des projets de virtualisation : elle a déjà trois serveurs virtualisés (fichiers, impression, applicatif) au siège. Elle ouvre une antenne à Lausanne.

### 1.1 Postes et utilisateurs

| Site | Rôle | Postes | Groupes d'utilisateurs |
|---|---|---|---|
| Genève | Siège | ~20 | Direction et comptabilité (~5), employés (~15) |
| Lausanne | Antenne | ~10 | Responsable d'antenne (~2), employés (~8) |

### 1.2 Services

| Service | Où | Qui l'utilise |
|---|---|---|
| Serveur de fichiers | Genève (VLAN Serveurs) | Les deux sites |
| Serveur d'impression | Genève (VLAN Serveurs) | Les deux sites |
| Serveur applicatif | Genève (VLAN Serveurs) | Les deux sites |
| Imprimante réseau | Une par site | Le site local |
| Accès Internet | Un accès FAI par site | Chaque site sort par son propre accès |

### 1.3 Contraintes

| Contrainte | Conséquence sur la conception |
|---|---|
| Le trafic entre les sites contient des données internes (fichiers, compta) | Il doit être **chiffré** s'il passe par Internet |
| PME, budget limité | Utiliser au maximum l'accès Internet déjà payé |
| Séparer direction, employés et serveurs | Un **VLAN** par groupe |
| L'entreprise peut grandir (nouveaux postes, nouveau site) | Plan d'adressage avec de la marge et facile à étendre |

---

## 2. Concevoir le plan d'adressage global

### 2.1 Logique du plan

Toutes les adresses internes suivent la forme **`10.<site>.<VLAN>.0/24`** :

- le **2e octet** donne le site : `1` = Genève, `2` = Lausanne (un 3e site serait `10.3.0.0/16`) ;
- le **3e octet** donne le VLAN : `10` = Admin, `20` = Employés, `30` = Serveurs, `99` = Gestion ;
- la **passerelle** est toujours `.1`.

En lisant `10.2.20.15`, on sait tout de suite qu'il s'agit d'un employé de Lausanne.

### 2.2 Réseaux internes

| Site | VLAN | Nom | Réseau | Passerelle | Hôtes possibles | Besoin |
|---|---|---|---|---|---|---|
| Genève | 10 | Admin | `10.1.10.0/24` | `10.1.10.1` | 254 | ~5 |
| Genève | 20 | Employés | `10.1.20.0/24` | `10.1.20.1` | 254 | ~15 + imprimante |
| Genève | 30 | Serveurs | `10.1.30.0/24` | `10.1.30.1` | 254 | 3 serveurs |
| Genève | 99 | Gestion | `10.1.99.0/24` | `10.1.99.1` | 254 | switch |
| Lausanne | 10 | Admin | `10.2.10.0/24` | `10.2.10.1` | 254 | ~2 |
| Lausanne | 20 | Employés | `10.2.20.0/24` | `10.2.20.1` | 254 | ~8 + imprimante |
| Lausanne | 99 | Gestion | `10.2.99.0/24` | `10.2.99.1` | 254 | switch |

### 2.3 Répartition dans chaque /24

| Plage | Usage |
|---|---|
| `.1` | Passerelle (sous-interface du routeur) |
| `.2` à `.49` | Équipements à adresse fixe : serveurs, imprimantes, switch |
| `.50` à `.199` | Postes |
| `.200` à `.254` | Réserve |

Adresses fixes prévues :

| Équipement | Adresse |
|---|---|
| Serveur de fichiers | `10.1.30.11` |
| Serveur d'impression | `10.1.30.12` |
| Serveur applicatif | `10.1.30.13` |
| Imprimante Genève | `10.1.20.20` |
| Imprimante Lausanne | `10.2.20.20` |
| Switch SW-GE / SW-LS (gestion) | `10.1.99.2` / `10.2.99.2` |

### 2.4 Liaisons entre les sites

| Liaison | Réseau | Côté Genève | Côté Lausanne / FAI |
|---|---|---|---|
| Ligne dédiée (série) | `10.255.0.0/30` | R-GE S1/0 `10.255.0.1` | R-LS S1/0 `10.255.0.2` |
| WAN Genève ↔ FAI | `203.0.113.0/30` | R-GE Fa0/1 `203.0.113.2` | FAI Fa0/0 `203.0.113.1` |
| WAN Lausanne ↔ FAI | `198.51.100.0/30` | — | R-LS Fa0/1 `198.51.100.2` · FAI Fa0/1 `198.51.100.1` |

Un **/30** donne exactement 2 adresses utilisables, une par extrémité d'une liaison point à point : aucune adresse n'est perdue. Les adresses publiques `203.0.113.0` et `198.51.100.0` sont des plages réservées à la documentation. Dans la réalité, ce sont les adresses fournies par le FAI de chaque site.

### 2.5 Adresses utilisées dans la maquette

La maquette garde exactement le même plan, avec un poste par VLAN :

| Poste | VLAN | Adresse | Passerelle | Port du switch |
|---|---|---|---|---|
| GE-ADM | 10 | `10.1.10.10/24` | `10.1.10.1` | SW-GE port 1 |
| GE-EMP | 20 | `10.1.20.10/24` | `10.1.20.1` | SW-GE port 2 |
| GE-SRV | 30 | `10.1.30.10/24` | `10.1.30.1` | SW-GE port 3 |
| LS-ADM | 10 | `10.2.10.10/24` | `10.2.10.1` | SW-LS port 1 |
| LS-EMP | 20 | `10.2.20.10/24` | `10.2.20.1` | SW-LS port 2 |

📸 *Capture 01 — Maquette GNS3 câblée : FAI, R-GE et R-LS, SW-GE et SW-LS, 5 postes*

![Capture 01 — Maquette GNS3 câblée : FAI, R-GE et R-LS, SW-GE et SW-LS, 5 postes](https://hackmd.io/_uploads/S1Ww_QhjMg.png)

### 2.6 VLAN sur les switchs

| Switch | Port | Type | VLAN | Branché à |
|---|---|---|---|---|
| SW-GE | 0 | dot1q (trunk) | 1 natif + 10, 20, 30 tagués | R-GE Fa0/0 |
| SW-GE | 1 | access | 10 | GE-ADM |
| SW-GE | 2 | access | 20 | GE-EMP |
| SW-GE | 3 | access | 30 | GE-SRV |
| SW-LS | 0 | dot1q (trunk) | 1 natif + 10, 20 tagués | R-LS Fa0/0 |
| SW-LS | 1 | access | 10 | LS-ADM |
| SW-LS | 2 | access | 20 | LS-EMP |

📸 *Capture 02 — SW-GE : port 0 en dot1q, ports 1, 2 et 3 en VLAN 10, 20 et 30*

![Capture 02 — SW-GE : port 0 en dot1q, ports 1, 2 et 3 en VLAN 10, 20 et 30](https://hackmd.io/_uploads/HkWPum3jzx.png)

📸 *Capture 03 — SW-LS : port 0 en dot1q, ports 1 et 2 en VLAN 10 et 20*

![Capture 03 — SW-LS : port 0 en dot1q, ports 1 et 2 en VLAN 10 et 20](https://hackmd.io/_uploads/r1ZwO72ofg.png)

### 2.7 Configuration des routeurs

Chaque routeur a une **sous-interface par VLAN** sur Fa0/0. C'est elle qui sert de passerelle `.1` au VLAN (router-on-a-stick).

**R-GE :**

```
interface fa0/0
 no shutdown
interface fa0/0.10
 encapsulation dot1Q 10
 ip address 10.1.10.1 255.255.255.0
interface fa0/0.20
 encapsulation dot1Q 20
 ip address 10.1.20.1 255.255.255.0
interface fa0/0.30
 encapsulation dot1Q 30
 ip address 10.1.30.1 255.255.255.0
interface s1/0
 ip address 10.255.0.1 255.255.255.252
 no shutdown
interface fa0/1
 ip address 203.0.113.2 255.255.255.252
 no shutdown
```

**R-LS :** même principe, avec `10.2.10.1`, `10.2.20.1`, S1/0 `10.255.0.2` et Fa0/1 `198.51.100.2`.

**FAI :** Fa0/0 `203.0.113.1/30` et Fa0/1 `198.51.100.1/30`. Il ne reçoit aucune route vers les réseaux `10.x` : comme un vrai fournisseur d'accès, il ne transporte que les adresses publiques.

| Routeur | Interface | Adresse | État |
|---|---|---|---|
| R-GE | Fa0/0.10 / .20 / .30 | `10.1.10.1` / `10.1.20.1` / `10.1.30.1` | up / up ✅ |
| R-GE | Fa0/1 | `203.0.113.2` | up / up ✅ |
| R-GE | Serial1/0 | `10.255.0.1` | up / down au moment de la capture (R-LS pas encore configuré) |
| R-LS | Fa0/0.10 / .20 | `10.2.10.1` / `10.2.20.1` | up / up ✅ |
| R-LS | Fa0/1 | `198.51.100.2` | up / up ✅ |
| R-LS | Serial1/0 | `10.255.0.2` | up / up ✅ (ligne montée) |
| FAI | Fa0/0 / Fa0/1 | `203.0.113.1` / `198.51.100.1` | up / up ✅ |

📸 *Capture 04 — R-GE : show ip interface brief*

![Capture 04 — R-GE : show ip interface brief](https://hackmd.io/_uploads/rJebDuXhjGx.png)

📸 *Capture 05 — R-LS : show ip interface brief, Serial1/0 up / up*

![Capture 05 — R-LS : show ip interface brief, Serial1/0 up / up](https://hackmd.io/_uploads/ryWDdX3jMx.png)

📸 *Capture 06 — FAI : show ip interface brief, Fa0/0 et Fa0/1 up / up*

![Capture 06 — FAI : show ip interface brief, Fa0/0 et Fa0/1 up / up](https://hackmd.io/_uploads/Ske-Pu7hsMl.png)

### 2.8 Configuration des postes

```
GE-ADM> ip 10.1.10.10 255.255.255.0 10.1.10.1
GE-EMP> ip 10.1.20.10 255.255.255.0 10.1.20.1
GE-SRV> ip 10.1.30.10 255.255.255.0 10.1.30.1
LS-ADM> ip 10.2.10.10 255.255.255.0 10.2.10.1
LS-EMP> ip 10.2.20.10 255.255.255.0 10.2.20.1
```

📸 *Capture 07 — GE-ADM : 10.1.10.10/24, passerelle 10.1.10.1*

![Capture 07 — GE-ADM : 10.1.10.10/24, passerelle 10.1.10.1](https://hackmd.io/_uploads/ryW-POQ2oMg.png)

📸 *Capture 08 — GE-EMP : 10.1.20.10/24, passerelle 10.1.20.1*

![Capture 08 — GE-EMP : 10.1.20.10/24, passerelle 10.1.20.1](https://hackmd.io/_uploads/H1lWDuX2jfg.png)

📸 *Capture 09 — GE-SRV : 10.1.30.10/24, passerelle 10.1.30.1*

![Capture 09 — GE-SRV : 10.1.30.10/24, passerelle 10.1.30.1](https://hackmd.io/_uploads/HJxZvuQ2ize.png)

📸 *Capture 10 — LS-ADM : 10.2.10.10/24, passerelle 10.2.10.1*

![Capture 10 — LS-ADM : 10.2.10.10/24, passerelle 10.2.10.1](https://hackmd.io/_uploads/B1l-POQhife.png)

📸 *Capture 11 — LS-EMP : 10.2.20.10/24, passerelle 10.2.20.1*

![Capture 11 — LS-EMP : 10.2.20.10/24, passerelle 10.2.20.1](https://hackmd.io/_uploads/SygZwdm2oMe.png)

### 2.9 Test du plan à l'intérieur de chaque site

```
GE-ADM> ping 10.1.10.1
84 bytes from 10.1.10.1 icmp_seq=1 ttl=255 time=15.663 ms
...
84 bytes from 10.1.10.1 icmp_seq=5 ttl=255 time=15.941 ms

GE-ADM> ping 10.1.30.10
10.1.30.10 icmp_seq=1 timeout
84 bytes from 10.1.30.10 icmp_seq=2 ttl=63 time=31.263 ms
...
84 bytes from 10.1.30.10 icmp_seq=5 ttl=63 time=31.006 ms
```

| Test | Résultat | Ce que ça prouve |
|---|---|---|
| GE-ADM → passerelle `10.1.10.1` | 5/5 ✅ | VLAN 10, trunk et sous-interface .10 fonctionnent |
| GE-ADM (VLAN 10) → GE-SRV `10.1.30.10` (VLAN 30) | 4/5 ✅, `ttl=63` | R-GE route entre les VLAN (le TTL a perdu 1 en traversant le routeur) |
| LS-ADM (VLAN 10) → LS-EMP `10.2.20.10` (VLAN 20) | 4/5 ✅, `ttl=63` | R-LS route entre les VLAN |

Le premier `timeout` correspond au temps de la résolution ARP.

📸 *Capture 12 — GE-ADM : ping de la passerelle et du serveur (VLAN 30)*

![Capture 12 — GE-ADM : ping de la passerelle et du serveur (VLAN 30)](https://hackmd.io/_uploads/B1g-DOQ3sfx.png)

📸 *Capture 13 — LS-ADM : ping de LS-EMP (VLAN 20)*

![Capture 13 — LS-ADM : ping de LS-EMP (VLAN 20)](https://hackmd.io/_uploads/S1WWvuQhoMx.png)

---

## 3. Prévoir la méthode d'interconnexion des sites (VPN site-à-site, ligne dédiée simulée)

### 3.1 Les deux méthodes comparées

| Critère | Ligne dédiée (louée à un opérateur) | VPN IPsec site-à-site sur Internet |
|---|---|---|
| Principe | Liaison privée point à point entre les deux sites | Tunnel chiffré entre les deux routeurs, à travers Internet |
| Coût | Abonnement spécifique, en plus de l'accès Internet | Utilise l'accès Internet déjà payé, aucune ligne en plus |
| Sécurité | Privée chez l'opérateur, mais **non chiffrée** par défaut | **Chiffrée** (AES-256) et authentifiée (clé partagée, SHA) |
| Débit et délai | Garantis par contrat | Dépendent de la qualité des accès Internet |
| Mise en place | Commande chez l'opérateur, délai d'installation | Configuration sur les deux routeurs |
| Ajout d'un site | Une nouvelle ligne à commander | Un nouveau tunnel à configurer |

**Choix : VPN IPsec site-à-site.** Pour une PME de 30 postes, le VPN donne la sécurité demandée sans payer une ligne de plus. La ligne dédiée reste une option si un jour l'entreprise a besoin d'un débit garanti, par exemple pour de la téléphonie sur IP entre les sites.

Les deux méthodes ont été testées dans la maquette.

### 3.2 Ligne dédiée simulée

La ligne dédiée est une liaison **série** directe entre R-GE et R-LS (`10.255.0.0/30`). Chaque routeur reçoit une route statique vers les réseaux de l'autre site :

```
R-GE(config)# ip route 10.2.0.0 255.255.0.0 10.255.0.2
R-LS(config)# ip route 10.1.0.0 255.255.0.0 10.255.0.1
```

Une seule route par site suffit, grâce au plan d'adressage : `10.2.0.0/16` regroupe tous les VLAN de Lausanne.

📸 *Capture 14 — R-GE : route statique vers 10.2.0.0/16 par la ligne dédiée*

![Capture 14 — R-GE : route statique vers 10.2.0.0/16 par la ligne dédiée](https://hackmd.io/_uploads/BkZwOQhizg.png)

📸 *Capture 15 — R-LS : route statique vers 10.1.0.0/16 par la ligne dédiée*

![Capture 15 — R-LS : route statique vers 10.1.0.0/16 par la ligne dédiée](https://hackmd.io/_uploads/HJg-wuX3ize.png)

**Test de LS-ADM (Lausanne) vers le serveur GE-SRV (Genève) :**

```
LS-ADM> ping 10.1.30.10
10.1.30.10 icmp_seq=1 timeout
10.1.30.10 icmp_seq=2 timeout
84 bytes from 10.1.30.10 icmp_seq=3 ttl=62 time=61.096 ms
84 bytes from 10.1.30.10 icmp_seq=4 ttl=62 time=61.475 ms
84 bytes from 10.1.30.10 icmp_seq=5 ttl=62 time=63.653 ms

LS-ADM> trace 10.1.30.10
 1   10.2.10.1    16.229 ms  15.737 ms  15.942 ms
 2   10.255.0.1   46.330 ms  46.244 ms  45.416 ms
 3   *10.1.30.10  60.846 ms (ICMP type:3, code:3, Destination port unreachable)
```

| Saut | Adresse | Appareil |
|---|---|---|
| 1 | `10.2.10.1` | R-LS (passerelle de LS-ADM) |
| 2 | `10.255.0.1` | R-GE, atteint **par la ligne série** |
| 3 | `10.1.30.10` | GE-SRV. « Port unreachable » est la réponse normale de la destination au `trace` |

Lausanne joint le serveur de Genève par la ligne dédiée ✅ Le `ttl=62` montre que le paquet a traversé 2 routeurs. Les 2 premiers `timeout` correspondent aux résolutions ARP sur les deux routeurs.

📸 *Capture 16 — LS-ADM : ping et trace vers le serveur de Genève par la ligne dédiée*

![Capture 16 — LS-ADM : ping et trace vers le serveur de Genève par la ligne dédiée](https://hackmd.io/_uploads/BJeDuQ3izg.png)

```
R-GE# show ip route
     203.0.113.0/30 is subnetted, 1 subnets
C       203.0.113.0 is directly connected, FastEthernet0/1
     10.0.0.0/8 is variably subnetted, 5 subnets, 3 masks
C       10.1.10.0/24 is directly connected, FastEthernet0/0.10
S       10.2.0.0/16 [1/0] via 10.255.0.2
C       10.1.30.0/24 is directly connected, FastEthernet0/0.30
C       10.1.20.0/24 is directly connected, FastEthernet0/0.20
C       10.255.0.0/30 is directly connected, Serial1/0
```

📸 *Capture 17 — R-GE : table de routage avec la route statique vers Lausanne*

![Capture 17 — R-GE : table de routage avec la route statique vers Lausanne](https://hackmd.io/_uploads/H1ewdm3jzx.png)

### 3.3 Passage par Internet : sans VPN, ça ne marche pas

La ligne dédiée est coupée, et chaque routeur envoie tout ce qu'il ne connaît pas vers le FAI :

```
R-GE(config)# interface s1/0
R-GE(config-if)# shutdown
R-GE(config)# ip route 0.0.0.0 0.0.0.0 203.0.113.1

R-LS(config)# interface s1/0
R-LS(config-if)# shutdown
R-LS(config)# ip route 0.0.0.0 0.0.0.0 198.51.100.1
```

| Routeur | Serial1/0 | Route par défaut |
|---|---|---|
| R-GE | administratively down | `S* 0.0.0.0/0 [1/0] via 203.0.113.1` |
| R-LS | administratively down | `S* 0.0.0.0/0 [1/0] via 198.51.100.1` |

La route vers `10.2.0.0/16` par la ligne disparaît de la table de R-GE dès que la ligne est coupée.

📸 *Capture 18 — R-GE : ligne coupée et route par défaut vers le FAI*

![Capture 18 — R-GE : ligne coupée et route par défaut vers le FAI](https://hackmd.io/_uploads/r1WP_XhjGx.png)

📸 *Capture 19 — R-LS : ligne coupée et route par défaut vers le FAI*

![Capture 19 — R-LS : ligne coupée et route par défaut vers le FAI](https://hackmd.io/_uploads/HJWwdm2sfl.png)

| Test | Résultat |
|---|---|
| LS-ADM → GE-SRV `10.1.30.10` | 5/5 `timeout` ❌ |
| R-LS → R-GE par Internet (`203.0.113.2`) | `.!!!!` 4/5 (80 %) ✅ |

Les adresses publiques des deux sites se joignent, mais pas les réseaux internes. Le FAI ne connaît pas les adresses privées `10.x` et ne les transporte pas. Pour relier les réseaux internes par Internet, il faut un tunnel : c'est le rôle du VPN.

📸 *Capture 20 — LS-ADM : ping du serveur de Genève sans VPN, 5 timeouts*

![Capture 20 — LS-ADM : ping du serveur de Genève sans VPN, 5 timeouts](https://hackmd.io/_uploads/H1-DOm3iGl.png)

📸 *Capture 21 — R-LS : ping de l'adresse publique de R-GE, 4/5*

![Capture 21 — R-LS : ping de l'adresse publique de R-GE, 4/5](https://hackmd.io/_uploads/rJZwdQ3jfl.png)

### 3.4 Configuration du VPN IPsec

| Élément | Valeur | Rôle |
|---|---|---|
| Phase 1 (ISAKMP), politique 10 | AES 256, SHA, clé partagée, Diffie-Hellman groupe 5, durée 86 400 s | Les deux routeurs s'authentifient et créent un canal sûr pour négocier |
| Clé partagée | Identique sur les deux routeurs, liée à l'adresse publique du pair | Authentification des routeurs |
| Phase 2 (transform set `TS-PME`) | `esp-aes 256` + `esp-sha-hmac` | Chiffrement et contrôle d'intégrité des données |
| Trafic protégé (ACL 110) | `10.1.0.0/16 ↔ 10.2.0.0/16` | Seul le trafic entre les deux sites entre dans le tunnel |
| Crypto map `VPN-PME` | Pair, transform set, ACL | Rassemble tout, appliquée sur Fa0/1 (côté Internet) |

**R-GE :**

```
crypto isakmp policy 10
 encryption aes 256
 hash sha
 authentication pre-share
 group 5
 lifetime 86400
crypto isakmp key MaCleVPN2026 address 198.51.100.2
crypto ipsec transform-set TS-PME esp-aes 256 esp-sha-hmac
access-list 110 permit ip 10.1.0.0 0.0.255.255 10.2.0.0 0.0.255.255
crypto map VPN-PME 10 ipsec-isakmp
 set peer 198.51.100.2
 set transform-set TS-PME
 match address 110
interface fa0/1
 crypto map VPN-PME
```

**R-LS :** même configuration en miroir. Le pair devient `203.0.113.2`, et l'ACL 110 `permit ip 10.2.0.0 0.0.255.255 10.1.0.0 0.0.255.255`.

Après l'application de la crypto map, les deux routeurs affichent `%CRYPTO-6-ISAKMP_ON_OFF: ISAKMP is ON`.

📸 *Capture 22 — R-GE : configuration du VPN IPsec*

![Capture 22 — R-GE : configuration du VPN IPsec](https://hackmd.io/_uploads/Sk-vuXnsfe.png)

📸 *Capture 23 — R-LS : configuration du VPN IPsec*

![Capture 23 — R-LS : configuration du VPN IPsec](https://hackmd.io/_uploads/HkZDum2szl.png)

```
R-LS# show access-lists 110
Extended IP access list 110
    10 permit ip 10.2.0.0 0.0.255.255 10.1.0.0 0.0.255.255

R-LS# show crypto map
Crypto Map "VPN-PME" 10 ipsec-isakmp
        Peer = 203.0.113.2
        Extended IP access list 110
            access-list 110 permit ip 10.2.0.0 0.0.255.255 10.1.0.0 0.0.255.255
        Current peer: 203.0.113.2
        Security association lifetime: 4608000 kilobytes/3600 seconds
        PFS (Y/N): N
        Transform sets={
                TS-PME,
        }
        Interfaces using crypto map VPN-PME:
                FastEthernet0/1
```

📸 *Capture 24 — R-LS : ACL 110 et crypto map VPN-PME sur FastEthernet0/1*

![Capture 24 — R-LS : ACL 110 et crypto map VPN-PME sur FastEthernet0/1](https://hackmd.io/_uploads/SJZPOm3jzg.png)

### 3.5 Vérification du VPN

Même test qu'en 3.3, avec la ligne dédiée toujours coupée :

```
LS-ADM> ping 10.1.30.10
10.1.30.10 icmp_seq=1 timeout
10.1.30.10 icmp_seq=2 timeout
10.1.30.10 icmp_seq=3 timeout
84 bytes from 10.1.30.10 icmp_seq=4 ttl=62 time=91.105 ms
84 bytes from 10.1.30.10 icmp_seq=5 ttl=62 time=93.701 ms
```

Les premiers paquets se perdent pendant la négociation du tunnel. Dès qu'il est monté, le serveur répond ✅

📸 *Capture 25 — LS-ADM : le serveur de Genève répond à travers le VPN*

![Capture 25 — LS-ADM : le serveur de Genève répond à travers le VPN](https://hackmd.io/_uploads/SygZPd73sGl.png)

```
R-LS# show crypto isakmp sa
dst             src             state          conn-id slot status
203.0.113.2     198.51.100.2    QM_IDLE           1001    0 ACTIVE

R-LS# show crypto ipsec sa | include pkts
    #pkts encaps: 4, #pkts encrypt: 4, #pkts digest: 4
    #pkts decaps: 4, #pkts decrypt: 4, #pkts verify: 4
    #pkts compressed: 0, #pkts decompressed: 0
    #pkts not compressed: 0, #pkts compr. failed: 0
    #pkts not decompressed: 0, #pkts decompress failed: 0
```

| Vérification | Résultat | Signification |
|---|---|---|
| `show crypto isakmp sa` | `QM_IDLE`, `ACTIVE` entre `198.51.100.2` et `203.0.113.2` | La phase 1 est établie, le tunnel est monté ✅ |
| `#pkts encaps / encrypt` | 4 | Paquets envoyés vers Genève, chiffrés ✅ |
| `#pkts decaps / decrypt / verify` | 4 | Paquets reçus de Genève, déchiffrés et vérifiés ✅ |
| Erreurs | 0 | Aucun échec ✅ |

Les compteurs indiquent 4 paquets chiffrés et 4 déchiffrés, alors que 2 réponses seulement sont visibles sur LS-ADM. Une partie des paquets a été chiffrée pendant que le tunnel finissait de se monter.

📸 *Capture 26 — R-LS : tunnel ISAKMP en QM_IDLE et compteurs IPsec*

![Capture 26 — R-LS : tunnel ISAKMP en QM_IDLE et compteurs IPsec](https://hackmd.io/_uploads/Hk-vd72iGg.png)

| Situation | LS-ADM → serveur de Genève |
|---|---|
| Ligne dédiée active | ✅ par `10.255.0.1` |
| Ligne coupée, sans VPN | ❌ 5/5 timeout |
| Ligne coupée, avec VPN | ✅ à travers le tunnel chiffré |

---

## 4. Réaliser un schéma d'architecture complet

### 4.1 Architecture cible

📸 *Capture 27 — Schéma de l'architecture cible : deux sites, VLAN, adressage, tunnel VPN et ligne dédiée en option*

![Capture 27 — Schéma de l'architecture cible : deux sites, VLAN, adressage, tunnel VPN et ligne dédiée en option](https://hackmd.io/_uploads/SJxbwuXhjfe.png)

Le même schéma en texte :

```
        GENÈVE (siège) 10.1.0.0/16                 LAUSANNE (antenne) 10.2.0.0/16
  VLAN10 Admin   10.1.10.0/24 ~5 postes      VLAN10 Admin    10.2.10.0/24 ~2 postes
  VLAN20 Employés 10.1.20.0/24 ~15 + impr.   VLAN20 Employés 10.2.20.0/24 ~8 + impr.
  VLAN30 Serveurs 10.1.30.0/24 .11 .12 .13   VLAN99 Gestion  10.2.99.0/24
  VLAN99 Gestion  10.1.99.0/24
           |  ports access                          |  ports access
        SW-GE -- trunk 802.1Q                    SW-LS -- trunk 802.1Q
           |                                        |
        R-GE  Fa0/0.10/.20/.30 = .1              R-LS  Fa0/0.10/.20 = .1
        Fa0/1 203.0.113.2/30                     Fa0/1 198.51.100.2/30
           |                                        |
           +------------ FAI / Internet ------------+
           |     .1                          .1     |
           ====== tunnel VPN IPsec (AES-256) =======
           |  10.1.0.0/16  <---------->  10.2.0.0/16 |
           |                                        |
        S1/0 10.255.0.1 ---- ligne dédiée ---- 10.255.0.2 S1/0
                         (option, /30, série)
```

### 4.2 Maquette GNS3 annotée

📸 *Capture 28 — Maquette GNS3 annotée : adresses, VLAN, liaisons et résultats des tests notés à côté des appareils*

![Capture 28 — Maquette GNS3 annotée : adresses, VLAN, liaisons et résultats des tests notés à côté des appareils](https://hackmd.io/_uploads/SyZ-vdQhiMe.png)

### 4.3 Choix des câbles et des liaisons

| Liaison dans la maquette | Type de câble / port | Équivalent réel | Pourquoi ce choix |
|---|---|---|---|
| Poste ↔ switch (GE-ADM, GE-EMP, GE-SRV, LS-ADM, LS-EMP) | Ethernet | Câble cuivre **droit** RJ45, catégorie 6 | Un poste et un switch sont deux types d'appareils différents : câble droit. Jusqu'à 100 m, suffisant dans un bureau |
| R-GE Fa0/0 ↔ SW-GE port 0, R-LS Fa0/0 ↔ SW-LS port 0 | Ethernet, port en **trunk 802.1Q** | Câble droit RJ45 | Un seul câble transporte tous les VLAN du site, chaque trame étant marquée avec son numéro de VLAN. Le c3725 n'a que 2 ports FastEthernet intégrés : un pour le LAN, un pour Internet. Le trunk évite d'avoir un port de routeur par VLAN |
| R-GE Fa0/1 ↔ FAI, R-LS Fa0/1 ↔ FAI | Ethernet | Câble droit vers le modem ou la box fibre du FAI | C'est la liaison normale entre un routeur d'entreprise et l'équipement de l'opérateur |
| R-GE S1/0 ↔ R-LS S1/0 | **Série** (module NM-4T) | Ligne louée point à point de l'opérateur | Une ligne dédiée est historiquement une liaison WAN série. Le port série la distingue clairement des liaisons Internet : ce trafic ne passe pas par le FAI |

Sur le matériel récent, les ports détectent le type de câble (auto-MDIX). On peut donc utiliser des câbles droits partout en Ethernet. Dans la maquette GNS3, le type de câble est choisi automatiquement selon les ports reliés.

---

## 5. Justifier les choix techniques dans un rapport

| Choix | Justification | Alternative écartée |
|---|---|---|
| Adressage `10.<site>.<VLAN>.0/24` | Lisible (le site et le VLAN se lisent dans l'adresse) ; 254 adresses par VLAN pour ~15 postes au maximum, donc de la marge ; un /16 par site résume tout le site en une seule route et une seule ligne d'ACL | `192.168.x.0/24` : moins de place pour structurer plusieurs sites et VLAN |
| /30 pour les liaisons point à point | Exactement 2 adresses, une par extrémité | /24 : gaspille 252 adresses |
| VLAN Admin, Employés, Serveurs, Gestion | Sépare les groupes. Un problème sur un poste employé (broadcast, virus) reste dans son VLAN ; les serveurs sont isolés ; on peut filtrer entre VLAN sur le routeur | Un seul réseau plat : aucune séparation |
| Router-on-a-stick (sous-interfaces 802.1Q) | Un seul routeur par site fait passerelle pour tous les VLAN avec un seul câble ; suffisant pour 20 postes | Switch de niveau 3 : plus cher, inutile à cette taille |
| Routes statiques | Deux sites seulement : une route par site suffit, rien à maintenir | OSPF : utile à partir de plusieurs sites ou de liens redondants |
| **VPN IPsec site-à-site** | Chiffré et authentifié ; utilise l'accès Internet existant, donc pas de coût de ligne ; testé et validé dans la maquette | Ligne dédiée : plus chère, non chiffrée par défaut ; gardée en option si un débit garanti devient nécessaire |
| AES-256 et SHA pour IPsec | AES-256 est un chiffrement fort et reconnu ; SHA vérifie que les paquets n'ont pas été modifiés | DES / 3DES : anciens et plus faibles |
| ACL 110 limitée à `10.1.0.0/16 ↔ 10.2.0.0/16` | Seul le trafic entre les sites passe dans le tunnel ; le reste (Internet) sort normalement | Tout chiffrer vers l'autre site : inutile, alourdit le lien |
| Serveurs centralisés à Genève | Une seule copie des données, sauvegardes au même endroit ; Lausanne y accède par le VPN | Serveurs dans chaque site : double coût, données dispersées |

### Ce qui serait ajouté en production

- **Accès Internet des postes :** NAT overload sur Fa0/1 de chaque routeur, en excluant le trafic entre les sites du NAT pour qu'il reste dans le tunnel.
- **DHCP :** une plage par VLAN, de `.50` à `.199`, sur le routeur ou sur un serveur.
- **Paramètres VPN plus récents :** IKEv2, SHA-256 et un groupe Diffie-Hellman 14 ou plus, avec une longue clé aléatoire gardée secrète. L'IOS 12.4 du c3725 émulé limite les choix ; les paramètres utilisés suffisent pour la démonstration.
- **Filtrage entre VLAN :** une ACL pour que seuls les postes autorisés joignent le VLAN Serveurs.
- **Ligne dédiée en secours :** si l'entreprise la garde, une route statique flottante peut basculer le trafic sur la ligne quand le VPN tombe.

---

## Limites

- **Besoins supposés :** il n'y a pas de vrai client. Les besoins reprennent la PME des projets précédents, avec un deuxième site.
- **Maquette réduite :** un poste par VLAN, pas de VLAN 99 Gestion, pas d'imprimante ni de vrais serveurs. GE-SRV représente le VLAN Serveurs.
- **Internet simulé :** le FAI est un simple routeur, et les adresses publiques sont des plages de documentation.
- **Pas de NAT ni de DHCP dans la maquette :** ils font partie de l'architecture cible, mais n'étaient pas nécessaires pour tester l'interconnexion.
- **Clé du VPN visible :** dans ce document, la clé partagée apparaît dans les configurations. En production, elle serait longue, aléatoire et jamais publiée.

---

## Incidents rencontrés

> **Incident 1 — Interface du FAI désactivée**
> **Constat :** `show ip interface brief` sur le FAI affiche FastEthernet0/0 `203.0.113.1` en `administratively down`.
> **Cause :** la commande `no shutdown` sous `interface fa0/0` n'a pas été prise en compte.
> **Résolution :** `interface fa0/0` puis `no shutdown`. Fa0/0 passe en `up / up` (capture 06). ✅

📸 *Capture 29 — FAI : FastEthernet0/0 en administratively down*

![Capture 29 — FAI : FastEthernet0/0 en administratively down](https://hackmd.io/_uploads/rkWwd7nifg.png)

> **Incident 2 — Postes injoignables : aucune adresse configurée**
> **Constat :** depuis GE-ADM, `ping 10.1.10.1` et `ping 10.1.30.10` répondent `host (…) not reachable`. Même chose depuis LS-ADM. `show ip` sur GE-ADM affiche `IP/MASK : 0.0.0.0/0` et `GATEWAY : 0.0.0.0`.
> **Cause :** les tests ont été lancés avant de configurer l'adresse des postes.
> **Résolution :** commande `ip` sur les 5 postes (captures 07 à 11). Les pings passent ensuite (captures 12 et 13). ✅

📸 *Capture 30 — GE-ADM : passerelle et serveur « not reachable »*

![Capture 30 — GE-ADM : passerelle et serveur « not reachable »](https://hackmd.io/_uploads/SkWPuQnjGe.png)

📸 *Capture 31 — GE-ADM : show ip, aucune adresse configurée*

![Capture 31 — GE-ADM : show ip, aucune adresse configurée](https://hackmd.io/_uploads/ry-v_72ozg.png)

> **Incident 3 — Commandes refusées par le routeur**
> **Constat :** `ip route 0.0.0.0 0.0.0.0 203.0.113.1` sur R-GE renvoie `% Invalid input detected at '^' marker`. Sur R-LS, d'autres lignes sont refusées de la même façon : une route coupée (`0.0.0.0.`), puis `group 5` et `lifetime 86400` mal saisies.
> **Cause :** sur R-GE, la commande a été tapée en mode privilégié (`R-GE#`) et non en mode configuration (`R-GE(config)#`). Sur R-LS, ce sont des fautes de frappe.
> **Résolution :** `configure terminal`, puis chaque commande retapée en vérifiant l'invite (captures 18, 19 et 23). ✅

📸 *Capture 32 — R-GE : ip route refusée hors du mode configuration*

![Capture 32 — R-GE : ip route refusée hors du mode configuration](https://hackmd.io/_uploads/Hkx-vOm3izl.png)

### Tableau récapitulatif des incidents

| N° | Machine | Incident | Cause | Résolution | État |
|---|---|---|---|---|---|
| 1 | FAI | Fa0/0 administratively down | `no shutdown` non pris | `no shutdown` retapé | ✅ |
| 2 | Postes VPCS | `host not reachable` partout | Adresses IP pas encore configurées | Commande `ip` sur les 5 postes | ✅ |
| 3 | R-GE, R-LS | `% Invalid input` | Mauvais mode (`#` au lieu de `(config)#`), fautes de frappe | Commandes retapées dans le bon mode | ✅ |

---

## Livrables

| Livrable attendu | Où le trouver dans ce document | État |
|---|---|---|
| Un schéma d'architecture réseau complet | Section 4 : schéma de l'architecture cible (capture 27), version texte, maquette annotée (capture 28), choix des câbles | ✅ |
| Un plan d'adressage détaillé | Section 2 : logique, réseaux par site et VLAN, répartition des plages, liaisons, adresses de la maquette (captures 01 à 13) | ✅ |
| Un rapport de justification technique | Sections 1, 3 et 5 : besoins, comparaison ligne dédiée / VPN, tests des deux méthodes (captures 14 à 26), tableau des choix et alternatives | ✅ |
