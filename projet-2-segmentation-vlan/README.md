# Projet 2 — Segmentation en VLAN d'un réseau d'entreprise

**Cours :** Networking Essentials — Infrastructure informatique et réseau d'une petite entreprise
**Classe :** E1A
**Auteur :** Drilon Nuhija
**Date :** 12 octobre 2026
**Outils :** GNS3 (serveur local), switch Cisco c3725 avec module NM-16ESW (Dynamips), VPCS 0.6.2

---

## Contexte

Une entreprise veut séparer le trafic de son service comptabilité de celui des autres employés, pour des raisons de sécurité. La solution est de créer deux VLAN sur le switch : un pour la comptabilité, un pour les employés.

Le réseau du Projet 1 (8 postes dans `192.168.10.0/24`) est repris dans GNS3. Le switch simple est remplacé par un switch Cisco manageable, configurable en ligne de commande.

### Résumé des résultats

| Élément | Résultat |
|---|---|
| Switch | SW1, Cisco c3725 avec module de switch NM-16ESW (16 ports) ✅ |
| VLAN 10 `COMPTABILITE` | Ports Fa2/0 à Fa2/3 → PC1 à PC4 ✅ |
| VLAN 20 `EMPLOYES` | Ports Fa2/4 à Fa2/7 → PC5 à PC8 ✅ |
| Test avant VLAN | PC1 ↔ PC5 se répondent ✅ |
| Même VLAN après | 24 tests sur 24 réussis ✅ |
| Entre les VLAN après | 32 tests sur 32 bloqués, dans les deux sens ✅ |

> 📸 Les captures servent d'illustration. Toutes les valeurs utiles sont aussi écrites en texte, dans les tableaux.

---

## Principe des VLAN

Un **VLAN** (Virtual LAN) découpe un switch physique en plusieurs switchs « virtuels ». Chaque port est placé dans un VLAN. Le switch ne transmet une trame que vers les ports du **même VLAN** que le port d'entrée, y compris les broadcasts.

Conséquence : deux postes dans des VLAN différents ne peuvent plus se parler, même branchés sur le même switch et même avec des adresses IP du même réseau. Pour qu'ils communiquent, il faudrait un routeur (routage inter-VLAN), ce qui n'est pas voulu ici.

| Sans VLAN | Avec VLAN |
|---|---|
| Un seul domaine de broadcast | Un domaine de broadcast par VLAN |
| Tous les postes se voient | Seuls les postes du même VLAN se voient |
| Un poste employé peut joindre les postes de la comptabilité | Le trafic de la comptabilité est isolé |

---

## 1. Configurer 2 VLAN distincts sur le switch

### 1.1 Plan des VLAN

| VLAN | Nom | Ports | Postes | Adresses |
|---|---|---|---|---|
| 10 | `COMPTABILITE` | Fa2/0 à Fa2/3 | PC1 à PC4 | `192.168.10.11` à `192.168.10.14` |
| 20 | `EMPLOYES` | Fa2/4 à Fa2/7 | PC5 à PC8 | `192.168.10.15` à `192.168.10.18` |
| 1 | `default` | Fa2/8 à Fa2/15 | aucun | — |

**Choix faits :**

- **Numéros 10 et 20** : on évite le VLAN 1, qui est le VLAN par défaut de tous les ports. Des numéros distincts et lisibles facilitent la maintenance.
- **Même sous-réseau IP pour tous les postes** : les adresses du Projet 1 sont gardées (`192.168.10.0/24`, masque `255.255.255.0`). Si un poste de la comptabilité ne joint plus un employé, c'est donc uniquement à cause du VLAN, et pas à cause de l'adressage. Dans un vrai réseau, on donnerait un sous-réseau à chaque VLAN.
- **Pas de routeur** : le routage entre VLAN n'est pas demandé. Sans routeur, aucune communication n'est possible entre les deux VLAN.

### 1.2 Maquette

Le switch intégré de GNS3 ne se configure pas en ligne de commande. Il est remplacé par un **routeur Cisco c3725 équipé du module NM-16ESW**, un module de switch 16 ports qui se configure comme un switch Cisco.

| Équipement | Modèle dans GNS3 | Ports utilisés |
|---|---|---|
| SW1 | Cisco c3725 + NM-16ESW, flash de 32 Mo | Fa2/0 à Fa2/7 |
| PC1 à PC8 | VPCS | Ethernet0 |

| Poste | Port de SW1 | Câble |
|---|---|---|
| PC1 | Fa2/0 | droit |
| PC2 | Fa2/1 | droit |
| PC3 | Fa2/2 | droit |
| PC4 | Fa2/3 | droit |
| PC5 | Fa2/4 | droit |
| PC6 | Fa2/5 | droit |
| PC7 | Fa2/6 | droit |
| PC8 | Fa2/7 | droit |

📸 *Capture 01 — Maquette GNS3 : SW1 (c3725) et les 8 postes sur les ports Fa2/0 à Fa2/7*

![Capture 01 — Maquette GNS3 : SW1 (c3725) et les 8 postes sur les ports Fa2/0 à Fa2/7](https://hackmd.io/_uploads/HyjhIZ5ofl.png)

### 1.3 Création des VLAN

Le switch enregistre ses VLAN dans un fichier `vlan.dat` sur sa mémoire flash. Dans GNS3, il faut donc lui donner une flash (incident 1) :

1. SW1 arrêté : clic droit → **Configure** → onglet **Memories and disks** → **PCMCIA disk0** : `32` Mo.
2. SW1 redémarré, formatage de la flash :

```
SW1#format flash:
Format operation may take a while. Continue? [confirm]
Format operation will destroy all data in "flash:".  Continue? [confirm]
...
Format: Operation completed successfully.
Format of flash: complete
```

3. Création des deux VLAN en mode `vlan database` :

```
SW1#vlan database
SW1(vlan)#vlan 10 name COMPTABILITE
VLAN 10 added:
    Name: COMPTABILITE
SW1(vlan)#vlan 20 name EMPLOYES
VLAN 20 added:
    Name: EMPLOYES
SW1(vlan)#apply
APPLY completed.
SW1(vlan)#exit
APPLY completed.
Exiting....
```

`apply` enregistre les VLAN dans `vlan.dat`. Sur ce module, les VLAN se créent dans `vlan database`. Sur un switch Catalyst récent, on utiliserait `vlan 10` puis `name COMPTABILITE` en mode `configure terminal`.

📸 *Capture 02 — Formatage de la flash puis création des VLAN 10 et 20 : « APPLY completed »*

![Capture 02 — Formatage de la flash puis création des VLAN 10 et 20 : « APPLY completed »](https://hackmd.io/_uploads/S1o2UZqjzg.png)

---

## 2. Assigner des ports à chaque VLAN

### 2.1 Configuration

```
SW1#configure terminal
SW1(config)#interface range FastEthernet2/0 - 3
SW1(config-if-range)#switchport mode access
SW1(config-if-range)#switchport access vlan 10
SW1(config-if-range)#no shutdown
SW1(config-if-range)#exit
SW1(config)#interface range FastEthernet2/4 - 7
SW1(config-if-range)#switchport mode access
SW1(config-if-range)#switchport access vlan 20
SW1(config-if-range)#no shutdown
SW1(config-if-range)#end
SW1#write memory
Building configuration...
[OK]
```

| Commande | Rôle |
|---|---|
| `interface range FastEthernet2/0 - 3` | Configure les 4 ports d'un coup |
| `switchport mode access` | Le port appartient à un seul VLAN : c'est le mode d'un port de poste de travail |
| `switchport access vlan 10` | Place le port dans le VLAN 10 |
| `no shutdown` | Active le port |
| `write memory` | Enregistre la configuration |

### 2.2 Vérification

```
SW1#show vlan-switch brief

VLAN Name                             Status    Ports
---- -------------------------------- --------- -------------------------------
1    default                          active    Fa2/8, Fa2/9, Fa2/10, Fa2/11
                                                Fa2/12, Fa2/13, Fa2/14, Fa2/15
10   COMPTABILITE                     active    Fa2/0, Fa2/1, Fa2/2, Fa2/3
20   EMPLOYES                         active    Fa2/4, Fa2/5, Fa2/6, Fa2/7
1002 fddi-default                     act/unsup
1003 token-ring-default               act/unsup
1004 fddinet-default                  act/unsup
1005 trnet-default                    act/unsup
```

Les VLAN 1002 à 1005 existent par défaut sur tous les switchs Cisco, pour d'anciennes technologies (FDDI, Token Ring). Ils ne sont pas utilisés.

📸 *Capture 03 — Configuration des ports et « show vlan-switch brief » : VLAN 10 sur Fa2/0-3, VLAN 20 sur Fa2/4-7*

![Capture 03 — Configuration des ports et « show vlan-switch brief » : VLAN 10 sur Fa2/0-3, VLAN 20 sur Fa2/4-7](https://hackmd.io/_uploads/Skon8-coMl.png)

Détail d'un port de chaque VLAN :

| Champ de `show interfaces … switchport` | Fa2/0 | Fa2/4 |
|---|---|---|
| Administrative Mode | static access | static access |
| Operational Mode | static access | static access |
| Access Mode VLAN | `10 (COMPTABILITE)` | `20 (EMPLOYES)` |
| Negotiation of Trunking | Disabled | Disabled |

📸 *Capture 04 — Port Fa2/0 : static access, VLAN 10 (COMPTABILITE)*

![Capture 04 — Port Fa2/0 : static access, VLAN 10 (COMPTABILITE)](https://hackmd.io/_uploads/rJn3Ib9jfe.png)

📸 *Capture 05 — Port Fa2/4 : static access, VLAN 20 (EMPLOYES)*

![Capture 05 — Port Fa2/4 : static access, VLAN 20 (EMPLOYES)](https://hackmd.io/_uploads/Bk2nLbqszx.png)

---

## 3. Tester qu'un poste d'un VLAN ne joint pas un poste de l'autre VLAN

### 3.1 Test témoin avant les VLAN

Avant la configuration, tous les ports étaient dans le VLAN 1. PC1 (futur VLAN 10) et PC5 (futur VLAN 20) se joignaient :

```
PC1> ping 192.168.10.15 -c 2
84 bytes from 192.168.10.15 icmp_seq=1 ttl=64 time=0.998 ms
84 bytes from 192.168.10.15 icmp_seq=2 ttl=64 time=0.815 ms
```

```
PC5> ping 192.168.10.11 -c 2
84 bytes from 192.168.10.11 icmp_seq=1 ttl=64 time=1.138 ms
84 bytes from 192.168.10.11 icmp_seq=2 ttl=64 time=0.980 ms
```

Ce test prouve que le câblage et l'adressage fonctionnent. Un échec après la configuration ne pourra donc venir que des VLAN.

📸 *Capture 06 — Avant les VLAN : PC1 joint PC5*

![Capture 06 — Avant les VLAN : PC1 joint PC5](https://hackmd.io/_uploads/BJ3nLZqjMg.png)

📸 *Capture 07 — Avant les VLAN : PC5 joint PC1*

![Capture 07 — Avant les VLAN : PC5 joint PC1](https://hackmd.io/_uploads/B1nn8Wqoze.png)

### 3.2 Tests après les VLAN

Sur chaque poste : un ping vers les 3 autres postes de son VLAN, puis vers les 4 postes de l'autre VLAN.

Commandes tapées sur PC1 :

```
ping 192.168.10.12 -c 2
ping 192.168.10.13 -c 2
ping 192.168.10.14 -c 2
ping 192.168.10.15 -c 2
ping 192.168.10.16 -c 2
ping 192.168.10.17 -c 2
ping 192.168.10.18 -c 2
```

Résultat sur PC1 :

```
84 bytes from 192.168.10.12 icmp_seq=1 ttl=64 time=1.603 ms
84 bytes from 192.168.10.12 icmp_seq=2 ttl=64 time=1.214 ms
84 bytes from 192.168.10.13 icmp_seq=1 ttl=64 time=1.548 ms
84 bytes from 192.168.10.13 icmp_seq=2 ttl=64 time=1.719 ms
84 bytes from 192.168.10.14 icmp_seq=1 ttl=64 time=1.170 ms
84 bytes from 192.168.10.14 icmp_seq=2 ttl=64 time=1.581 ms

host (192.168.10.15) not reachable
host (192.168.10.16) not reachable
host (192.168.10.17) not reachable
host (192.168.10.18) not reachable
```

Résultat sur PC5 (dans l'autre sens) :

```
84 bytes from 192.168.10.16 icmp_seq=1 ttl=64 time=1.472 ms
84 bytes from 192.168.10.16 icmp_seq=2 ttl=64 time=1.536 ms
84 bytes from 192.168.10.17 icmp_seq=1 ttl=64 time=1.403 ms
84 bytes from 192.168.10.17 icmp_seq=2 ttl=64 time=1.207 ms
84 bytes from 192.168.10.18 icmp_seq=1 ttl=64 time=1.334 ms
84 bytes from 192.168.10.18 icmp_seq=2 ttl=64 time=1.145 ms

host (192.168.10.11) not reachable
host (192.168.10.12) not reachable
host (192.168.10.13) not reachable
host (192.168.10.14) not reachable
```

Même méthode sur les 6 autres postes.

### 3.3 Matrice d'étanchéité

| Depuis ↓ / vers → | PC1 | PC2 | PC3 | PC4 | PC5 | PC6 | PC7 | PC8 |
|---|---|---|---|---|---|---|---|---|
| **PC1** (VLAN 10) | — | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| **PC2** (VLAN 10) | ✅ | — | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| **PC3** (VLAN 10) | ✅ | ✅ | — | ✅ | ❌ | ❌ | ❌ | ❌ |
| **PC4** (VLAN 10) | ✅ | ✅ | ✅ | — | ❌ | ❌ | ❌ | ❌ |
| **PC5** (VLAN 20) | ❌ | ❌ | ❌ | ❌ | — | ✅ | ✅ | ✅ |
| **PC6** (VLAN 20) | ❌ | ❌ | ❌ | ❌ | ✅ | — | ✅ | ✅ |
| **PC7** (VLAN 20) | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ | — | ✅ |
| **PC8** (VLAN 20) | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ | — |

✅ = 2 réponses sur 2. ❌ = `host (…) not reachable`.

| Bilan | Valeur |
|---|---|
| Tests dans le même VLAN | 24 sur 24 réussis, 48 réponses, 0,894 à 3,058 ms ✅ |
| Tests entre VLAN 10 et VLAN 20 | 32 sur 32 bloqués, dans les deux sens ✅ |
| PC1 ↔ PC5 | Avant : ✅ réponse. Après : ❌ not reachable |

**Pourquoi « not reachable » et pas « timeout » :** pour envoyer un ping, PC1 doit d'abord connaître l'adresse MAC de PC5. Il envoie une requête ARP en broadcast (« qui a 192.168.10.15 ? »). Le switch diffuse ce broadcast seulement dans le VLAN 10 : PC5 ne le reçoit jamais et ne répond pas. Sans adresse MAC, PC1 ne peut même pas envoyer le ping, d'où `not reachable`.

📸 *Capture 08 — PC1 (VLAN 10) : PC2, PC3, PC4 répondent, PC5 à PC8 not reachable*

![Capture 08 — PC1 (VLAN 10) : PC2, PC3, PC4 répondent, PC5 à PC8 not reachable](https://hackmd.io/_uploads/H1x23UW5sfg.png)

📸 *Capture 09 — PC2 (VLAN 10) : PC1, PC3, PC4 répondent, PC5 à PC8 not reachable*

![Capture 09 — PC2 (VLAN 10) : PC1, PC3, PC4 répondent, PC5 à PC8 not reachable](https://hackmd.io/_uploads/H132L-csGe.png)

📸 *Capture 10 — PC3 (VLAN 10) : PC1, PC2, PC4 répondent, PC5 à PC8 not reachable*

![Capture 10 — PC3 (VLAN 10) : PC1, PC2, PC4 répondent, PC5 à PC8 not reachable](https://hackmd.io/_uploads/r133LZ9jfl.png)

📸 *Capture 11 — PC4 (VLAN 10) : PC1, PC2, PC3 répondent, PC5 à PC8 not reachable*

![Capture 11 — PC4 (VLAN 10) : PC1, PC2, PC3 répondent, PC5 à PC8 not reachable](https://hackmd.io/_uploads/ry3n8Z5ofx.png)

📸 *Capture 12 — PC5 (VLAN 20) : PC6, PC7, PC8 répondent, PC1 à PC4 not reachable*

![Capture 12 — PC5 (VLAN 20) : PC6, PC7, PC8 répondent, PC1 à PC4 not reachable](https://hackmd.io/_uploads/H12hUZcoze.png)

📸 *Capture 13 — PC6 (VLAN 20) : PC5, PC7, PC8 répondent, PC1 à PC4 not reachable*

![Capture 13 — PC6 (VLAN 20) : PC5, PC7, PC8 répondent, PC1 à PC4 not reachable](https://hackmd.io/_uploads/B1W33L-5sGe.png)

📸 *Capture 14 — PC7 (VLAN 20) : PC5, PC6, PC8 répondent, PC1 à PC4 not reachable*

![Capture 14 — PC7 (VLAN 20) : PC5, PC6, PC8 répondent, PC1 à PC4 not reachable](https://hackmd.io/_uploads/HJlh38-5sMg.png)

📸 *Capture 15 — PC8 (VLAN 20) : PC5, PC6, PC7 répondent, PC1 à PC4 not reachable*

![Capture 15 — PC8 (VLAN 20) : PC5, PC6, PC7 répondent, PC1 à PC4 not reachable](https://hackmd.io/_uploads/rJ2hU-cjMg.png)

---

## 4. Documenter la configuration VLAN

### 4.1 Schéma

📸 *Capture 16 — Schéma annoté : SW1, les deux VLAN, leurs ports et le résultat des tests*

![Capture 16 — Schéma annoté : SW1, les deux VLAN, leurs ports et le résultat des tests](https://hackmd.io/_uploads/SyT3L-cszg.png)

Le même schéma en texte :

```
                 [ SW1 — c3725 + NM-16ESW ]
                  |                      |
     VLAN 10 COMPTABILITE        VLAN 20 EMPLOYES
     Fa2/0 Fa2/1 Fa2/2 Fa2/3     Fa2/4 Fa2/5 Fa2/6 Fa2/7
       |     |     |     |         |     |     |     |
      PC1   PC2   PC3   PC4       PC5   PC6   PC7   PC8
      .11   .12   .13   .14       .15   .16   .17   .18

   Même VLAN : ✅ communication
   VLAN 10 <-> VLAN 20 : ❌ bloqué (pas de routeur)
   Réseau IP commun : 192.168.10.0/24
```

### 4.2 Tableau de configuration

| Port | Mode | VLAN | Nom du VLAN | Poste | Adresse IP | MAC du poste |
|---|---|---|---|---|---|---|
| Fa2/0 | access | 10 | COMPTABILITE | PC1 | `192.168.10.11/24` | `00:50:79:66:68:00` |
| Fa2/1 | access | 10 | COMPTABILITE | PC2 | `192.168.10.12/24` | `00:50:79:66:68:01` |
| Fa2/2 | access | 10 | COMPTABILITE | PC3 | `192.168.10.13/24` | `00:50:79:66:68:02` |
| Fa2/3 | access | 10 | COMPTABILITE | PC4 | `192.168.10.14/24` | `00:50:79:66:68:03` |
| Fa2/4 | access | 20 | EMPLOYES | PC5 | `192.168.10.15/24` | `00:50:79:66:68:04` |
| Fa2/5 | access | 20 | EMPLOYES | PC6 | `192.168.10.16/24` | `00:50:79:66:68:05` |
| Fa2/6 | access | 20 | EMPLOYES | PC7 | `192.168.10.17/24` | `00:50:79:66:68:06` |
| Fa2/7 | access | 20 | EMPLOYES | PC8 | `192.168.10.18/24` | `00:50:79:66:68:07` |
| Fa2/8 à Fa2/15 | — | 1 | default | aucun | — | — |

### 4.3 Configuration complète du switch

```
! Création des VLAN (mode vlan database)
vlan database
 vlan 10 name COMPTABILITE
 vlan 20 name EMPLOYES
 apply
 exit
! Attribution des ports
configure terminal
interface range FastEthernet2/0 - 3
 switchport mode access
 switchport access vlan 10
 no shutdown
interface range FastEthernet2/4 - 7
 switchport mode access
 switchport access vlan 20
 no shutdown
end
write memory
```

### 4.4 Commandes de vérification

| Commande | Ce qu'elle montre |
|---|---|
| `show vlan-switch brief` | La liste des VLAN et les ports de chacun |
| `show interfaces FastEthernet2/0 switchport` | Le mode et le VLAN d'un port précis |
| `ping` depuis les postes | La communication dans un VLAN et le blocage entre VLAN |

### 4.5 Limites

- **Un seul sous-réseau IP** : choisi pour prouver l'effet des VLAN. En production, chaque VLAN aurait son propre sous-réseau (par exemple `192.168.10.0/24` pour la comptabilité et `192.168.20.0/24` pour les employés).
- **Pas d'accès commun** : sans routeur, la comptabilité ne peut plus joindre ni un serveur partagé, ni Internet. Il faudrait un routeur ou un switch de niveau 3 avec des règles de filtrage (ACL) pour n'autoriser que les flux nécessaires.
- **Switch émulé** : le module NM-16ESW est émulé par GNS3. Une commande de vérification (`show mac-address-table`) n'affiche pas les adresses apprises (incident 3).

---

## Incidents rencontrés

> **Incident 1 — Création des VLAN impossible : pas de mémoire flash**
> **Constat :** dans `vlan database`, `apply` répond `% not enough space on flash to store vlan database`, `(No device available)`, puis `Error on database apply 40: NV storage failure`. `exit` est refusé, il faut sortir avec `abort`, et les VLAN ne sont pas créés.
> **Cause :** le switch enregistre ses VLAN dans `vlan.dat` sur sa flash, et le c3725 émulé dans GNS3 n'avait pas de flash.
> **Résolution :** ajout d'un disque PCMCIA disk0 de 32 Mo dans la configuration du nœud SW1, puis `format flash:`. `apply` répond ensuite `APPLY completed.` ✅

📸 *Capture 17 — vlan database : « NV storage failure », la flash est absente*

![Capture 17 — vlan database : « NV storage failure », la flash est absente](https://hackmd.io/_uploads/Bkp2IWcjfg.png)

> **Incident 2 — Flash présente mais non formatée**
> **Constat :** après l'ajout du disque, le premier essai de formatage a échoué (`format flash :`, avec un espace : `% Invalid input detected`). `apply` répond alors `(Invalid DOS media or no media in slot)`.
> **Cause :** le disque existe mais n'est pas formaté. La commande correcte s'écrit sans espace : `format flash:`.
> **Résolution :** `abort`, puis `format flash:` correctement tapé. Formatage terminé, 33 538 048 octets. ✅

📸 *Capture 18 — Disque non formaté : « Invalid DOS media or no media in slot »*

![Capture 18 — Disque non formaté : « Invalid DOS media or no media in slot »](https://hackmd.io/_uploads/Bk6nUbqifx.png)

> **Incident 3 — Table des adresses MAC vide**
> **Constat :** `show mac-address-table` n'affiche que l'adresse du switch lui-même (`Self`, VLAN 1), même juste après des pings réussis entre postes.
> **Cause :** limite de l'émulation du module NM-16ESW dans GNS3 : il transmet les trames, mais n'affiche pas les adresses qu'il apprend.
> **Résolution :** l'appartenance des ports aux VLAN est prouvée par `show vlan-switch brief` et `show interfaces … switchport` (section 2.2). ✅ (contourné)

📸 *Capture 19 — show mac-address-table : seule l'entrée « Self » apparaît*

![Capture 19 — show mac-address-table : seule l'entrée « Self » apparaît](https://hackmd.io/_uploads/H1GhhLbcizl.png)

### Tableau récapitulatif des incidents

| N° | Machine | Incident | Cause | Résolution | État |
|---|---|---|---|---|---|
| 1 | SW1 | `NV storage failure` à la création des VLAN | Pas de flash sur le c3725 émulé | Disque PCMCIA de 32 Mo | ✅ |
| 2 | SW1 | `Invalid DOS media` | Flash non formatée, commande mal tapée | `format flash:` | ✅ |
| 3 | SW1 | Table MAC vide | Limite de l'émulation | Preuve par `show vlan-switch` et `switchport` | ✅ contourné |

---

## Livrables

| Livrable attendu | Où le trouver dans ce document | État |
|---|---|---|
| La configuration VLAN opérationnelle | Sections 1 et 2 (création, attribution des ports, vérifications), captures 02 à 05 | ✅ |
| Preuve d'étanchéité entre VLAN | Section 3 : test avant/après, matrice, captures 06 à 15 | ✅ |
| Documentation de la configuration | Section 4 : schéma (capture 16), tableau des ports, configuration complète | ✅ |
