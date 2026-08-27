# ADR · Wiki interne RAG

## Statut
Accepté — 27 août 2026

## Contexte

GENIUS GROUPS a besoin d'une base de connaissance interne couvrant l'ensemble des dossiers (projets, conditionnalités, présentations, procédures), consultable :
- en interne, par toute l'équipe, via un backoffice
- publiquement, via un chat accessible sur `wiki.geniuspay.ci`

Une partie du corpus est stable (procédures, chartes validées), une autre évolue en continu (projets en cours, conditionnalités mises à jour). La stack existante repose sur n8n (automatisation), Docker/Coolify (déploiement) et Cloudflare (edge/DNS).

## Décision

**1. n8n comme orchestrateur unique**, plutôt qu'un backend applicatif dédié (Laravel/Filament).
Raison : réutilise l'expertise et l'infra déjà en place, évite de dupliquer une couche backend pour un besoin encore en phase de validation.

**2. PostgreSQL + pgvector comme unique magasin de données**, plutôt qu'une base vectorielle managée externe (Pinecone, Weaviate).
Raison : cohérent avec le reste de l'écosystème GeniusPay (déjà sur Postgres), pas de dépendance à un service tiers payant, hébergement souverain sur le serveur GeniusPay.

**3. Architecture hybride RAG + cache figé**, plutôt qu'un RAG pur.
Raison : les articles marqués `stable` (procédures, chartes) sont préchargés en contexte pour une latence minimale ; les articles `dynamique` (projets, conditionnalités) passent par la recherche vectorielle classique pour rester à jour.

**4. Deux points d'entrée n8n distincts** (`Chat Trigger` public, `Form Trigger` + webhooks pour le backoffice), plutôt qu'un frontend custom.
Raison : n8n fournit nativement un widget de chat embarquable et des formulaires — évite de construire et maintenir une interface web dédiée dans un premier temps.

## Alternatives considérées

| Option | Rejetée car |
|---|---|
| Backend Laravel/Filament dédié | Duplique une couche déjà couverte par n8n ; plus lourd à maintenir pour du CRUD simple |
| Base vectorielle managée (Pinecone) | Dépendance externe payante, hébergement hors du serveur GeniusPay |
| CAG pur (tout en cache) | Le corpus contient une part significative de contenu évolutif — un cache figé complet deviendrait obsolète |
| RAG pur (rien en cache) | Latence inutile sur les contenus qui ne changent jamais |

## Conséquences

**Positives**
- Réutilisation à 100 % de l'infra existante (n8n, Docker, Coolify, Cloudflare, Postgres)
- Latence minimale sur le contenu stable, fraîcheur garantie sur le contenu dynamique
- Pas de coût de service tiers pour la base vectorielle

**Négatives / points de vigilance**
- Le backoffice via Form Trigger reste limité en confort de gestion (pas de vraie UI de liste/recherche) — migration vers Filament à prévoir si le volume de contenu grandit
- La classification `figé` / `dynamique` est déclarative (choisie à l'ingestion) — nécessite une discipline de l'équipe pour rester pertinente
- Le chat public doit être strictement séparé du contenu sensible — aucune donnée confidentielle ne doit être indexée dans le corpus accessible sans authentification
