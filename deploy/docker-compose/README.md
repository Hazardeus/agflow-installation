# Stack ag-flow mutualisée (portal + rag + doc)

Installation prod **reproductible** d'une stack `docker compose` **unique**
hébergeant les produits ag-flow, avec les composants partagés **mutualisés**
(voir objectifs dans `../../CLAUDE.md`).

## Mutualisation

- **1 Postgres** (`pgvector/pgvector:pg16`) partagé :
  - rôle/base `rag` (superuser — crée aussi les bases workspace dynamiques de rag) ;
  - rôle/base `docflow` (`initdb/10-docflow.sh`) ;
  - rôle/base `portal` (`initdb/20-portal.sh`).
- **1 réseau** Docker (`agflow`).
- **1 reverse-proxy Caddy** (API admin `:2019` pour les routes `ws-*` du portal) :
  - `portal` → hôte de base `{BASE_DOMAIN}` (+ `ws-*.{BASE_DOMAIN}` dynamiques) ;
  - `rag` → `rag.{BASE_DOMAIN}` ;
  - `doc` → `doc.{BASE_DOMAIN}`.

## Produits

| Produit | Image | Accès |
|---|---|---|
| portal | `ghcr.io/gaelgael5/workspace-portal` | `{BASE_DOMAIN}` (+ `ws-*`) |
| rag | `ghcr.io/ag-flow/rag-backend` + `rag-frontend` | `rag.{BASE_DOMAIN}` |
| doc | `ghcr.io/ag-flow/doc` | `doc.{BASE_DOMAIN}` |

Mode de livraison : **PULL** (images publiques ghcr). Aucun build, aucune
dépendance tirée des dépôts sources à l'exécution. TLS terminé en amont par
Cloudflare Tunnel — Caddy tourne en HTTP simple (`auto_https off`).

## Installation (sur la machine cible)

```bash
git clone https://github.com/ag-flow/installation.git
cd installation
deploy/docker-compose/deploy.sh
```

Le script :
1. prépare les répertoires durables sous `/srv/agflow` (bind mounts explicites,
   voir plus bas) — Postgres chown `999:999` ;
2. copie les dépendances dans `DEPLOY_DIR` et génère `DEPLOY_DIR/.env`
   (secrets rag + doc + mot de passe Postgres portal, non-interactif, idempotent) ;
3. initialise `PORTAL_DATA_DIR` (CA, certs, `config.yaml`, `.env`) via le
   `portal/install.sh` vendorisé, puis y complète `DATABASE_URL` + `PORTAL_VAULT_KEK` ;
4. `docker compose pull && up -d`, applique les migrations Alembic du portal ;
5. vérifie la santé de **portal**, **rag** et **doc**.

Les mots de passe admin générés sont affichés en fin d'exécution.

### Variables optionnelles

| Variable | Défaut | Rôle |
|---|---|---|
| `DEPLOY_DIR` | `/srv/agflow/app` | répertoire d'installation (compose, .env, Caddyfile...) |
| `BASE_DOMAIN` | `agflow.local` | portal = base ; rag/doc en sous-domaines |
| `RAG_PUBLIC_URL` | `http://<ip-hôte>` | URL publique rag |
| `IMAGE_TAG` / `DOC_IMAGE_TAG` / `PORTAL_IMAGE_TAG` | `latest`/`latest`/`main` | versions |
| `GHCR_TOKEN` | *(vide)* | token `read:packages` si images privées |
| `PORTAL_DATA_DIR` | `/srv/agflow/portal-data` | données portal (CA, certs, config, .env) |
| `POSTGRES_DATA_DIR` | `/srv/agflow/data/postgres` | données Postgres (chown `999:999`) |
| `RAG_REPOS_DIR` | `/srv/agflow/data/rag-repos` | dépôts indexés par rag |
| `CADDY_DATA_DIR` | `/srv/agflow/data/caddy-data` | état Caddy (certs internes) |
| `CADDY_CONFIG_DIR` | `/srv/agflow/data/caddy-config` | config runtime Caddy |

> Le portal stocke sa config et ses secrets sous `PORTAL_DATA_DIR` (CA, certs,
> `config.yaml`, `.env`) — distinct de `DEPLOY_DIR`. Le conteneur portal voit
> toujours ce répertoire sous `/data` (chemin interne inchangé).

### Disposition sur disque (VM Azure `vm-agflow-control-lab`)

```
/srv/agflow/
├── app/                 # DEPLOY_DIR — compose, .env, Caddyfile, initdb/, homepage/
├── portal-data/         # PORTAL_DATA_DIR — CA, certs, config.yaml, .env, .devpod
├── data/
│   ├── postgres/        # POSTGRES_DATA_DIR — chown 999:999 (uid/gid "postgres")
│   ├── rag-repos/       # RAG_REPOS_DIR
│   ├── caddy-data/      # CADDY_DATA_DIR
│   └── caddy-config/    # CADDY_CONFIG_DIR
└── backups/             # réservé — non encore câblé dans la stack
```

Ces chemins sont des **bind mounts stack-scoped** (pas de volumes Docker
nommés) : tout vit sous `/srv/agflow`, le disque managé persistant de la VM.
Le data-root global du moteur Docker (`/var/lib/docker`) n'est pas modifié.

## Contenu

| Fichier | Rôle |
|---|---|
| `deploy.sh` | installe la stack complète |
| `docker-compose.yml` | stack mutualisée (postgres, backend, frontend, doc, portal, caddy) |
| `Caddyfile` | reverse-proxy commun (admin API + routage par hôte) |
| `initdb/10-docflow.sh`, `initdb/20-portal.sh` | création des rôles/bases doc & portal |
| `portal/install.sh` | init `/data` du portal (vendorisé depuis devpod-ui) |
| `pricing.yml` | tarifs embeddings rag (lecture seule) |
| `.env.example` | gabarit de configuration (rag + doc + portal) |
