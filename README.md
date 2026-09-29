*Projet réalisé dans le cadre du cursus de l'École 42 Paris par gustgonz.*

# NetPractice — adressage IP, sous-réseaux et routage

## Description

NetPractice est un projet de réseau TCP/IP : 10 niveaux de schémas réseau (hôtes, switchs, routeurs, « Internet ») à rendre fonctionnels en choisissant les **adresses IP**, les **masques** et les **routes**. Aucun code : tout se fait à la main, avec le calcul binaire des sous-réseaux.

Ce dépôt contient :

- `solutions/levelN.json` — mes configurations exportées depuis l'outil d'entraînement (uniquement les champs modifiables que j'ai remplis ; les valeurs imposées par l'exercice n'y figurent pas). Les adresses sont générées pour chaque étudiant, elles ne correspondent pas à celles d'un autre login. Le niveau 3 n'a pas été exporté.
- ce README — les notions du projet, illustrées par des calculs tirés de ces fichiers.

L'outil d'entraînement fourni par 42 n'est pas inclus : sa licence interdit de le redistribuer.

---

## Architecture

Exemple — niveau 8 (`solutions/level8.json`) : un bloc découpé en trois sous-réseaux /28 et une route agrégée.

```
                    Internet (I)
                        │  Internet → gateway 163.165.250.12 (interface R12)
                ┌───────┴───────┐
                │      R1       │  R13 168.214.158.62/28
                └───────┬───────┘  route 168.214.158.0/27 → 168.214.158.61
                        │ 168.214.158.48/28
                ┌───────┴───────┐
                │      R2       │  R21 168.214.158.61/28
                └──┬─────────┬──┘
     R22 .17/28    │         │    R23 .1/28
  168.214.158.16/28│         │168.214.158.0/28
                ┌──┴──┐   ┌──┴──┐
                │  C  │   │  D  │
                │ .20 │   │ .2  │
                └─────┘   └─────┘
      C : default → .17        D : default → .1
```

*(La destination de la route d'Internet et certaines valeurs sont imposées par l'exercice ; seules les valeurs des fichiers JSON sont les miennes.)*

---

## Ce que j'ai mis en place

### 1. Adresse IPv4 et masque

Une adresse IPv4 fait 32 bits, écrits en 4 octets décimaux. Le **masque** indique quelle partie désigne le **réseau** (bits à 1) et quelle partie désigne l'**hôte** (bits à 0). Deux machines peuvent se parler directement (sans routeur) si elles sont dans le **même réseau** : `IP ET masque` donne le même résultat.

Niveau 2 — `C1 192.168.69.5` et `D1 192.168.69.6` avec le masque `255.255.255.224` :

```
255.255.255.224 = 11111111.11111111.11111111.111 00000  → /27 (27 bits réseau, 5 bits hôte)
192.168.69.5    → réseau 192.168.69.0/27
192.168.69.6    → réseau 192.168.69.0/27   ✔ même réseau
```

### 2. Notation CIDR et taille d'un sous-réseau

`/n` = nombre de bits à 1 dans le masque. Avec `h = 32 − n` bits d'hôte : `2^h` adresses, dont **2 réservées** — la première (adresse du réseau) et la dernière (broadcast). Il reste `2^h − 2` adresses utilisables.

| CIDR | Masque | Adresses | Utilisables | Utilisé dans |
|---|---|---|---|---|
| /18 | 255.255.192.0 | 16 384 | 16 382 | niveaux 5, 9 |
| /24 | 255.255.255.0 | 256 | 254 | niveau 7 |
| /25 | 255.255.255.128 | 128 | 126 | niveaux 5, 6, 9, 10 |
| /26 | 255.255.255.192 | 64 | 62 | niveau 10 |
| /27 | 255.255.255.224 | 32 | 30 | niveau 2 |
| /28 | 255.255.255.240 | 16 | 14 | niveaux 4, 8 |
| /30 | 255.255.255.252 | 4 | 2 | niveaux 7, 9, 10 (liaisons routeur ↔ routeur) |

Méthode rapide : la taille du bloc = `256 − valeur de l'octet du masque`, et les réseaux commencent aux multiples de cette taille.

Niveau 4 — `B1 75.237.117.131/28`, `R1 75.237.117.129/28` : bloc de 16 → réseaux .0, .16, … .128, .144. `.131` et `.129` sont dans **75.237.117.128/28** : hôtes `.129` à `.142`, broadcast `.143`.

Niveau 5 — `B1 139.98.74.25/18` : le masque agit sur le 3ᵉ octet (`256 − 192 = 64`) → blocs .0, .64, .128, .192. `74` est dans le bloc 64 → réseau **139.98.64.0/18**, hôtes `139.98.64.1` à `139.98.127.254`. La passerelle `139.98.74.254` est bien dans ce réseau.

### 3. Découper un bloc en sous-réseaux sans chevauchement

Deux interfaces reliées à des routeurs différents doivent être dans des réseaux **disjoints**. Niveau 8, trois /28 dans le même /26 :

| Sous-réseau | Plage utilisable | Interfaces |
|---|---|---|
| 168.214.158.0/28 | .1 – .14 | R23 `.1`, D1 `.2` |
| 168.214.158.16/28 | .17 – .30 | R22 `.17`, C1 `.20` |
| 168.214.158.48/28 | .49 – .62 | R21 `.61`, R13 `.62` |

Niveau 10, même principe avec des tailles différentes dans 166.56.18.0/24 : `166.56.18.0/25` (H21 `.3`), `166.56.18.128/26` (R23 `.129`), `166.56.18.248/30` (R22 `.249` ↔ H31 `.250`).

Liaisons point à point en **/30** : exactement 2 adresses utilisables, une par extrémité. Niveau 7 : `101.198.14.0/30` (A1 `.2`) et `101.198.14.252/30` (R21 `.253`).

### 4. Routage et passerelle

Quand la destination n'est pas dans son réseau, une machine envoie le paquet à une **passerelle** (gateway) : un routeur **situé dans son propre réseau**. Une route = `destination → gateway`.

- **Route par défaut** `0.0.0.0/0` (ou `default`) : « tout le reste ». Niveau 6 : `A → 0.0.0.0/0 via 117.157.89.129`, l'interface R1 du routeur, dans le même /25 que A.
- **Route spécifique** : niveau 5, A (`51.238.229.121/25`) joint le réseau de B par `139.98.64.0/18 via 51.238.229.126` — `.126` est la dernière adresse utilisable de `51.238.229.0/25`, donc joignable directement par A.
- **Le chemin retour compte** : un ping n'aboutit que si la réponse a aussi une route. Au niveau 7, R1 a une route vers `111.198.14.0/24` et R2 une route vers `101.198.14.0/30`.
- **Agrégation de routes** : niveau 8, R1 atteint les deux sous-réseaux de R2 avec une seule route `168.214.158.0/27 via 168.214.158.61`, car `.0/28` et `.16/28` sont contenus dans `.0/27`. Même idée côté Internet au niveau 10 (`166.56.18.0/24`).
- **Internet** : les routeurs d'Internet ne connaissent pas nos réseaux ; il faut leur donner une route vers le bloc public, et les adresses **privées** (RFC 1918 : `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`) n'y sont jamais routées.

---

## Lancer / tester

Il n'y a rien à exécuter : les fichiers de `solutions/` sont au format d'export de l'outil NetPractice de 42 et ne sont valables que pour les adresses générées pour mon login.

Vérifier un calcul depuis un terminal :

```bash
python3 -c "import ipaddress as i; n=i.ip_interface('139.98.74.25/18').network; print(n, n.broadcast_address, n.num_addresses-2)"
# 139.98.64.0/18 139.98.127.255 16382
```

---

## Difficultés rencontrées

- **Masques non alignés sur un octet** (/18, /27…) : le calcul en décimal ne marche plus « à l'œil », il faut repasser par le binaire ou par la taille de bloc.
- **Chevauchements** : deux sous-réseaux qui se recouvrent semblent corrects séparément mais cassent le routage (niveaux 8 et 10).
- **Routes aller ET retour** : un ping échoue tant que la réponse n'a pas de chemin, même si l'aller fonctionne.
- **Adresses réservées** : ne pas attribuer l'adresse réseau ou le broadcast à une interface.

---

## Compétences mobilisées

- Adressage IPv4, masques, notation CIDR, calcul binaire
- Découpage en sous-réseaux (VLSM), liaisons /30
- Tables de routage : route par défaut, routes spécifiques, agrégation
- Rôle des passerelles, switchs et routeurs ; adresses publiques / privées (RFC 1918)
- Méthode de diagnostic : vérifier réseau, passerelle, route aller et route retour
