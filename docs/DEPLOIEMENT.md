# Déploiement — Wiki interne RAG GeniusPay

Guide pas à pas pour déployer le wiki RAG (n8n + PostgreSQL/pgvector) sur le serveur GeniusPay, avec le domaine `wiki.geniuspay.ci`, et pour créer les workflows, la base de données et les credentials nécessaires.

Prérequis : accès SSH ou UI au serveur GeniusPay (Coolify), accès DNS Cloudflare de `geniuspay.ci`, une clé API Anthropic et une clé API OpenAI (embeddings).

## 0. Vue d'ensemble des étapes

1. DNS Cloudflare (`wiki.geniuspay.ci`, `n8n.geniuspay.ci`)
2. Déploiement des conteneurs (Postgres+pgvector, n8n) via Coolify
3. Vérification de la base de données (migration auto au premier démarrage)
4. Création des credentials dans n8n
5. Import et activation des 4 workflows
6. Routage Coolify/Cloudflare vers les bons webhooks
7. Tests bout en bout
8. Sécurisation du backoffice

## 1. DNS (Cloudflare)

Dans la zone `geniuspay.ci` sur Cloudflare, ajouter deux enregistrements pointant vers le serveur GeniusPay :

| Type | Nom | Cible | Proxy |
|---|---|---|---|
| A ou CNAME | `wiki` | IP du serveur / hostname Coolify | Activé (orange) |
| A ou CNAME | `n8n` | IP du serveur / hostname Coolify | Activé (orange), ou désactivé si tu préfères garder l'éditeur n8n hors CDN |

`wiki.geniuspay.ci` sert le chat public + le backoffice (webhooks n8n). `n8n.geniuspay.ci` est optionnel — utile si tu veux un accès direct à l'éditeur n8n séparé de l'URL publique.

## 2. Déploiement des conteneurs (Coolify)

1. Dans Coolify, créer une nouvelle **Resource** de type *Docker Compose*, pointant sur ce dépôt Git et le fichier `docker-compose.yml` à la racine.
2. Définir les variables d'environnement du service (Coolify → Environment Variables), à partir de `.env.example` :

```env
APP_NAME="GeniusPay Wiki"
APP_URL=https://wiki.geniuspay.ci

DB_PASSWORD=<mot de passe fort, généré>

N8N_HOST=n8n.geniuspay.ci
N8N_WEBHOOK_URL=https://wiki.geniuspay.ci/webhook

ANTHROPIC_API_KEY=<clé API Anthropic>
LLM_MODEL=claude-sonnet-5

EMBEDDING_API_KEY=<clé API OpenAI>
EMBEDDING_MODEL=text-embedding-3-small

ADMIN_BASIC_AUTH_USER=admin
ADMIN_BASIC_AUTH_PASSWORD=<mot de passe fort, généré>
```

   Ne jamais commiter ces valeurs dans le dépôt — elles restent dans la configuration Coolify.

3. Configurer le domaine du service `n8n` sur `n8n.geniuspay.ci` (ou `wiki.geniuspay.ci` si un seul domaine est utilisé) dans Coolify, avec TLS automatique (Let's Encrypt) — Coolify gère le reverse proxy, aucune modification du `docker-compose.yml` n'est nécessaire pour ça.
4. Déployer (*Deploy*). Coolify construit et démarre les deux services : `wiki-db` (Postgres+pgvector) et `wiki-n8n`.

## 3. Vérifier la base de données

Le conteneur `wiki-db` applique automatiquement `database/migrations/001_create_wiki_schema.sql` au tout premier démarrage (dossier monté sur `/docker-entrypoint-initdb.d`). Pour vérifier :

```bash
docker exec -it wiki-db psql -U geniuspay -d geniuspay_wiki -c "\dt"
```

Tu dois voir les tables `documents`, `chunks`, `static_cache`. Pour confirmer que `pgvector` est actif :

```bash
docker exec -it wiki-db psql -U geniuspay -d geniuspay_wiki -c "\dx"
```

Si la migration n'a pas tourné (volume déjà initialisé auparavant), l'appliquer manuellement :

```bash
docker exec -i wiki-db psql -U geniuspay -d geniuspay_wiki < database/migrations/001_create_wiki_schema.sql
```

## 4. Créer les credentials dans n8n

Ouvrir `https://n8n.geniuspay.ci` (connexion avec `ADMIN_BASIC_AUTH_USER` / `ADMIN_BASIC_AUTH_PASSWORD`), puis dans **Credentials → Add credential**, créer :

| Nom exact | Type | Valeurs |
|---|---|---|
| `Wiki DB (Postgres)` | Postgres | Host `wiki-db`, Port `5432`, Database `geniuspay_wiki`, User `geniuspay`, Password = `DB_PASSWORD` |
| `OpenAI Embeddings` | Header Auth | Header `Authorization`, Value `Bearer <EMBEDDING_API_KEY>` |
| `Anthropic API` | Header Auth | Header `x-api-key`, Value `<ANTHROPIC_API_KEY>` |
| `Backoffice Wiki` | Basic Auth | User = `ADMIN_BASIC_AUTH_USER`, Password = `ADMIN_BASIC_AUTH_PASSWORD` |

Ces noms correspondent à ceux référencés dans les 4 fichiers `workflows/*.json` — à l'import, n8n te demandera de réassigner chaque credential (les IDs internes du JSON ne matchent pas ceux de ton instance, c'est normal).

## 5. Importer et activer les workflows

Dans l'éditeur n8n, **Workflows → Import from File**, importer dans cet ordre :

1. `workflows/ingestion.json`
2. `workflows/chat-public.json`
3. `workflows/admin-backoffice.json`
4. `workflows/static-cache-rebuild.json`

Pour chaque workflow importé :

1. Ouvrir chaque node Postgres / HTTP Request / Webhook marqué d'une icône d'erreur de credential, et assigner la credential correspondante créée à l'étape 4.
2. Lire la sticky note en haut à gauche du canvas — elle rappelle les réglages spécifiques à ce workflow.
3. Cliquer sur **Active** (toggle en haut à droite) pour l'activer.

Une fois les 4 workflows actifs, n8n expose ces routes (préfixées par `N8N_WEBHOOK_URL`, soit `https://wiki.geniuspay.ci/webhook`) :

| Route | Workflow |
|---|---|
| `POST /webhook/admin/upload` | ingestion |
| `POST /webhook/chat` (ou l'URL du widget Chat Trigger) | chat-public |
| `GET /webhook/admin/documents` | admin-backoffice |
| `POST /webhook/admin/reindex` | admin-backoffice |
| `POST /webhook/admin/rebuild-cache` | static-cache-rebuild |

## 6. Routage Coolify/Cloudflare

Si un seul domaine `wiki.geniuspay.ci` sert à la fois le chat public et les webhooks admin, aucune config supplémentaire n'est nécessaire : Coolify route déjà tout le trafic HTTPS du domaine vers le conteneur `wiki-n8n` sur le port `5678`, et n8n route en interne selon le path (`/webhook/...`).

Pour l'éditeur n8n (`n8n.geniuspay.ci`), s'assurer que Coolify pointe ce second domaine vers le même conteneur `wiki-n8n`.

## 7. Tests bout en bout

**a. Ingestion d'un document de test**

Via le formulaire backoffice : `https://wiki.geniuspay.ci/webhook/admin/upload` n'est pas destiné à un accès navigateur direct — utiliser plutôt le **Form Trigger** de `admin-backoffice.json`, dont l'URL de test/production est visible dans le node "Formulaire nouveau document" de n8n (bouton *Test URL* / *Production URL*).

Remplir : Titre, Catégorie, Statut (`dynamique` ou `fige`), Fichier (PDF de test), Auteur → Envoyer.

**b. Vérifier l'indexation**

```bash
docker exec -it wiki-db psql -U geniuspay -d geniuspay_wiki \
  -c "SELECT title, category, status FROM documents ORDER BY created_at DESC LIMIT 5;"
docker exec -it wiki-db psql -U geniuspay -d geniuspay_wiki \
  -c "SELECT count(*) FROM chunks;"
```

**c. Interroger le chat public**

Ouvrir `https://wiki.geniuspay.ci` (ou l'URL du widget Chat Trigger) et poser une question portant sur le document de test. La réponse doit citer le titre du document en source.

**d. Lister les documents (backoffice)**

```bash
curl -u "$ADMIN_BASIC_AUTH_USER:$ADMIN_BASIC_AUTH_PASSWORD" \
  https://wiki.geniuspay.ci/webhook/admin/documents
```

**e. Cache figé** (si un document a été marqué `fige`)

```bash
curl -u "$ADMIN_BASIC_AUTH_USER:$ADMIN_BASIC_AUTH_PASSWORD" \
  -X POST https://wiki.geniuspay.ci/webhook/admin/rebuild-cache
docker exec -it wiki-db psql -U geniuspay -d geniuspay_wiki \
  -c "SELECT category, regenerated_at FROM static_cache;"
```

## 8. Sécuriser le backoffice

- Basic Auth est déjà en place sur les 3 routes `/admin/*` (credential `Backoffice Wiki`).
- En plus, créer une **Cloudflare Access Policy** restreignant `wiki.geniuspay.ci/webhook/admin/*` (et `n8n.geniuspay.ci` en entier) aux emails de l'équipe GeniusPay — Cloudflare Zero Trust → Access → Applications.
- Le chat public (`/webhook/chat` et le widget) reste volontairement hors Access : il doit être accessible sans authentification. Vérifier qu'aucun document sensible n'est indexé (voir `docs/ADR-001-wiki-rag.md`, section Sécurité).

## Dépannage

| Symptôme | Piste |
|---|---|
| Le workflow échoue sur le node Postgres | Vérifier la credential `Wiki DB (Postgres)` et que `wiki-db` est bien joignable depuis le conteneur `wiki-n8n` (même réseau Docker Compose) |
| `403`/`401` sur un webhook admin | Vérifier les identifiants Basic Auth envoyés, et la credential `Backoffice Wiki` dans n8n |
| Embedding en erreur | Vérifier le quota/la validité de `EMBEDDING_API_KEY` (OpenAI) |
| Réponse du chat vide ou générique | Vérifier `ANTHROPIC_API_KEY`, et que la recherche pgvector retourne bien des lignes (voir requête section 7b) |
| Extraction de texte vide | Le node *Extraire le texte* est réglé sur `operation: pdf` par défaut — l'ajuster (`docx`, `text`, ...) selon le format réel du document |
