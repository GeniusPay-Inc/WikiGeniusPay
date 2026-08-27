# GeniusPay · Wiki interne RAG

**Base de connaissance interne pilotée par n8n, exposée en public via chat et en interne via backoffice**

![n8n](https://img.shields.io/badge/n8n-orchestration-EA4B71) ![PostgreSQL](https://img.shields.io/badge/PostgreSQL-16-336791) ![pgvector](https://img.shields.io/badge/pgvector-recherche%20vectorielle-1D9E75) ![Docker](https://img.shields.io/badge/Docker-Coolify-2496ED) ![Licence](https://img.shields.io/badge/licence-propri%C3%A9taire-lightgrey)

Un microservice de connaissance de l'écosystème **GENIUS GROUPS SAS** — exploité par **GeniusPay, INC.**

Source unique de vérité pour les procédures, dossiers projets, conditionnalités et présentations internes, interrogeable en langage naturel par les collaborateurs (backoffice) et par le grand public (chat public).

🌐 wiki.geniuspay.ci · 🔧 n8n Editor (interne) · 📖 Workflows API

## Table des matières

- [Vue d'ensemble](#vue-densemble)
- [Architecture](#architecture)
- [Modules fonctionnels](#modules-fonctionnels)
- [Contrats webhooks n8n](#contrats-webhooks-n8n)
- [Stack technique](#stack-technique)
- [Démarrage rapide](#démarrage-rapide)
- [Configuration](#configuration)
- [Structure du projet](#structure-du-projet)
- [Sécurité & accès](#sécurité--accès)
- [Écosystème GeniusPay](#écosystème-geniuspay)
- [Licence](#licence)

## Vue d'ensemble

Le Wiki interne RAG centralise la connaissance de GENIUS GROUPS aujourd'hui dispersée entre dossiers, présentations et échanges internes. Il combine deux approches :

| Défi | Solution Wiki RAG |
|---|---|
| Connaissance dispersée entre dossiers/canaux | Base vectorielle unique, alimentée en continu |
| Contenu figé (procédures, chartes) recherché à chaque fois | Cache précompilé pour les articles marqués `stable` |
| Contenu évolutif (projets, conditionnalités) vite obsolète | Pipeline d'ingestion n8n déclenché à chaque mise à jour |
| Pas d'accès public à la connaissance non sensible | Chat public exposé sur wiki.geniuspay.ci |
| Pas de gestion structurée des dossiers | Backoffice avec statut, catégorie et droits par contenu |

## Architecture

Architecture orchestrée entièrement par **n8n**, avec **PostgreSQL + pgvector** comme unique magasin de données :

```
                    ┌──────────────────────────────────────────────────────────────────┐
                    │                     SERVEUR GENIUSPAY (Coolify)                   │
                    │                    wiki.geniuspay.ci (Cloudflare)                 │
                    │                                                                    │
                    │  ┌──────────────────┐              ┌──────────────────────────┐  │
                    │  │   Backoffice      │              │        Public            │  │
                    │  │  (Form Trigger)   │              │     (Chat Trigger)        │  │
                    │  └────────┬──────────┘              └────────────┬──────────────┘  │
                    │           │                                       │                 │
                    │  ┌────────▼───────────────────────────────────────▼──────────────┐  │
                    │  │                         N8N — ORCHESTRATEUR                    │  │
                    │  │   Workflow Ingestion · Workflow Requête · Workflow Admin       │  │
                    │  └────────────────────────────┬────────────────────────────────┘  │
                    │                                │ embeddings / recherche             │
                    │  ┌─────────────────────────────▼────────────────────────────────┐  │
                    │  │                POSTGRESQL 16 + PGVECTOR                       │  │
                    │  │   documents · chunks · embeddings · statut (figé/dynamique)   │  │
                    │  └────────────────────────────────────────────────────────────────┘  │
                    └────────────────────────────────────────────────────────────────────┘
```

## Modules fonctionnels

**1. Ingestion**
Responsabilité : surveillance d'une source (dossier, Drive), extraction du texte, découpage en chunks, génération d'embeddings, écriture en base.
Déclenchement : automatique à l'ajout/modification d'un document, ou manuel depuis le backoffice.

**2. Requête publique (Chat)**
Responsabilité : réception d'une question en langage naturel, recherche par similarité vectorielle, injection du contexte au LLM, réponse sourcée.
Accès : ouvert à tous via wiki.geniuspay.ci, sans authentification.

**3. Administration (Backoffice)**
Responsabilité : upload de documents, attribution catégorie et statut (`figé` / `dynamique`), déclenchement manuel de réindexation.
Accès : restreint (Basic Auth ou Cloudflare Access).

**4. Cache figé**
Responsabilité : précompilation du contexte pour les contenus stables (procédures, chartes), régénérée uniquement à la validation d'une nouvelle version — latence minimale, pas de recherche vectorielle nécessaire.

## Contrats webhooks n8n

| Méthode | Route | Description | Consommateur |
|---|---|---|---|
| POST | `/webhook/chat` | Reçoit une question et renvoie une réponse sourcée | Widget chat public |
| POST | `/webhook/admin/upload` | Ingestion d'un nouveau document + métadonnées | Backoffice |
| POST | `/webhook/admin/reindex` | Déclenche une réindexation manuelle d'un document | Backoffice |
| GET | `/webhook/admin/documents` | Liste des documents avec statut et catégorie | Backoffice |

## Stack technique

| Technologie | Rôle |
|---|---|
| n8n | Orchestrateur — ingestion, requêtes, administration |
| PostgreSQL 16 | Base de données |
| pgvector | Extension de recherche vectorielle |
| Docker / Coolify | Conteneurisation et déploiement |
| Cloudflare | DNS, accès restreint au backoffice |

## Démarrage rapide

### Prérequis
- Docker & Docker Compose
- PostgreSQL 16+ avec extension `pgvector`
- Instance n8n (self-hosted)

### Installation

```bash
# 1. Cloner le projet
git clone https://github.com/GeniusGroups/geniuspay-wiki-rag.git
cd geniuspay-wiki-rag

# 2. Configuration
cp .env.example .env

# 3. Lancer les conteneurs
docker compose up -d

# 4. Activer l'extension pgvector
docker exec -it wiki-db psql -U geniuspay -d geniuspay_wiki -c "CREATE EXTENSION IF NOT EXISTS vector;"

# 5. Importer les workflows n8n
# Depuis l'éditeur n8n : Import from File > workflows/*.json
```

## Configuration

Variables d'environnement clés :

```env
# Application
APP_NAME="GeniusPay Wiki"
APP_URL=https://wiki.geniuspay.ci

# Base de données PostgreSQL
DB_HOST=127.0.0.1
DB_PORT=5432
DB_DATABASE=geniuspay_wiki
DB_USERNAME=geniuspay
DB_PASSWORD=secret

# n8n
N8N_HOST=n8n.geniuspay.ci
N8N_WEBHOOK_URL=https://wiki.geniuspay.ci/webhook

# LLM
LLM_API_KEY=your-api-key
LLM_MODEL=claude-sonnet-5

# Backoffice
ADMIN_BASIC_AUTH_USER=admin
ADMIN_BASIC_AUTH_PASSWORD=secret
```

## Structure du projet

```
geniuspay-wiki-rag/
├── workflows/                   # Exports JSON des workflows n8n
│   ├── ingestion.json
│   ├── chat-public.json
│   └── admin-backoffice.json
├── database/
│   └── migrations/               # Schéma Postgres (documents, chunks, embeddings)
├── docker-compose.yml
├── .env.example
└── README.md
```

## Sécurité & accès

- **Chat public** : accès libre, aucune donnée sensible ne doit être indexée dans le corpus public.
- **Backoffice** : protégé par Basic Auth au niveau du webhook n8n, doublé d'une policy Cloudflare Access restreinte aux emails autorisés de l'équipe.
- **Documents sources** : stockage d'origine hors du dépôt (Drive ou espace interne dédié) — le dépôt ne contient que les workflows et le schéma de base.

## Écosystème GeniusPay

| Service | Stack | Rôle |
|---|---|---|
| Wiki interne RAG (ce dépôt) | n8n + PostgreSQL/pgvector | Base de connaissance interne et publique |
| GeniusPay Merchant Core | Laravel 13 + PHP 8.3 | Identité marchande, Apps, KYB, KYC, Team |
| GeniusPay Core | Laravel 12 + PHP 8.3 | Paiement, passerelle, routage |
| GeniusWallet | React + PWA | Wallet client final |
| GeniusPartners | Laravel 12 + React | Commissions et partenaires |

## Licence

Propriétaire — © 2024-2026 GENIUS GROUPS SAS. Tous droits réservés.

Développé et exploité par GeniusPay, INC., filiale de GENIUS GROUPS SAS.

**GeniusPay Wiki — la connaissance de l'écosystème, interrogeable en langage naturel.**

Fabriqué à Abidjan, Côte d'Ivoire 🇨🇮
