# Session — Mémoire Centrale ASP — Étapes 1 & 2

**Date :** 2026-06-19
**Projet :** AFTER SUN PEOPLE (ASP) — AI Operating System
**Objectif de la session :** Étape 1 (détection environnement) + Étape 2 (socle pgvector + table mémoire)
**Statut :** ✅ Atteint et vérifié

---

## Ce qui a été fait

**Décision d'architecture validée en amont (avec apport de ChatGPT) :** table mémoire enrichie de 6 colonnes optionnelles (`title`, `collection`, `knowledge_type`, `status`, `content_hash`, `updated_at`) en plus du socle. Coût quasi nul aujourd'hui, évite des migrations plus tard. Reste 100 % MVP (colonnes à valeurs par défaut, qu'on ne remplit que si besoin).

**Étape 1 — Détection.** Création dans n8n d'un workflow manuel + node Postgres « Execute a SQL query », connecté à la base. Résultat :
- PostgreSQL **16.14** (image `postgres:16-alpine`).
- Extension `vector` (pgvector) **non disponible** dans cette image → impossible de créer l'extension en l'état.

**Sauvegarde.** Avant toute modification : déclenchement d'un backup Postgres dans Coolify (en plus du backup quotidien existant). Données préservées tout du long.

**Étape 2 — Socle.**
- Image Postgres Coolify basculée de `postgres:16-alpine` vers **`pgvector/pgvector:pg16`**, puis redémarrage du service. Stack remontée saine (postgresql, n8n, worker, task-runners, redis tous *healthy*). **Données intactes** (62 tables n8n toujours présentes).
- `CREATE EXTENSION vector` → **pgvector 0.8.3** installé.
- Table **`asp_memory`** créée (13 colonnes, version finale).
- Index de recherche sémantique **HNSW** `asp_memory_embedding_idx` (`embedding vector_cosine_ops`).
- Le DDL a été exécuté via le **terminal Postgres de Coolify (psql)** — voir Challenges.

**Vérification finale (`\d asp_memory`) :** 13 colonnes conformes, `embedding vector(1536)`, 3 index (PK `id`, UNIQUE `content_hash`, HNSW `embedding`).

---

## Décisions

- Table mémoire figée à **13 colonnes** (socle + 6 ajouts). Aucune retirée.
- Image Postgres = **`pgvector/pgvector:pg16`** (Postgres 16, pgvector inclus).
- Modèle d'embedding figé : **`text-embedding-3-small`** (1536 dimensions) — déjà reflété par `VECTOR(1536)`.
- Numérotation des étapes : **commence à 1**, plus jamais 0.

---

## Challenges & solutions

1. **Identifiants Postgres.** n8n et Postgres sont dans la même stack Coolify (`n8n-with-postgres-and-worker`) → même réseau Docker, le host interne est simplement `postgresql`. Les identifiants sont les variables d'env de n8n (n8n utilise ce Postgres comme sa propre base).
2. **Superuser réel = `CgLLcKfWAN2pIcZd`** (avec un **I majuscule**, pas un `l`). Le rôle `postgres` n'existe pas. → dans le conteneur, toujours utiliser `psql -U "$POSTGRES_USER" -d "$POSTGRES_DB"`.
3. **Changement d'image = sensible** car c'est aussi la BDD de n8n (musl Alpine → glibc Debian). Mitigation : backup d'abord ; au final aucun souci au démarrage, données intactes.
4. **Node Postgres de n8n bloqué après le redémarrage.** Après la bascule, le node « Execute a SQL query » reste en exécution sans répondre (pool de connexions n8n resté accroché à l'ancien conteneur). Contournement : DDL exécuté directement via le terminal Postgres de Coolify (psql). **À régler avant l'Étape 3.**

---

## Lessons learned

- Quand n8n et Postgres partagent une stack Coolify, le host interne est le **nom du service** (`postgresql`), pas une IP/URL publique.
- Pour lire un identifiant sans risque d'erreur (I/l, O/0), **utiliser les variables d'environnement** (`$POSTGRES_USER`) plutôt que recopier à l'œil.
- Toujours **prendre un backup avant** un changement d'image sur une base de production.
- pgvector n'est **pas** dans les images Postgres standard → image dédiée `pgvector/pgvector`.

---

## Points critiques à valider / arbitrer (CEO)

1. **Redémarrage du conteneur n8n** pour rafraîchir son pool de connexions (corrige le node Postgres bloqué). Courte coupure n8n. À faire au début de l'Étape 3.
2. Confirmer qu'on **garde `asp_memory` dans la base `n8n`** (même base que n8n) pour le MVP, plutôt qu'une base dédiée. (Recommandé : oui pour le MVP.)

---

## Incident n8n — ✅ RÉSOLU

Pendant la session, après plusieurs redémarrages du stack, **n8n est tombé en crash-loop** sur Redis. **Désormais entièrement réparé** (n8n debout + exécution de workflows OK + node Postgres confirmé `vector_available=true`). État et diagnostic :

**Ce qui est confirmé**
- Redis n'a **aucun mot de passe** (`redis-cli PING` sans auth = `PONG` ; `redis-server` lancé sans `--requirepass`). Le compose n8n ne définit **aucun** `QUEUE_BULL_REDIS_PASSWORD`. Le worker était *healthy*.
- Crash fatal initial de l'instance principale : `Failed to subscribe to channel n8n:n8n.commands → NOAUTH Authentication required` → `Exiting due to an error` (en boucle).

**Correctif partiel appliqué (réversible)**
- Retiré la ligne dépréciée `- N8N_RUNNERS_ENABLED=true` des services `n8n` **et** `n8n-worker` dans le compose (Edit Compose File), puis **Stop → Deploy**.
- Résultat : **n8n ne crashe plus, l'interface est de nouveau accessible** (https://n8n.aftersunpeople.com). ✅

**Cause racine trouvée (confirmée)**
- `getent hosts redis` renvoyait une IP (`fde4:75c5:b9d4::2`) **différente** de celle du conteneur Redis du stack (`...::a`). Le hostname générique `redis` résolvait vers un **autre Redis** (protégé par mot de passe) du réseau partagé Coolify → d'où `NOAUTH` puis `ERR Protocol error: unauthenticated multibulk length`. (`postgresql` ne posait pas ce souci car ce nom est unique sur le réseau.)

**Correctif final (appliqué, ça marche)**
- Dans le compose, dans les services `n8n` **et** `n8n-worker` : `QUEUE_BULL_REDIS_HOST=redis` → `QUEUE_BULL_REDIS_HOST=redis-m10y6cwbuxsc5tt6hkfizh54` (nom unique du conteneur). Save → Stop → Deploy.
- Résultat : exécution de workflow OK, node Postgres renvoie `vector_available=true`. ✅

**Important :** l'Étape 2 (pgvector + table + index) a été réalisée **directement via le terminal psql de Coolify**, donc elle est **acquise indépendamment** de ce bug n8n. Les **données sont intactes** (backup pris + 62 tables présentes).

---

## Prochaines étapes

- **Étape 3 — Ingestion** (n8n) : texte → chunks (~500–1000 tokens) → embedding OpenAI `text-embedding-3-small` (node HTTP Request) → insert dans `asp_memory`. Vecteur au format chaîne `'[...]'`. Finir l'insertion par `ON CONFLICT (content_hash) DO NOTHING` (anti-doublon). Tester avec 1 document.
- **Étape 4 — Récupération** : question → embedding → recherche cosinus (`ORDER BY embedding <=> $1 LIMIT 5`).
- **Étape 5 — Notion** : branchement source d'écriture (sens unique) + import progressif par petits lots triés.
