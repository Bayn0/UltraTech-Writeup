# Writeup FR - Ultratech (TryHackMe)

## Description

Ce writeup présente l'exploitation de la machine Ultratech disponible sur TryHackMe. L'objectif est de compromettre la machine en suivant une méthodologie structurée : énumération, exploitation d'une vulnérabilité backend, puis élévation de privilèges via une mauvaise configuration système.

**Méthodologie adoptée :** Reconnaissance → Énumération → Exploitation → Privilege Escalation

---

## Reconnaissance

Pour commencer, j'ai effectué un scan de ports TCP complet avec `nmap` afin d'identifier les services exposés (voir `artifacts/scan.xml`).

### Services découverts

- **FTP (21)**
- **SSH (22)**
- **Service HTTP sur le port 8081**
- **Service HTTP sur le port 31331**

L'accès FTP anonyme a été testé mais refusé, ce qui permet d'écarter ce vecteur d'attaque initial.

Le scan nmap nous a également dévoilé deux ports HTTP : le premier, le port **8081**, est utilisé par **Node.js**, et le **31331** héberge quant à lui un serveur **Apache**.

On passe donc à l'analyse web.

---

## Énumération web

On utilise `gobuster` pour découvrir les répertoires cachés sur les deux ports HTTP (voir `artifacts/gobuster_8081.txt` et `artifacts/gobuster_31331.txt`).

### Port 8081 — API Node.js

- `/auth` — Authentification
- `/ping` — Vérification de disponibilité

Cela confirme qu'il y a une API backend qui gère l'authentification et les vérifications système.

### Port 31331 — Serveur Apache (Frontend)

De nombreuses ressources ont été trouvées, notamment :

- `/robots.txt`
- Répertoire `/js` contenant des fichiers JavaScript
- Page `partners.html`

En consultant `robots.txt`, on découvre un sitemap supplémentaire qui révèle `partners.html`, une page de login destinée aux partenaires.

On peut en déduire que le serveur Apache agit comme frontend et utilise l'API Node.js sur le port 8081.

### Analyse du code JavaScript

En analysant le fichier `API.js` (trouvé dans `/js`), on peut observer comment le frontend interagit avec l'API :

- L'endpoint `/auth` gère l'authentification
- L'endpoint `/ping` vérifie la disponibilité de l'API

En accédant directement à `/ping` sans paramètres :

```
http://TARGET_IP:8081/ping
```

On obtient une erreur serveur qui affiche une stack trace Node.js complète et les chemins internes du système de fichiers. La divulgation de ces informations confirme une mauvaise gestion des erreurs et révèle des détails sensibles sur l'implémentation backend.

### Vulnérabilité identifiée : Command Injection

L'endpoint `/ping` accepte un paramètre `ip` qui peut être contrôlé par l'utilisateur, censé représenter une adresse IP ou un nom d'hôte. Le problème est que ce paramètre est utilisé directement dans une commande système sans validation, ce qui suggère fortement une **vulnérabilité d'injection de commandes**.

La sortie brute de la commande exécutée est renvoyée par l'API, ce qui permet à un attaquant d'interagir indirectement avec le système sous-jacent.

Nous pouvons donc tester une injection de commande telle que `ls`, ce qui donnerait ceci :

```
http://TARGET_IP:8081/ping?ip=`ls`
```

**Résultat :** La commande `ls` s'exécute et on voit les fichiers du répertoire.

---

## Exploitation

*(Voir les captures associées dans le dossier `artifacts/`)*

En exploitant la command injection, on liste les fichiers et on découvre **`utech.db.sqlite`**.

On lit le contenu de cette base de données :

```
http://TARGET_IP:8081/ping?ip=`cat%20utech.db.sqlite`
```

La base de données contient des identifiants d'authentification incluant un hash de mot de passe pour l'utilisateur **r00t**.

On déchiffre le hash (avec CrackStation, Hashcat ou John the Ripper) et on obtient le mot de passe en clair.

### Connexion SSH

Nous pouvons désormais établir une connexion SSH sur le port 22 avec les identifiants et le mot de passe obtenus :

```bash
ssh r00t@TARGET_IP
```

**Accès obtenu !**

---

## Privilege Escalation

Une fois connecté en tant que **r00t**, nous pouvons vérifier les groupes de l'utilisateur actuel grâce à la commande :

```bash
id
```

Ce qui révèle que nous appartenons au groupe **docker**.

Cette appartenance est critique, car elle permet d'exécuter des conteneurs Docker avec des privilèges élevés, offrant un chemin direct vers l'escalade de privilèges.

### Exploitation via Docker

On consulte **GTFOBins** pour la technique d'exploitation.

On liste les images disponibles :

```bash
docker images
```

On lance un conteneur en montant le système de fichiers racine de l'hôte :

```bash
docker run -v /:/mnt --rm -it bash chroot /mnt sh
```

**Explication :**

- `-v /:/mnt` : Monte la racine de l'hôte dans `/mnt` du conteneur
- `chroot /mnt` : Change la racine vers le système de fichiers de l'hôte

On obtient un shell avec les privilèges root sur la machine hôte.

**Root obtenu !**

---

## Notes sur les artefacts

L'ensemble des preuves techniques est fourni dans le dossier `artifacts/` :

- Résultats du scan Nmap
- Sorties d'énumération de répertoires
- Captures d'écran des étapes clés

Les informations sensibles ont volontairement été exclues de ce document.
