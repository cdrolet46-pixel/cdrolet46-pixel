# Guide d'infrastructure — drovez.ca

Documentation complète de la mise en place du serveur, Docker, n8n et GitHub.

---

## Table des matières

1. [Clé SSH](#1-clé-ssh)
2. [Profil GitHub](#2-profil-github)
3. [Droplet DigitalOcean](#3-droplet-digitalocean)
4. [Installation Docker](#4-installation-docker)
5. [Configuration DNS Cloudflare](#5-configuration-dns-cloudflare)
6. [n8n + Caddy (reverse proxy SSL)](#6-n8n--caddy-reverse-proxy-ssl)
7. [Backup automatique des workflows n8n vers GitHub](#7-backup-automatique-des-workflows-n8n-vers-github)
8. [Ajouter un nouveau service](#8-ajouter-un-nouveau-service)

---

## 1. Clé SSH

Génération d'une clé SSH ED25519 sur Windows pour accéder au droplet.

```powershell
ssh-keygen -t ed25519 -C "cdrolet46-pixel-digitalocean" -f "$env:USERPROFILE/.ssh/id_ed25519" -N '""'
```

Fichiers générés :
- `C:\Users\Admin\.ssh\id_ed25519` — clé privée (ne jamais partager)
- `C:\Users\Admin\.ssh\id_ed25519.pub` — clé publique

**Ajouter la clé publique dans DigitalOcean :**
Settings → Security → SSH Keys → Add SSH Key

**Se connecter au droplet :**
```powershell
ssh -i $env:USERPROFILE\.ssh\id_ed25519 root@142.93.145.22
```

---

## 2. Profil GitHub

### Amélioration des repos existants

Mise à jour des descriptions et ajout de topics via `gh` CLI :

```powershell
$GH = "C:\Program Files\GitHub CLI\gh.exe"

& $GH repo edit cdrolet46-pixel/CISCO-SWITCH-CLI `
  --description "Reference interactive des commandes CLI pour switch Cisco" `
  --add-topic "cisco,networking,cli,html,cheatsheet"
```

### README de profil

Créer un repo nommé exactement comme ton username : `cdrolet46-pixel/cdrolet46-pixel`

```powershell
& $GH repo create cdrolet46-pixel/cdrolet46-pixel --public
```

Le fichier `README.md` à la racine de ce repo s'affiche automatiquement sur ta page de profil GitHub.

---

## 3. Droplet DigitalOcean

| Paramètre | Valeur |
|-----------|--------|
| IP | 142.93.145.22 |
| Nom | DrovezSRV |
| OS | Ubuntu 24.04.4 LTS |
| RAM | 1.9 Go |
| Disque | 67 Go |

**Connexion SSH :**
```bash
ssh -i ~/.ssh/id_ed25519 root@142.93.145.22
```

---

## 4. Installation Docker

```bash
curl -fsSL https://get.docker.com | sh
```

Vérifie l'installation :
```bash
docker --version
docker compose version
```

**Structure des dossiers :**
```
/opt/docker/
└── n8n/
    ├── docker-compose.yml
    ├── Caddyfile
    └── .env
```

Chaque service a son propre dossier dans `/opt/docker/`.

---

## 5. Configuration DNS Cloudflare

Pour chaque nouveau service, ajouter un record DNS dans Cloudflare :

| Type | Name | Content | Proxy |
|------|------|---------|-------|
| A | `n8n` | `142.93.145.22` | OFF (nuage gris) |

> **Important :** Le proxy Cloudflare doit être **désactivé** pour les sous-domaines qui utilisent le SSL Let's Encrypt via Caddy. Si le proxy est activé, le renouvellement automatique du certificat échouera au bout de 90 jours.

---

## 6. n8n + Caddy (reverse proxy SSL)

### Structure des fichiers

**`/opt/docker/n8n/docker-compose.yml`**
```yaml
services:
  caddy:
    image: caddy:latest
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
      - "443:443/udp"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
      - caddy_data:/data
      - caddy_config:/config
    networks:
      - web

  n8n:
    image: n8nio/n8n:latest
    restart: unless-stopped
    environment:
      - N8N_HOST=n8n.drovez.ca
      - N8N_PORT=5678
      - N8N_PROTOCOL=https
      - NODE_ENV=production
      - WEBHOOK_URL=https://n8n.drovez.ca/
      - GENERIC_TIMEZONE=America/Toronto
      - N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY}
      - N8N_SOURCECONTROL_ENABLED=true
    volumes:
      - n8n_data:/home/node/.n8n
    networks:
      - web

volumes:
  caddy_data:
  caddy_config:
  n8n_data:

networks:
  web:
```

**`/opt/docker/n8n/Caddyfile`**
```
n8n.drovez.ca {
    reverse_proxy n8n:5678
}
```

**`/opt/docker/n8n/.env`**
```
N8N_ENCRYPTION_KEY=<clé générée automatiquement>
N8N_API_KEY=<clé API n8n>
```

### Commandes utiles

```bash
cd /opt/docker/n8n

docker compose up -d          # démarrer
docker compose down           # arrêter
docker compose restart        # redémarrer
docker compose logs -f        # logs en temps réel
docker compose ps             # statut des containers
docker compose pull && docker compose up -d  # mettre à jour les images
```

### Accès

- **URL** : https://n8n.drovez.ca
- **SSL** : Let's Encrypt automatique via Caddy (renouvellement automatique)

---

## 7. Backup automatique des workflows n8n vers GitHub

### Repo GitHub

Repo privé : `cdrolet46-pixel/n8n-workflows`

### Clé SSH dédiée sur le droplet

```bash
ssh-keygen -t ed25519 -C "n8n-drovez-droplet" -f /root/.ssh/n8n_github -N ""
```

Ajouter la clé publique comme Deploy Key dans GitHub :
```bash
cat /root/.ssh/n8n_github.pub
```
GitHub → repo `n8n-workflows` → Settings → Deploy keys → Add deploy key (write access)

**Config SSH `~/.ssh/config` sur le droplet :**
```
Host github.com
    HostName github.com
    User git
    IdentityFile /root/.ssh/n8n_github
    StrictHostKeyChecking no
```

### Script de backup

**`/opt/docker/n8n/backup-workflows.sh`**
```bash
#!/bin/bash
set -e

export $(grep -v '^#' /opt/docker/n8n/.env | xargs)

REPO_DIR=/opt/n8n-workflows
N8N_URL=https://n8n.drovez.ca
DATE=$(date +"%Y-%m-%d %H:%M")

if [ ! -d "$REPO_DIR" ]; then
  git clone git@github.com:cdrolet46-pixel/n8n-workflows.git $REPO_DIR
  git -C $REPO_DIR config user.email "n8n@drovez.ca"
  git -C $REPO_DIR config user.name "n8n-drovez"
fi

mkdir -p $REPO_DIR/workflows

curl -s -H "X-N8N-API-KEY: $N8N_API_KEY" "$N8N_URL/api/v1/workflows?limit=100" | \
  python3 -c "
import json,sys
data = json.load(sys.stdin)
workflows = data.get('data', [])
for wf in workflows:
    name = wf['name'].replace('/', '-').replace(' ', '_')
    wid = wf['id']
    fname = f'/opt/n8n-workflows/workflows/{wid}_{name}.json'
    with open(fname, 'w') as f:
        json.dump(wf, f, indent=2)
    print(f'Exported: {fname}')
"

cd $REPO_DIR
git add -A
if git diff --cached --quiet; then
  echo "Aucun changement a pousser"
else
  git commit -m "backup: $DATE"
  git push origin main
  echo "Push reussi"
fi
```

### Cron — backup toutes les heures

```bash
crontab -e
# Ajouter :
0 * * * * /opt/docker/n8n/backup-workflows.sh >> /var/log/n8n-backup.log 2>&1
```

Vérifier les logs :
```bash
tail -f /var/log/n8n-backup.log
```

---

## 8. Ajouter un nouveau service

1. Créer le dossier du service :
```bash
mkdir -p /opt/docker/MON-SERVICE
```

2. Créer `docker-compose.yml` — connecter au réseau `web`, ne pas exposer le port publiquement

3. Ajouter un bloc dans `/opt/docker/n8n/Caddyfile` :
```
mon-service.drovez.ca {
    reverse_proxy mon-service:PORT
}
```

4. Redémarrer Caddy pour prendre en compte la nouvelle config :
```bash
cd /opt/docker/n8n
docker compose restart caddy
```

5. Ajouter un record DNS dans Cloudflare :
   - Type `A`, Name `mon-service`, Content `142.93.145.22`, Proxy **OFF**

6. Démarrer le service :
```bash
cd /opt/docker/MON-SERVICE
docker compose up -d
```
