# Projet 4 — Diagnostic d'une panne réseau

**Cours :** Networking Essentials — Infrastructure informatique et réseau d'une petite entreprise
**Classe :** E1A
**Auteur :** Drilon Nuhija
**Date :** 12 octobre 2026
**Outils :** GNS3 (serveur local), maquette du Projet 3 (routeur Cisco c3725, switch GNS3, VPCS 0.6.2), accès Internet par le réseau NAT VMware (VMnet8)

---

## Contexte

Un utilisateur signale qu'il n'accède plus au réseau. Trois pannes ont été simulées l'une après l'autre sur son poste, PC3, dans la maquette du Projet 3 : un câble coupé, un port de switch dans le mauvais VLAN et une mauvaise passerelle. Pour chaque panne, le diagnostic part de la couche physique et remonte le modèle OSI jusqu'à trouver la cause, puis la panne est corrigée et le poste retesté.

### Résumé des résultats

| Panne | Couche OSI | Symptôme sur PC3 | Cause racine | État |
|---|---|---|---|---|
| A | 1 — Physique | `host (192.168.10.1) not reachable` | Câble PC3–SW1 coupé (lien suspendu) | ✅ Résolue |
| B | 2 — Liaison | `host (192.168.10.1) not reachable` | Port de PC3 (Ethernet2) placé en VLAN 20 | ✅ Résolue |
| C | 3 — Réseau | `host (192.168.10.254) not reachable` | Passerelle erronée `192.168.10.254` au lieu de `.1` | ✅ Résolue |

> 📸 Les captures servent d'illustration. Toutes les valeurs utiles sont aussi écrites en texte, dans les tableaux et les blocs de sortie.

---

## 1. Recevoir le scénario de panne du formateur

### 1.1 Scénario

| Élément | Valeur |
|---|---|
| Signalement | « Je n'accède plus au réseau » |
| Poste concerné | PC3, `192.168.10.13/24`, passerelle `192.168.10.1` |
| Réseau | LAN `192.168.10.0/24`, switch SW1, routeur R1 (NAT + pare-feu), accès Internet par VMnet8 |
| Pannes simulées | A : câble · B : switch (VLAN) · C : adressage IP |
| Qui a créé les pannes | Moi-même, sur la maquette, une panne à la fois |

Le sujet prévoit que le formateur crée la panne. Ici, je l'ai créée moi-même dans GNS3, puis je l'ai diagnostiquée comme si je ne la connaissais pas, en suivant la même démarche pour les trois. Le sujet cite aussi le DNS : PC3 n'a pas de serveur DNS configuré (seul PC1 en a un depuis le Projet 3), donc il n'y avait pas de résolution de noms à casser sur ce poste. La panne DNS n'a pas été faite.

### 1.2 Équivalence des commandes

Les postes de la maquette sont des VPCS, pas des PC Windows. Les commandes demandées ont un équivalent direct :

| Demandé (Windows) | Utilisé sur VPCS | Ce que ça montre |
|---|---|---|
| `ipconfig` | `show ip` | Adresse, masque, passerelle, DNS du poste |
| `ping` | `ping` | Réponse d'un hôte en ICMP |
| `tracert` | `trace` | Chaque routeur traversé jusqu'à la destination |
| Test d'un port TCP | `ping 8.8.8.8 -P 6 -p 443` | Connexion TCP 443 (HTTPS) vers Internet |

Au Projet 3, les pings ICMP des postes vers Internet ne revenaient pas (double NAT, R1 puis VMware). L'accès à Internet est donc testé en TCP 443 : chaque essai affiche `Connect`, `SendData`, `Close` quand la connexion passe.

### 1.3 État de référence, avant toute panne

Avant de casser quoi que ce soit, PC3 fonctionne normalement :

```
PC3> show ip
NAME        : PC3[1]
IP/MASK     : 192.168.10.13/24
GATEWAY     : 192.168.10.1
DNS         :
MAC         : 00:50:79:66:68:02
```

📸 *Capture 01 — Référence : show ip de PC3*

![Capture 01 — Référence : show ip de PC3](https://hackmd.io/_uploads/rkNA9PcjGe.png)

```
PC3> ping 192.168.10.1
192.168.10.1 icmp_seq=1 timeout
84 bytes from 192.168.10.1 icmp_seq=2 ttl=255 time=15.627 ms
84 bytes from 192.168.10.1 icmp_seq=3 ttl=255 time=16.142 ms
```

Le premier `timeout` correspond au temps de la résolution ARP de la passerelle. Les réponses suivantes arrivent.

📸 *Capture 02 — Référence : ping de la passerelle 192.168.10.1*

![Capture 02 — Référence : ping de la passerelle 192.168.10.1](https://hackmd.io/_uploads/rJ4C5v9sMg.png)

```
PC3> ping 8.8.8.8 -P 6 -p 443
Connect   443@8.8.8.8 seq=1 ttl=127 time=1086.289 ms
SendData  443@8.8.8.8 seq=1 ttl=127 time=45.025 ms
Close     443@8.8.8.8 seq=1 ttl=127 time=106.894 ms
...
Close     443@8.8.8.8 seq=5 ttl=127 time=60.368 ms
```

5 connexions TCP 443 sur 5 vers `8.8.8.8` ✅

📸 *Capture 03 — Référence : connexion TCP 443 vers 8.8.8.8*

![Capture 03 — Référence : connexion TCP 443 vers 8.8.8.8](https://hackmd.io/_uploads/S1N09v5jMe.png)

```
PC3> trace 8.8.8.8
trace to 8.8.8.8, 8 hops max, press Ctrl+C to stop
 1   192.168.10.1   15.804 ms  16.404 ms  16.649 ms
 2   *192.168.0.2   29.947 ms  31.185 ms
 3   *  *  *
```

Premier saut : R1 (`192.168.10.1`). Deuxième saut : la passerelle NAT de VMware (`192.168.0.2`). Ensuite les réponses ICMP ne reviennent plus, comme au Projet 3. C'est l'état normal de cette maquette.

📸 *Capture 04 — Référence : trace vers 8.8.8.8*

![Capture 04 — Référence : trace vers 8.8.8.8](https://hackmd.io/_uploads/B1VA9D9sGg.png)

> Dans les sorties VPCS, la ligne `PC1 : 192.168.10.13 …` qui suit une commande `ip` désigne l'instance VPCS numéro 1 (il y en a une par nœud dans GNS3), pas le poste PC1.

---

## 2. Diagnostiquer couche par couche (physique, IP, DNS...)

### 2.1 Méthode

Pour chaque panne, je pars du symptôme, puis je teste du bas vers le haut. Dès qu'une couche est bonne, je passe à la suivante. Dès qu'une couche est fausse, j'ai trouvé où chercher.

| Couche | Question | Test |
|---|---|---|
| 1 — Physique | Le câble est-il branché et actif ? | Lien vert ou suspendu dans la maquette |
| 2 — Liaison | Le poste parle-t-il à son switch et à ses voisins ? | Ping d'un voisin du même réseau (PC1, `.11`), VLAN du port sur SW1 |
| 3 — Réseau | La configuration IP est-elle correcte ? La passerelle répond-elle ? | `show ip`, `ping 192.168.10.1`, `trace 8.8.8.8` |
| 4 à 7 — Transport, application | Le service fonctionne-t-il ? | `ping 8.8.8.8 -P 6 -p 443` |

Dans VPCS, le message `host (X) not reachable` veut dire que le poste n'a reçu aucune réponse ARP de X. Pour une destination hors du réseau local, X est la passerelle. Ce message indique donc à quelle adresse le poste essaie de parler, ce qui oriente le diagnostic.

### 2.2 Panne A

**Symptôme :**

```
PC3> ping 8.8.8.8 -P 6 -p 443
host (192.168.10.1) not reachable
```

PC3 n'arrive même pas à joindre sa passerelle.

📸 *Capture 05 — Panne A : symptôme, la passerelle 192.168.10.1 est injoignable*

![Capture 05 — Panne A : symptôme, la passerelle 192.168.10.1 est injoignable](https://hackmd.io/_uploads/rkNRcv9jfl.png)

**Tests :**

| Couche | Test | Résultat | Conclusion |
|---|---|---|---|
| 3 | `show ip` | Identique à la référence : `192.168.10.13/24`, passerelle `192.168.10.1` | Config IP correcte ✅ |
| 3 | `ping 192.168.10.1` | `host (192.168.10.1) not reachable` | Passerelle injoignable ❌ |
| 2 | `ping 192.168.10.11` (PC1) | `host (192.168.10.11) not reachable` | Même le voisin ne répond pas ❌ |
| 1 | Maquette GNS3 | Symbole pause sur le lien PC3–SW1, extrémités en jaune | Lien coupé ❌ |

La config IP est bonne, mais PC3 ne joint personne, même pas un poste de son propre réseau. Le problème est donc sous la couche 3. Dans la maquette, le lien PC3–SW1 est suspendu.

📸 *Capture 06 — Panne A : show ip correct, passerelle et voisin injoignables*

![Capture 06 — Panne A : show ip correct, passerelle et voisin injoignables](https://hackmd.io/_uploads/SkVCqP5jGl.png)

📸 *Capture 07 — Panne A : maquette, le lien PC3–SW1 est suspendu (symbole pause)*

![Capture 07 — Panne A : maquette, le lien PC3–SW1 est suspendu (symbole pause)](https://hackmd.io/_uploads/SkHA9D9sfe.png)

### 2.3 Panne B

**Symptôme :** le même que pour la panne A.

```
PC3> ping 8.8.8.8 -P 6 -p 443
host (192.168.10.1) not reachable
```

**Tests :**

| Couche | Test | Résultat | Conclusion |
|---|---|---|---|
| 3 | `show ip` | Identique à la référence | Config IP correcte ✅ |
| 3 | `ping 192.168.10.1` | `host (192.168.10.1) not reachable` | Passerelle injoignable ❌ |
| 2 | `ping 192.168.10.11` (PC1) | `host (192.168.10.11) not reachable` | Voisin injoignable ❌ |
| 1 | Maquette GNS3 | Tous les liens verts, aucun symbole pause | Câble correct ✅ |
| 2 | Port de PC3 sur SW1 | `Link from PC3 port Ethernet0 to SW1 port Ethernet2` | PC3 est sur le port 2 |
| 2 | SW1 → Configure | Port 2 : VLAN **20**. Ports 0, 1, 3 à 6 : VLAN 1 | Mauvais VLAN ❌ |

Mêmes symptômes que la panne A, mais cette fois le câble est bon. La couche 1 est donc hors de cause et on regarde le switch. Le port de PC3 n'est pas dans le même VLAN que les autres postes et que la passerelle.

📸 *Capture 08 — Panne B : show ip correct, passerelle et voisin injoignables*

![Capture 08 — Panne B : show ip correct, passerelle et voisin injoignables](https://hackmd.io/_uploads/BySAqwciGx.png)

📸 *Capture 09 — Panne B : maquette, tous les liens sont verts*

![Capture 09 — Panne B : maquette, tous les liens sont verts](https://hackmd.io/_uploads/rJU05wcjze.png)

📸 *Capture 10 — Panne B : PC3 est branché sur le port Ethernet2 de SW1*

![Capture 10 — Panne B : PC3 est branché sur le port Ethernet2 de SW1](https://hackmd.io/_uploads/ByBR5D9jfl.png)

📸 *Capture 11 — Panne B : configuration de SW1, port 2 en VLAN 20*

![Capture 11 — Panne B : configuration de SW1, port 2 en VLAN 20](https://hackmd.io/_uploads/SkrR5DciMl.png)

### 2.4 Panne C

**Symptôme :**

```
PC3> ping 8.8.8.8 -P 6 -p 443
host (192.168.10.254) not reachable
```

Le message ne parle plus de `192.168.10.1` mais de `192.168.10.254`. C'est un premier indice : le poste cherche une autre passerelle.

**Tests :**

| Couche | Test | Résultat | Conclusion |
|---|---|---|---|
| 1 et 2 | `ping 192.168.10.11` (PC1) | 5 réponses sur 5, `ttl=64`, environ 1 ms | Câble et switch corrects ✅ |
| 3 | `ping 192.168.10.1` | 5 réponses sur 5, `ttl=255`, environ 16 ms | R1 fonctionne ✅ |
| 3 | `show ip` | `GATEWAY : 192.168.10.254` | Mauvaise passerelle ❌ |
| 3 | `trace 8.8.8.8` | `host (192.168.10.254) not reachable` | Bloqué dès le premier saut ❌ |

```
PC3> show ip
NAME        : PC3[1]
IP/MASK     : 192.168.10.13/24
GATEWAY     : 192.168.10.254
```

Le poste joint tout son réseau local, y compris R1. Mais pour sortir du réseau, il envoie ses paquets à `192.168.10.254`, une adresse qui n'existe pas. Le trafic local n'utilise pas la passerelle, c'est pour ça qu'il fonctionne encore.

📸 *Capture 12 — Panne C : symptôme, ping du voisin et de la passerelle réussis*

![Capture 12 — Panne C : symptôme, ping du voisin et de la passerelle réussis](https://hackmd.io/_uploads/BkU0qvcifx.png)

📸 *Capture 13 — Panne C : show ip, passerelle 192.168.10.254*

![Capture 13 — Panne C : show ip, passerelle 192.168.10.254](https://hackmd.io/_uploads/BkrAcD9sfx.png)

📸 *Capture 14 — Panne C : trace bloqué, 192.168.10.254 injoignable*

![Capture 14 — Panne C : trace bloqué, 192.168.10.254 injoignable](https://hackmd.io/_uploads/BJSCqwcsMx.png)

---

## 3. Identifier la cause racine

| Panne | Couche | Cause racine | Preuve |
|---|---|---|---|
| A | 1 — Physique | Le câble entre PC3 et le port 2 de SW1 est coupé (lien suspendu dans GNS3) | Capture 07 |
| B | 2 — Liaison | Le port 2 de SW1 (PC3) est en VLAN 20, alors que les autres postes et R1 sont en VLAN 1. Le switch ne transmet les trames de PC3, ARP compris, qu'aux ports du VLAN 20 | Captures 10 et 11 |
| C | 3 — Réseau | La passerelle de PC3 est `192.168.10.254` au lieu de `192.168.10.1`. Aucun appareil n'a cette adresse | Captures 13 et 14 |

Les pannes A et B donnent exactement le même symptôme : `host (192.168.10.1) not reachable`, avec une config IP correcte. Le symptôme seul ne suffit pas, c'est le test de la couche 1 qui les différencie. Pour la panne C, le poste joint son réseau local mais pas l'extérieur : c'est le signe d'un problème de passerelle.

---

## 4. Résoudre la panne

### 4.1 Panne A — remettre le câble

**Action :** clic droit sur le lien PC3–SW1 → **Resume**.

📸 *Capture 15 — Panne A : réactivation du lien (Resume)*

![Capture 15 — Panne A : réactivation du lien (Resume)](https://hackmd.io/_uploads/BkrAcDqife.png)

**Re-test :**

```
PC3> ping 192.168.10.1
84 bytes from 192.168.10.1 icmp_seq=1 ttl=255 time=15.788 ms
...
84 bytes from 192.168.10.1 icmp_seq=5 ttl=255 time=15.522 ms

PC3> ping 8.8.8.8 -P 6 -p 443
Connect   443@8.8.8.8 seq=1 ttl=127 time=1144.191 ms
SendData  443@8.8.8.8 seq=1 ttl=127 time=45.951 ms
Close     443@8.8.8.8 seq=1 ttl=127 time=106.633 ms
...
```

Passerelle : 5 réponses sur 5 ✅ · TCP 443 vers Internet : connexions établies ✅

📸 *Capture 16 — Panne A : re-test réussi, passerelle et TCP 443*

![Capture 16 — Panne A : re-test réussi, passerelle et TCP 443](https://hackmd.io/_uploads/HkS0qvcjGl.png)

### 4.2 Panne B — remettre le port dans le bon VLAN

**Action :** SW1 → **Configure** → Port `2`, VLAN `1`, Type `access` → **Add** → **Apply** → **OK**.

| Port | VLAN avant | VLAN après |
|---|---|---|
| 2 (PC3) | 20 ❌ | 1 ✅ |

📸 *Capture 17 — Panne B : port 2 remis en VLAN 1*

![Capture 17 — Panne B : port 2 remis en VLAN 1](https://hackmd.io/_uploads/HyxBCcP9jze.png)

**Re-test :**

```
PC3> ping 192.168.10.1
84 bytes from 192.168.10.1 icmp_seq=1 ttl=255 time=16.696 ms
...
84 bytes from 192.168.10.1 icmp_seq=5 ttl=255 time=15.886 ms

PC3> ping 8.8.8.8 -P 6 -p 443
Connect   443@8.8.8.8 seq=1 ttl=127 time=1146.234 ms
...
Close     443@8.8.8.8 seq=5 ttl=127 time=46.176 ms
```

Passerelle : 5 réponses sur 5 ✅ · TCP 443 : 5 connexions sur 5 ✅

📸 *Capture 18 — Panne B : re-test réussi, passerelle et TCP 443*

![Capture 18 — Panne B : re-test réussi, passerelle et TCP 443](https://hackmd.io/_uploads/HJUCcwcjze.png)

### 4.3 Panne C — corriger la passerelle

**Action :**

```
PC3> ip 192.168.10.13 255.255.255.0 192.168.10.1
Checking for duplicate address...
PC1 : 192.168.10.13 255.255.255.0 gateway 192.168.10.1

PC3> save
Saving startup configuration to startup.vpc
.  done
```

Le `save` enregistre la correction, pour qu'elle reste après un redémarrage du poste.

📸 *Capture 19 — Panne C : passerelle corrigée et configuration sauvegardée*

![Capture 19 — Panne C : passerelle corrigée et configuration sauvegardée](https://hackmd.io/_uploads/HyBAqwciMx.png)

**Re-test :**

```
PC3> show ip
IP/MASK     : 192.168.10.13/24
GATEWAY     : 192.168.10.1

PC3> trace 8.8.8.8
 1   192.168.10.1   15.982 ms  16.447 ms  15.712 ms
 2   *192.168.0.2   46.494 ms  30.324 ms
 3   *  *  *
```

Le trace repasse par R1 puis par la passerelle VMware, comme dans l'état de référence ✅

📸 *Capture 20 — Panne C : show ip et trace identiques à la référence*

![Capture 20 — Panne C : show ip et trace identiques à la référence](https://hackmd.io/_uploads/r1U0qPqizl.png)

```
PC3> ping 8.8.8.8 -P 6 -p 443
Connect   443@8.8.8.8 seq=1 ttl=127 time=1072.313 ms
...
Close     443@8.8.8.8 seq=5 ttl=127 time=60.283 ms
```

TCP 443 : 5 connexions sur 5 ✅

📸 *Capture 21 — Panne C : re-test réussi, TCP 443 vers Internet*

![Capture 21 — Panne C : re-test réussi, TCP 443 vers Internet](https://hackmd.io/_uploads/rkIAqv9oMg.png)

---

## 5. Rédiger un rapport d'intervention

### 5.1 Fiche d'intervention

| Champ | Panne A | Panne B | Panne C |
|---|---|---|---|
| Poste | PC3, `192.168.10.13` | PC3, `192.168.10.13` | PC3, `192.168.10.13` |
| Signalement | Plus d'accès au réseau | Plus d'accès au réseau | Plus d'accès au réseau |
| Symptôme observé | `host (192.168.10.1) not reachable` | `host (192.168.10.1) not reachable` | `host (192.168.10.254) not reachable` |
| Couches testées | 3 → 2 → 1 | 3 → 2 → 1 → 2 | 2 → 3 |
| Couche en cause | 1 — Physique | 2 — Liaison | 3 — Réseau |
| Cause racine | Câble PC3–SW1 coupé | Port 2 de SW1 en VLAN 20 | Passerelle `.254` au lieu de `.1` |
| Action | Lien réactivé (Resume) | Port 2 remis en VLAN 1 | Passerelle corrigée + `save` |
| Vérification | Passerelle 5/5, TCP 443 ✅ | Passerelle 5/5, TCP 443 5/5 ✅ | `trace` comme la référence, TCP 443 5/5 ✅ |
| État | Résolue ✅ | Résolue ✅ | Résolue ✅ |

### 5.2 Ce qu'il faut retenir

- **Toujours comparer à un état de référence.** Sans les captures 01 à 04, on ne saurait pas que le `* * *` au troisième saut du `trace` est normal dans cette maquette.
- **Un même symptôme peut avoir plusieurs causes.** A et B affichent le même message : seul le test de la couche du dessous les distingue.
- **Le message d'erreur donne des indices.** L'adresse dans `host (…) not reachable` montre à qui le poste essaie de parler. En panne C, voir `.254` au lieu de `.1` mène directement à la passerelle.
- **Pour éviter que ça se reproduise :** étiqueter les câbles et les ports du switch, tenir à jour un tableau des ports et de leurs VLAN, et distribuer les adresses par DHCP pour éviter les erreurs de saisie de passerelle.

### 5.3 Schéma

📸 *Capture 22 — Schéma : maquette, emplacement des 3 pannes, méthode et résultat du diagnostic*

![Capture 22 — Schéma : maquette, emplacement des 3 pannes, méthode et résultat du diagnostic](https://hackmd.io/_uploads/rJ_0qDcofx.png)

Le même schéma en texte :

```
 Internet (VMnet8)      R1 c3725            SW1                  PC3
 gw 192.168.0.2  ---  Fa0/1 .0.8      ---  ports VLAN 1   ---   .10.13/24
                      Fa0/0 .10.1          PC3 = port 2         gw .1
                                           [B] port 2 VLAN 20   [A] câble coupé
                                                                [C] gw .254

 A  couche 1 : gw + voisin KO, lien suspendu   -> Resume
 B  couche 2 : gw + voisin KO, liens verts,
               port 2 en VLAN 20               -> VLAN 1
 C  couche 3 : voisin + gw OK, show ip = .254  -> gw .1 + save

 Re-tests : passerelle 5/5, TCP 443 vers 8.8.8.8 ✅
```

---

## Limites

- **Pannes créées par moi-même :** je connaissais la cause avant de commencer. J'ai quand même fait tous les tests dans l'ordre des couches, sans sauter directement à la réponse.
- **Pas de panne DNS :** PC3 n'a pas de DNS configuré, donc pas de résolution de noms à tester sur ce poste.
- **Postes VPCS et non Windows :** `show ip` et `trace` remplacent `ipconfig` et `tracert`. Ils donnent les mêmes informations.
- **ICMP vers Internet :** à cause du double NAT de la maquette (R1 puis VMware), les pings ICMP et le `trace` s'arrêtent après `192.168.0.2`. L'accès à Internet est prouvé par les connexions TCP 443.

---

## Incidents rencontrés

> **Incident 1 — Capture de la correction prise trop tôt (panne B)**
> **Constat :** sur la première capture de la correction, les champs indiquaient bien Port 2 et VLAN 1, mais le tableau des ports affichait encore le port 2 en VLAN 20. Avant cela, cliquer sur une ligne du tableau ne remplissait pas les champs : le champ Port restait sur 9.
> **Cause :** la capture a été prise avant de cliquer sur **Add**. Dans cette fenêtre, ce sont les champs de gauche qui comptent, pas la ligne sélectionnée.
> **Résolution :** clic sur **Add**, puis nouvelle capture avec le port 2 en VLAN 1 dans le tableau (capture 17). ✅

### Tableau récapitulatif des incidents

| N° | Machine | Incident | Cause | Résolution | État |
|---|---|---|---|---|---|
| 1 | SW1 | Capture de correction montrant encore VLAN 20 | Capture prise avant **Add** | Add, puis nouvelle capture | ✅ |

---

## Livrables

| Livrable attendu | Où le trouver dans ce document | État |
|---|---|---|
| Le réseau rétabli et fonctionnel | Section 4 : re-tests des trois pannes (captures 15 à 21) | ✅ |
| Un rapport d'intervention détaillant la démarche | Sections 2, 3 et 5 : méthode, diagnostic par couche, causes, fiche d'intervention, schéma (capture 22) | ✅ |
| Preuves des tests de diagnostic (ipconfig, ping, tracert) | `show ip`, `ping`, `trace` en sections 1.3, 2 et 4 (captures 01 à 06, 08, 12 à 14, 16, 18, 20, 21) | ✅ |
