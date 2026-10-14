# Projet 1 — Câblage et adressage IP d'un petit bureau

**Cours :** Networking Essentials — Infrastructure informatique et réseau d'une petite entreprise
**Classe :** E1A
**Auteur :** Drilon Nuhija
**Date :** 11 octobre 2026
**Outils :** GNS3 (serveur local), routeur Cisco c3725 (Dynamips), switch Ethernet intégré de GNS3, VPCS 0.6.2

---

## Contexte

Un bureau de 8 postes doit être câblé et adressé en IP statique sur un même sous-réseau. Le réseau est monté dans GNS3 : 8 postes (VPCS), un switch et un routeur qui sert de passerelle.

### Résumé des résultats

| Élément | Résultat |
|---|---|
| Sous-réseau | `192.168.10.0/24`, masque `255.255.255.0` ✅ |
| Passerelle | R1, `192.168.10.1` sur FastEthernet0/0, `up / up` ✅ |
| Postes | PC1 à PC8, `192.168.10.11` à `192.168.10.18`, IP statiques sauvegardées ✅ |
| Câblage | 9 liens en étoile sur SW1 (8 postes + routeur), tous actifs ✅ |
| Tests | 64 tests ping (56 entre postes + 8 vers la passerelle), 128 réponses sur 128 ✅ |
| Table ARP de R1 | Les 8 postes apparaissent avec leur adresse MAC ✅ |

> 📸 Les captures servent d'illustration. Toutes les valeurs utiles sont aussi écrites en texte, dans les tableaux.

---

## 1. Définir le plan d'adressage (sous-réseau, masque, passerelle)

### 1.1 Choix du sous-réseau

| Élément | Valeur |
|---|---|
| Adresse réseau | `192.168.10.0` |
| Masque | `255.255.255.0` (`/24`) |
| Première adresse utilisable | `192.168.10.1` |
| Dernière adresse utilisable | `192.168.10.254` |
| Adresse de broadcast | `192.168.10.255` |
| Nombre d'hôtes possibles | 2⁸ − 2 = **254** |
| Passerelle par défaut | `192.168.10.1` (routeur R1) |

**Pourquoi ce choix :**

- `192.168.0.0/16` est une plage **privée** (RFC 1918) : elle est faite pour les réseaux internes et n'est pas routée sur Internet.
- Le `/24` est simple à lire : les trois premiers octets désignent le réseau, le dernier désigne la machine.
- Il faut ici 10 adresses (8 postes, 1 passerelle, de la marge). Un `/28` (14 hôtes) suffirait, mais il ne laisserait presque pas de place pour ajouter des imprimantes ou de nouveaux postes. Le `/24` garde de la marge sans compliquer le plan.

| Masque | Hôtes utilisables | Suffit pour 8 postes + passerelle ? |
|---|---|---|
| `/29` (255.255.255.248) | 6 | ❌ |
| `/28` (255.255.255.240) | 14 | ✅ mais très peu de marge |
| `/24` (255.255.255.0) | 254 | ✅ retenu |

### 1.2 Répartition des adresses

| Plage | Usage |
|---|---|
| `192.168.10.1` | Passerelle (routeur R1) |
| `192.168.10.2` à `192.168.10.10` | Réservées aux équipements (switch administrable, imprimantes…) |
| `192.168.10.11` à `192.168.10.18` | Postes PC1 à PC8 |
| `192.168.10.19` à `192.168.10.254` | Libres pour l'évolution du bureau |

Règle facile à retenir : PC**n** = `192.168.10.1n`.

### 1.3 Plan d'adressage IP

| Équipement | Interface | Adresse IP | Masque | Passerelle | Adresse MAC | Port de SW1 |
|---|---|---|---|---|---|---|
| R1 | FastEthernet0/0 | `192.168.10.1` | `255.255.255.0` | — | `c201.545c.0000` | e8 |
| SW1 | — | aucune (switch de couche 2) | — | — | — | — |
| PC1 | Ethernet0 | `192.168.10.11` | `255.255.255.0` | `192.168.10.1` | `00:50:79:66:68:00` | e0 |
| PC2 | Ethernet0 | `192.168.10.12` | `255.255.255.0` | `192.168.10.1` | `00:50:79:66:68:01` | e1 |
| PC3 | Ethernet0 | `192.168.10.13` | `255.255.255.0` | `192.168.10.1` | `00:50:79:66:68:02` | e2 |
| PC4 | Ethernet0 | `192.168.10.14` | `255.255.255.0` | `192.168.10.1` | `00:50:79:66:68:03` | e3 |
| PC5 | Ethernet0 | `192.168.10.15` | `255.255.255.0` | `192.168.10.1` | `00:50:79:66:68:04` | e4 |
| PC6 | Ethernet0 | `192.168.10.16` | `255.255.255.0` | `192.168.10.1` | `00:50:79:66:68:05` | e5 |
| PC7 | Ethernet0 | `192.168.10.17` | `255.255.255.0` | `192.168.10.1` | `00:50:79:66:68:06` | e6 |
| PC8 | Ethernet0 | `192.168.10.18` | `255.255.255.0` | `192.168.10.1` | `00:50:79:66:68:07` | e7 |

Les adresses MAC viennent des commandes `show ip` des postes et `show arp` du routeur.

---

## 2. Câbler les postes au switch

### 2.1 Matériel et câbles

| Équipement | Modèle dans GNS3 | Rôle |
|---|---|---|
| R1 | Routeur Cisco c3725 | Passerelle du bureau |
| SW1 | Ethernet switch (intégré à GNS3) | Relie tous les équipements |
| PC1 à PC8 | VPCS | Postes de travail |

| Lien | Câble | Pourquoi |
|---|---|---|
| PC → SW1 (×8) | Câble droit RJ45 | Un PC et un switch sont deux types d'appareils différents |
| R1 → SW1 | Câble droit RJ45 | Un routeur et un switch sont deux types d'appareils différents |

Dans un vrai bureau : câble catégorie 5e ou 6, connecteurs sertis en norme T568B aux deux bouts. Un câble croisé ne servirait qu'entre deux appareils du même type (PC ↔ PC, switch ↔ switch), et la plupart des cartes actuelles s'adaptent seules (Auto-MDIX).

### 2.2 Préparation du switch

Le switch de GNS3 n'a que 8 ports par défaut (0 à 7). Il en faut 9 : 8 postes et le routeur.

1. Clic droit sur SW1 → **Configure**.
2. Port **8**, VLAN **1**, type **access** → **Add** → **OK**.

| Port | VLAN | Type | Branché sur |
|---|---|---|---|
| 0 à 7 | 1 | access | PC1 à PC8 |
| 8 | 1 | access | R1 Fa0/0 |

Tous les ports sont dans le même VLAN : les 9 équipements sont dans le même réseau local.

📸 *Capture 01 — Configuration de SW1 : ports en mode access, VLAN 1*

![Capture 01 — Configuration de SW1 : ports en mode access, VLAN 1](https://hackmd.io/_uploads/BJECA-Yofe.png)

📸 *Capture 02 — Configuration de SW1 : le port 8 ajouté pour le routeur*

![Capture 02 — Configuration de SW1 : le port 8 ajouté pour le routeur](https://hackmd.io/_uploads/B1EAC-YoMe.png)

### 2.3 Câblage

Câblage en **étoile** : chaque équipement a son propre câble vers SW1.

| De | Port | Vers | Port |
|---|---|---|---|
| PC1 | e0 | SW1 | e0 |
| PC2 | e0 | SW1 | e1 |
| PC3 | e0 | SW1 | e2 |
| PC4 | e0 | SW1 | e3 |
| PC5 | e0 | SW1 | e4 |
| PC6 | e0 | SW1 | e5 |
| PC7 | e0 | SW1 | e6 |
| PC8 | e0 | SW1 | e7 |
| R1 | Fa0/0 | SW1 | e8 |

Une fois tous les nœuds démarrés, les 9 liens sont verts aux deux extrémités : chaque câble est actif.

📸 *Capture 03 — Maquette GNS3 démarrée : liens verts et noms des ports affichés*

![Capture 03 — Maquette GNS3 démarrée : liens verts et noms des ports affichés](https://hackmd.io/_uploads/HJBRAbYizg.png)

---

## 3. Configurer les IP statiques sur chaque poste

### 3.1 Passerelle R1

Dans la console de R1 :

```
enable
configure terminal
hostname R1
interface FastEthernet0/0
 description LAN bureau 8 postes
 ip address 192.168.10.1 255.255.255.0
 no shutdown
end
write memory
show ip interface brief
```

`no shutdown` allume l'interface (elle est éteinte par défaut sur un routeur Cisco). `write memory` enregistre la configuration pour qu'elle survive à un redémarrage.

```
R1#show ip interface brief
Interface          IP-Address      OK? Method Status   Protocol
FastEthernet0/0    192.168.10.1    YES manual up       up
```

`up / up` : l'interface est allumée et le lien avec le switch fonctionne ✅.

📸 *Capture 04 — Configuration de l'interface FastEthernet0/0 de R1 et sauvegarde*

![Capture 04 — Configuration de l'interface FastEthernet0/0 de R1 et sauvegarde](https://hackmd.io/_uploads/HkBCAZYiMl.png)

📸 *Capture 05 — R1 : FastEthernet0/0 en 192.168.10.1, up / up*

![Capture 05 — R1 : FastEthernet0/0 en 192.168.10.1, up / up](https://hackmd.io/_uploads/SyH0AbFsGx.png)

### 3.2 Postes PC1 à PC8

Dans la console de chaque poste, la commande `ip` donne l'adresse, le masque et la passerelle. `save` garde la configuration, `show ip` la vérifie.

```
PC1> ip 192.168.10.11 255.255.255.0 192.168.10.1
Checking for duplicate address...
PC1> save
Saving startup configuration to startup.vpc
.  done
PC1> show ip
NAME        : PC1[1]
IP/MASK     : 192.168.10.11/24
GATEWAY     : 192.168.10.1
MAC         : 00:50:79:66:68:00
```

Même commande sur les 7 autres postes, avec leur adresse :

| Poste | Commande | Résultat de `show ip` |
|---|---|---|
| PC1 | `ip 192.168.10.11 255.255.255.0 192.168.10.1` | `192.168.10.11/24`, passerelle `192.168.10.1` ✅ |
| PC2 | `ip 192.168.10.12 255.255.255.0 192.168.10.1` | `192.168.10.12/24`, passerelle `192.168.10.1` ✅ |
| PC3 | `ip 192.168.10.13 255.255.255.0 192.168.10.1` | `192.168.10.13/24`, passerelle `192.168.10.1` ✅ |
| PC4 | `ip 192.168.10.14 255.255.255.0 192.168.10.1` | `192.168.10.14/24`, passerelle `192.168.10.1` ✅ |
| PC5 | `ip 192.168.10.15 255.255.255.0 192.168.10.1` | `192.168.10.15/24`, passerelle `192.168.10.1` ✅ |
| PC6 | `ip 192.168.10.16 255.255.255.0 192.168.10.1` | `192.168.10.16/24`, passerelle `192.168.10.1` ✅ |
| PC7 | `ip 192.168.10.17 255.255.255.0 192.168.10.1` | `192.168.10.17/24`, passerelle `192.168.10.1` ✅ |
| PC8 | `ip 192.168.10.18 255.255.255.0 192.168.10.1` | `192.168.10.18/24`, passerelle `192.168.10.1` ✅ |

La vérification d'adresse en double (« Checking for duplicate address ») n'a signalé aucun conflit.

📸 *Capture 06 — PC1 : 192.168.10.11/24, passerelle 192.168.10.1*

![Capture 06 — PC1 : 192.168.10.11/24, passerelle 192.168.10.1](https://hackmd.io/_uploads/SJBRAZtsfg.png)

📸 *Capture 07 — PC2 : 192.168.10.12/24, passerelle 192.168.10.1*

![Capture 07 — PC2 : 192.168.10.12/24, passerelle 192.168.10.1](https://hackmd.io/_uploads/HkH00WFoGl.png)

📸 *Capture 08 — PC3 : 192.168.10.13/24, passerelle 192.168.10.1*

![Capture 08 — PC3 : 192.168.10.13/24, passerelle 192.168.10.1](https://hackmd.io/_uploads/BkB0RZtsGg.png)

📸 *Capture 09 — PC4 : 192.168.10.14/24, passerelle 192.168.10.1*

![Capture 09 — PC4 : 192.168.10.14/24, passerelle 192.168.10.1](https://hackmd.io/_uploads/ByB0C-tizl.png)

📸 *Capture 10 — PC5 : 192.168.10.15/24, passerelle 192.168.10.1*

![Capture 10 — PC5 : 192.168.10.15/24, passerelle 192.168.10.1](https://hackmd.io/_uploads/BkrAR-FoMx.png)

📸 *Capture 11 — PC6 : 192.168.10.16/24, passerelle 192.168.10.1*

![Capture 11 — PC6 : 192.168.10.16/24, passerelle 192.168.10.1](https://hackmd.io/_uploads/Bkr0AZFjfl.png)

📸 *Capture 12 — PC7 : 192.168.10.17/24, passerelle 192.168.10.1*

![Capture 12 — PC7 : 192.168.10.17/24, passerelle 192.168.10.1](https://hackmd.io/_uploads/H1gHCAWtjMx.png)

📸 *Capture 13 — PC8 : 192.168.10.18/24, passerelle 192.168.10.1*

![Capture 13 — PC8 : 192.168.10.18/24, passerelle 192.168.10.1](https://hackmd.io/_uploads/SJlHARZtozg.png)

---

## 4. Tester la connectivité (ping) entre tous les postes

### 4.1 Méthode

Sur chaque poste : un ping vers la passerelle, puis vers chacun des 7 autres postes. `-c 2` envoie 2 requêtes par adresse.

Commandes tapées sur PC1 (sur les autres postes, même bloc sans leur propre adresse) :

```
ping 192.168.10.1 -c 2
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
84 bytes from 192.168.10.1 icmp_seq=1 ttl=255 time=15.988 ms
84 bytes from 192.168.10.1 icmp_seq=2 ttl=255 time=16.512 ms
84 bytes from 192.168.10.12 icmp_seq=1 ttl=64 time=1.226 ms
84 bytes from 192.168.10.12 icmp_seq=2 ttl=64 time=1.055 ms
84 bytes from 192.168.10.13 icmp_seq=1 ttl=64 time=0.919 ms
84 bytes from 192.168.10.13 icmp_seq=2 ttl=64 time=1.038 ms
84 bytes from 192.168.10.14 icmp_seq=1 ttl=64 time=1.440 ms
84 bytes from 192.168.10.14 icmp_seq=2 ttl=64 time=1.247 ms
84 bytes from 192.168.10.15 icmp_seq=1 ttl=64 time=1.169 ms
84 bytes from 192.168.10.15 icmp_seq=2 ttl=64 time=1.224 ms
84 bytes from 192.168.10.16 icmp_seq=1 ttl=64 time=1.289 ms
84 bytes from 192.168.10.16 icmp_seq=2 ttl=64 time=1.246 ms
84 bytes from 192.168.10.17 icmp_seq=1 ttl=64 time=0.714 ms
84 bytes from 192.168.10.17 icmp_seq=2 ttl=64 time=0.780 ms
84 bytes from 192.168.10.18 icmp_seq=1 ttl=64 time=0.729 ms
84 bytes from 192.168.10.18 icmp_seq=2 ttl=64 time=1.255 ms
```

Chaque ligne `84 bytes from …` est une réponse reçue. Une absence de réponse aurait affiché `… timeout`.

### 4.2 Matrice de connectivité

| Depuis ↓ / vers → | R1 .1 | PC1 .11 | PC2 .12 | PC3 .13 | PC4 .14 | PC5 .15 | PC6 .16 | PC7 .17 | PC8 .18 |
|---|---|---|---|---|---|---|---|---|---|
| **PC1** | ✅ | — | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **PC2** | ✅ | ✅ | — | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **PC3** | ✅ | ✅ | ✅ | — | ✅ | ✅ | ✅ | ✅ | ✅ |
| **PC4** | ✅ | ✅ | ✅ | ✅ | — | ✅ | ✅ | ✅ | ✅ |
| **PC5** | ✅ | ✅ | ✅ | ✅ | ✅ | — | ✅ | ✅ | ✅ |
| **PC6** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | — | ✅ | ✅ |
| **PC7** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | — | ✅ |
| **PC8** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | — |

✅ = 2 réponses sur 2.

| Bilan | Valeur |
|---|---|
| Tests entre postes | 56 (8 postes × 7 destinations) ✅ |
| Tests vers la passerelle | 8 ✅ |
| Réponses reçues | 128 sur 128, 0 % de perte |
| Temps entre postes | 0,648 à 1,650 ms, `ttl=64` |
| Temps vers la passerelle | 15,070 à 16,512 ms, `ttl=255` (une seule valeur plus haute : 62,520 ms, 1er ping de PC6) |

**Lecture des résultats :**

- Les postes se répondent en environ 1 ms : ils sont sur le même réseau local et passent seulement par le switch.
- La passerelle répond plus lentement (environ 16 ms) car R1 est un routeur émulé par Dynamips, plus lent à traiter les requêtes que les postes VPCS.
- Le `ttl` de départ dépend du système : 64 pour les postes, 255 pour le routeur Cisco. Il n'a pas diminué, ce qui confirme qu'aucun routeur n'est traversé entre deux postes.

📸 *Capture 14 — PC1 : la passerelle et les 7 autres postes répondent*

![Capture 14 — PC1 : la passerelle et les 7 autres postes répondent](https://hackmd.io/_uploads/SklHAAWtiMe.png)

📸 *Capture 15 — PC2 : la passerelle et les 7 autres postes répondent*

![Capture 15 — PC2 : la passerelle et les 7 autres postes répondent](https://hackmd.io/_uploads/rkxB00-FjMe.png)

📸 *Capture 16 — PC3 : la passerelle et les 7 autres postes répondent*

![Capture 16 — PC3 : la passerelle et les 7 autres postes répondent](https://hackmd.io/_uploads/H1lB0C-tsGl.png)

📸 *Capture 17 — PC4 : la passerelle et les 7 autres postes répondent*

![Capture 17 — PC4 : la passerelle et les 7 autres postes répondent](https://hackmd.io/_uploads/SJSCC-FjGg.png)

📸 *Capture 18 — PC5 : la passerelle et les 7 autres postes répondent*

![Capture 18 — PC5 : la passerelle et les 7 autres postes répondent](https://hackmd.io/_uploads/H1rRR-toGe.png)

📸 *Capture 19 — PC6 : la passerelle et les 7 autres postes répondent*

![Capture 19 — PC6 : la passerelle et les 7 autres postes répondent](https://hackmd.io/_uploads/r1gBAAWKszg.png)

📸 *Capture 20 — PC7 : la passerelle et les 7 autres postes répondent*

![Capture 20 — PC7 : la passerelle et les 7 autres postes répondent](https://hackmd.io/_uploads/HylrACbKoMe.png)

📸 *Capture 21 — PC8 : la passerelle et les 7 autres postes répondent*

![Capture 21 — PC8 : la passerelle et les 7 autres postes répondent](https://hackmd.io/_uploads/HkSCCWtsGl.png)

### 4.3 Vérification depuis la passerelle

Après les tests, la table ARP de R1 contient les 8 postes. Le routeur connaît donc l'adresse MAC de chacun.

```
R1#show arp
Protocol  Address          Age (min)  Hardware Addr   Type   Interface
Internet  192.168.10.1            -   c201.545c.0000  ARPA   FastEthernet0/0
Internet  192.168.10.11           6   0050.7966.6800  ARPA   FastEthernet0/0
Internet  192.168.10.12           5   0050.7966.6801  ARPA   FastEthernet0/0
Internet  192.168.10.13           5   0050.7966.6802  ARPA   FastEthernet0/0
Internet  192.168.10.14           4   0050.7966.6803  ARPA   FastEthernet0/0
Internet  192.168.10.15           4   0050.7966.6804  ARPA   FastEthernet0/0
Internet  192.168.10.16           3   0050.7966.6805  ARPA   FastEthernet0/0
Internet  192.168.10.17           3   0050.7966.6806  ARPA   FastEthernet0/0
Internet  192.168.10.18           2   0050.7966.6807  ARPA   FastEthernet0/0
```

Les adresses MAC correspondent à celles relevées sur chaque poste avec `show ip` ✅.

📸 *Capture 22 — Table ARP de R1 : les 8 postes et leur adresse MAC*

![Capture 22 — Table ARP de R1 : les 8 postes et leur adresse MAC](https://hackmd.io/_uploads/BJlSRAbKiGl.png)

---

## 5. Documenter le schéma réseau

📸 *Capture 23 — Schéma réseau annoté : R1, SW1, les 8 postes, leurs adresses et les ports utilisés*

![Capture 23 — Schéma réseau annoté : R1, SW1, les 8 postes, leurs adresses et les ports utilisés](https://hackmd.io/_uploads/BylrRC-Kofg.png)

Le même schéma en texte :

```
            [ R1 — passerelle ]
            Fa0/0 192.168.10.1 /24
                    |
               câble droit
                    | e8
            [ SW1 — switch ]
       VLAN 1, ports e0-e8 en access
   e0  e1  e2  e3  e4  e5  e6  e7
   |   |   |   |   |   |   |   |
  PC1 PC2 PC3 PC4 PC5 PC6 PC7 PC8
  .11 .12 .13 .14 .15 .16 .17 .18

Réseau 192.168.10.0/24
Masque 255.255.255.0
Passerelle 192.168.10.1
Broadcast 192.168.10.255
```

| Annotation | Détail |
|---|---|
| Topologie | Étoile, un seul switch |
| Domaine de broadcast | Un seul (VLAN 1) |
| Câbles | 9 câbles droits |
| Adressage | Statique, un seul sous-réseau `/24` |
| Passerelle | R1 Fa0/0, utile pour sortir du bureau (Internet, autres sites) |

---

## Incidents rencontrés

> **Incident 1 — Pas assez de ports sur le switch**
> **Constat :** le switch de GNS3 n'a que 8 ports (0 à 7), alors qu'il faut brancher 8 postes et le routeur.
> **Cause :** configuration par défaut du switch intégré.
> **Résolution :** ajout du port 8 (access, VLAN 1) dans la configuration de SW1, utilisé pour R1. ✅

> **Incident 2 — Commandes ping collées sur une seule ligne**
> **Constat :** en collant le bloc de pings d'un coup dans la console de PC1, les lignes se sont mélangées. La console a répondu `Bad command: "168.10.15 -c 2". Use ? for help.`
> **Cause :** le collage de plusieurs lignes à la fois n'est pas bien transmis à la console.
> **Résolution :** ligne vidée, puis commandes tapées une par une. ✅

### Tableau récapitulatif des incidents

| N° | Machine | Incident | Cause | Résolution | État |
|---|---|---|---|---|---|
| 1 | SW1 | 8 ports pour 9 équipements | Switch par défaut | Ajout du port 8 | ✅ |
| 2 | PC1 | Commandes collées mélangées | Collage multi-lignes | Commandes tapées une par une | ✅ |

---

## Livrables

| Livrable attendu | Où le trouver dans ce document | État |
|---|---|---|
| Le schéma réseau annoté | Section 5 (capture 23 et version texte), capture 03 | ✅ |
| Le plan d'adressage IP | Section 1 (sous-réseau, répartition, tableau complet) | ✅ |
| Preuve de connectivité totale (captures ping) | Section 4 : matrice, captures 14 à 21, table ARP (capture 22) | ✅ |
