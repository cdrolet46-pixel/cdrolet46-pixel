# Configuration Serveur & Infrastructure

## Droplet DigitalOcean

- **IP** : 142.93.145.22
- **Nom** : DrovezSRV
- **OS** : Ubuntu 24.04.4 LTS
- **RAM** : 1.9 Go | **Disque** : 67 Go
- **Acces SSH** : `ssh -i ~/.ssh/id_ed25519 root@142.93.145.22`
- **Cle SSH** : `~/.ssh/id_ed25519` (ED25519, commentaire: cdrolet46-pixel-digitalocean)

## Domaine

- **Domaine** : drovez.ca
- **DNS** : Cloudflare
- **Proxy Cloudflare** : desactive (DNS only) pour les sous-domaines avec SSL Let's Encrypt

## Docker — /opt/docker/

Tous les services sont dans `/opt/docker/<service>/` avec un `docker-compose.yml` et un `.env`.

Commandes utiles :
```bash
cd /opt/docker/n8n
docker compose up -d       # demarrer
docker compose down        # arreter
docker compose logs -f     # logs en temps reel
docker compose ps          # statut
```

## n8n

- **URL** : https://n8n.drovez.ca
- **Dossier** : `/opt/docker/n8n/`
- **Port interne** : 5678 (non expose publiquement)
- **Timezone** : America/Toronto
- **Cle de chiffrement** : dans `/opt/docker/n8n/.env` (N8N_ENCRYPTION_KEY)
- **Donnees** : volume Docker `n8n_n8n_data`

## Caddy (reverse proxy)

- **Image** : caddy:latest
- **Ports** : 80, 443 (HTTP/HTTPS)
- **SSL** : Let's Encrypt automatique (ACME)
- **Config** : `/opt/docker/n8n/Caddyfile`

```
n8n.drovez.ca {
    reverse_proxy n8n:5678
}
```

Pour ajouter un nouveau service, ajouter un bloc dans le Caddyfile et un record DNS A dans Cloudflare (proxy OFF).

## Ajouter un nouveau service

1. Creer `/opt/docker/<service>/docker-compose.yml`
2. Ajouter le service au reseau `web` sans exposer le port publiquement
3. Ajouter un bloc dans `/opt/docker/n8n/Caddyfile`
4. `docker compose restart` dans le dossier n8n (pour recharger Caddy)
5. Ajouter un record DNS A dans Cloudflare (proxy OFF)

## GitHub

- **Compte** : cdrolet46-pixel
- **gh CLI** : `C:\Program Files\GitHub CLI\gh.exe`
- **Repos** : outils interactifs HTML (Cisco CLI, quiz CompTIA, emulateurs)

## Clef API n8n

Disponible dans n8n : Settings → API (ne jamais stocker en clair)

## Workflows n8n

### Veille Technologique Quotidienne
- **ID** : NV4gvhEsABkR3pEZ
- **Declencheur** : Cron 8h chaque matin
- **Sources** : Hacker News, Krebs on Security, The Hacker News (RSS)
- **IA** : Groq API (llama-3.3-70b-versatile) — resume et groupe par theme
- **Sortie** : Digest envoye sur Discord via webhook
- **Noeuds** : Schedule → RSS x3 → Merge → Limit 15 → Aggregate → Prep Groq (Code) → Groq HTTP → Prep Discord (Code) → Discord HTTP
- **Backup** : Script `/opt/docker/n8n/backup-workflows.sh` — cron toutes les heures vers `cdrolet46-pixel/n8n-workflows`

## Backup workflows

- **Script** : `/opt/docker/n8n/backup-workflows.sh`
- **Cron** : toutes les heures
- **Repo GitHub** : `cdrolet46-pixel/n8n-workflows` (prive)
- **Logs** : `/var/log/n8n-backup.log`
- **Cle SSH deploy** : `/root/.ssh/n8n_github`