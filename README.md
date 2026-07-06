# RemoteWake

*(anciennement Serv_ktin)*

**RemoteWake** est une application web permettant de démarrer et surveiller un PC à distance via **Wake-on-LAN (WoL)**, hébergée sur un **Raspberry Pi** et accessible de n'importe où grâce à un **VPN Tailscale**. Une fois le PC démarré, l'utilisateur peut s'y connecter en **RDP** pour travailler dessus à distance.

> Il s'agit d'une **v1 / projet d'apprentissage** : premier projet complet de l'auteur combinant réseau, backend Flask et base de données. Le code fonctionne mais contient des choix de configuration à ne pas reproduire en production (voir la section [Sécurité](#-sécurité--points-à-corriger-avant-toute-exposition)).

## Objectif du projet

Pouvoir, depuis n'importe quel endroit :
1. Se connecter à une interface web protégée par un compte utilisateur.
2. Réveiller son PC principal via un paquet **Wake-on-LAN**.
3. Vérifier que le PC est bien allumé et joignable sur le réseau.
4. Surveiller l'état du VPN, la température CPU et l'utilisation RAM du PC.
5. Se connecter ensuite dessus en RDP pour travailler comme si on y était.

## Architecture

```
[ Toi, n'importe où ]
        │  (Tailscale VPN)
        ▼
[ Raspberry Pi ] ── héberge l'app Flask (Serv_ktin)
        │  ── envoie le paquet Wake-on-LAN sur le réseau local
        │  ── interroge une API locale (OpenHardwareMonitor) pour CPU/RAM
        ▼
[ PC principal ] ── se réveille (WoL) ── accessible ensuite en RDP
```

- Le **Raspberry Pi** reste allumé en permanence sur le réseau local et sert de point d'entrée.
- **Tailscale** crée un réseau privé virtuel entre le Raspberry Pi et tes autres appareils, sans avoir à ouvrir de ports sur ta box internet.
- Le PC principal doit avoir le **Wake-on-LAN activé dans le BIOS/UEFI et dans les paramètres de la carte réseau** (Windows : panneau de gestion des périphériques → carte réseau → Gestion de l'énergie).
- Une fois le PC démarré, la connexion de travail se fait en **RDP** (Bureau à distance), en dehors de l'application elle-même.

## Fonctionnalités

- Authentification par compte utilisateur (mot de passe hashé avec **bcrypt**)
- Réveil du PC à distance (Wake-on-LAN)
- Vérification de l'état du PC (ping) et du VPN (ping Tailscale)
- Suivi de la température CPU et de la charge RAM (via une API locale type OpenHardwareMonitor exposée sur le PC)
- Page d'administration pour ajouter/supprimer des comptes utilisateurs
- Interface web (HTML/CSS/JS) légère, pensée pour tourner sur un Raspberry Pi

## Stack technique

| Composant | Technologie |
|---|---|
| Backend | Python 3.13, Flask |
| Base de données | MySQL (via `flask-mysqldb`) |
| Authentification | `flask-bcrypt` |
| Réseau | Wake-on-LAN (socket UDP), ping ICMP |
| VPN | Tailscale |
| Hébergement | Raspberry Pi |
| Gestion des dépendances | Poetry |

## Structure du projet

```
RemoteWake/
├── app.py            # Application Flask : routes, auth, admin
├── utils.py           # Fonctions utilitaires : ping, WoL, CPU/RAM
├── mysql.py            # (réservé à la config/connexion MySQL)
├── static/             # CSS / JS / assets
├── templates/          # Templates HTML (login, index, admin)
├── pyproject.toml       # Dépendances Poetry
└── poetry.lock
```

## Installation

### Pré-requis

- Un Raspberry Pi (ou toute machine Linux) allumé en permanence sur le même réseau que le PC à réveiller
- Python ≥ 3.13
- [Poetry](https://python-poetry.org/)
- Un serveur MySQL/MariaDB
- Un compte [Tailscale](https://tailscale.com/) installé sur le Raspberry Pi et sur les appareils depuis lesquels tu veux te connecter
- Wake-on-LAN activé sur le PC cible (BIOS + carte réseau)
- Un client RDP (Bureau à distance Windows, Remmina, Microsoft Remote Desktop, etc.)

### Étapes

```bash
# 1. Cloner le repo
git clone https://github.com/Thepizon/Serv_ktin.git
cd RemoteWake

# 2. Installer les dépendances
poetry install

# 3. Configurer la base de données MySQL
mysql -u root -p
CREATE DATABASE Serv_Ktin;
USE Serv_Ktin;
CREATE TABLE user (
    User_name VARCHAR(50) NOT NULL PRIMARY KEY,
    Password VARCHAR(255) NOT NULL
);

# 4. Configurer les variables d'environnement (voir section Sécurité)
cp .env.example .env
# puis éditer .env avec tes propres identifiants

# 5. Lancer l'application
poetry run python app.py
```

### Configuration Tailscale

1. Installer Tailscale sur le Raspberry Pi : `curl -fsSL https://tailscale.com/install.sh | sh`
2. Le connecter à ton compte : `tailscale up`
3. Installer Tailscale sur les appareils depuis lesquels tu veux accéder à l'app
4. Récupérer l'IP Tailscale du Raspberry Pi (`tailscale ip -4`) pour y accéder à distance

## Sécurité — points à corriger avant toute exposition

> **Note (suite à une revue de code récente) :** plusieurs failles ci-dessous ont déjà été identifiées et corrigées dans une version ultérieure (dont la gestion des mots de passe). Le reste n'est **pas dangereux dans l'usage actuel** : l'application n'est accessible que via mon réseau **Tailscale** (VPN privé) et personne d'autre que moi n'a accès à ce tailnet — ces failles ne sont donc exploitables par personne en l'état.
>
> Elles restent toutefois **à corriger dans une future v2**, notamment si le tailnet est un jour partagé avec quelqu'un, si le service venait à être exposé publiquement, ou si ce code est repris tel quel par quelqu'un d'autre (les secrets codés en dur, en particulier, seraient alors exploitables sur un système qu'on croit sécurisé).

Voici les points identifiés lors d'une relecture du code, du plus critique au moins critique :

### Critique

1. **Identifiants MySQL en dur dans `app.py` et poussés sur un repo public** (`MYSQL_USER`, `MYSQL_PASSWORD`). Ce mot de passe doit être considéré comme compromis : il faut le **changer immédiatement côté MySQL**, et ne plus jamais committer de secrets en clair.
2. **`app.secret_key` codée en dur dans le code source.** Cette clé signe les cookies de session : si elle fuite (ce qui est le cas ici, publiquement), n'importe qui peut forger un cookie de session valide et se faire passer pour un utilisateur connecté.
3. **Route `/add` (création de compte) accessible sans authentification.** Rien ne vérifie que l'appelant est connecté ou admin avant de créer un compte : n'importe qui connaissant l'URL peut créer un utilisateur, y compris un accès potentiellement équivalent à un accès admin selon la logique de l'app.
4. **`app.run(debug=True, ...)`** : le mode debug de Flask expose le débogueur Werkzeug interactif, qui permet **l'exécution de code arbitraire à distance** si la page d'erreur est atteignable. À ne jamais activer sur une instance exposée, même via VPN.
5. **Toutes les routes de monitoring (`/status`, `/vpn`, `/cpu`, `/ram`) et `/wol` ne vérifient pas la session.** N'importe qui pouvant atteindre le serveur (donc, toute personne sur ton tailnet ou sur le même réseau) peut réveiller le PC ou lire ces infos sans être connecté.

### Important

6. **Pas de protection CSRF** sur les formulaires (`login`, `add`, `del`) : sans jeton anti-CSRF, un site tiers pourrait déclencher ces actions à l'insu de l'utilisateur connecté.
7. **Validation du nom d'utilisateur incomplète** : `re.match(r'[A-Za-z0-9]+', username)` vérifie seulement que la chaîne *commence* par des caractères alphanumériques, pas qu'elle en est *entièrement* composée (il faudrait `re.fullmatch`).
8. **Pas de limitation du nombre de tentatives de connexion** (brute-force possible sur `/login`).
9. **Cookies de session sans attributs de sécurité explicites** (`Secure`, `HttpOnly`, `SameSite`) — à configurer via `app.config['SESSION_COOKIE_...']`.

### Mineur / bonnes pratiques

10. `python-dotenv` est dans les dépendances mais n'est pas utilisé dans `app.py` : les secrets sont en dur au lieu d'être chargés depuis un `.env` (qui devrait lui-même être dans `.gitignore`).
11. `host='192.168.1.50'` codé en dur dans `app.run()` : fragile si l'IP du Raspberry Pi change ; en local mieux vaut `0.0.0.0` (en le protégeant par le VPN/pare-feu) ou lire l'IP depuis la config.
12. Le fichier `.DS_Store` et le dossier `__pycache__` sont commités : à ajouter à `.gitignore`.
13. Utilisation du serveur de développement Flask (`app.run`) plutôt qu'un serveur WSGI de production (Gunicorn, Waitress) — suffisant pour un usage perso derrière Tailscale, mais à noter.
14. Point positif à souligner : les requêtes SQL utilisent bien des **requêtes paramétrées** (`%s` + tuple), ce qui protège contre l'injection SQL classique — bon réflexe déjà en place. Les mots de passe sont également correctement **hashés avec bcrypt**.

### Recommandations rapides

- Changer le mot de passe MySQL et régénérer une `secret_key` aléatoire (`python -c "import secrets; print(secrets.token_hex(32))"`), les charger via variables d'environnement (`.env` + `python-dotenv`, déjà en dépendance) et ne jamais les committer.
- Ajouter un décorateur type `@login_required` sur toutes les routes sensibles (`/add`, `/del`, `/wol`, `/status`, `/vpn`, `/cpu`, `/ram`, `/admin`).
- Vérifier en plus le rôle admin (`session['username'] == 'ktin'`) sur `/add` et `/del`, comme c'est déjà fait sur `/admin`.
- Passer `debug=False` dès que l'app n'est plus testée en local.
- Ajouter `.env`, `__pycache__/` et `.DS_Store` à un `.gitignore`.
- Nettoyer l'historique Git (`git filter-repo` ou BFG Repo-Cleaner) pour supprimer les secrets déjà commités, en plus de les changer.

## Roadmap (idées pour une v2)

- [ ] Authentification par rôle propre (admin / utilisateur) au lieu d'un nom codé en dur
- [ ] Protection CSRF (Flask-WTF)
- [ ] Variables d'environnement pour toute la config sensible
- [ ] Déploiement derrière un vrai serveur WSGI (Gunicorn + Nginx)
- [ ] Notifications (Discord/Telegram) au démarrage du PC
- [ ] Historique des connexions / logs d'activité

## Licence

Non spécifiée — projet personnel à but d'apprentissage.

## Auteur

Projet développé par **Quentin Remacle (Thepizon)**.
