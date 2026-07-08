---
type: wiki
title: "SOP — Installer pgvector sur un Postgres géré par Coolify"
status: active
publish: notion
vault: ai-automation
brand: null
sources:
  - "raw/2026-06-24--sop-installer-pgvector-sur-postgres-coolify.md"
  - "raw/2026-06-29--sop-installer-pgvector-sur-postgres-coolify.md"
related:
  - "sop/SOP_systeme-multi-agents-memoire-centrale-mcp-n8n"
  - "sop/sop-creer-memoire-agents-humains"
  - "synthese-lumina-systeme-reference"
updated: 2026-06-29
---

# SOP — Installer pgvector sur un Postgres géré par Coolify

**But :** activer l'extension `pgvector` sur une base PostgreSQL déjà déployée via Coolify, quand l'image standard ne la contient pas, puis créer une table vectorielle et son index de recherche.
**Niveau :** débutant. **Durée :** ~15–20 min (dont une courte coupure de la base).
**Pré-requis :** accès admin à Coolify ; la base tourne déjà.

> ⚠️ Si cette base est aussi utilisée par une application (ex. n8n, un back-office), changer son image **redémarre la base** → courte coupure de l'app. Prévenir et choisir un créneau calme.

---

## 1. Vérifier si pgvector est disponible

Ouvrir un accès SQL à la base (terminal du conteneur, ou un client SQL). Exécuter :

```sql
SELECT version();
SELECT EXISTS (SELECT 1 FROM pg_available_extensions WHERE name = 'vector') AS vector_available;
```

- `vector_available = true` → pgvector est déjà dans l'image. **Sauter à l'étape 4.**
- `vector_available = false` → continuer.

Noter la **version majeure** de PostgreSQL (ex. 16) : elle détermine le tag de l'image.

## 2. Sauvegarder AVANT toute modification

Dans Coolify → la ressource Postgres (ou le service) → onglet **Backups** → **Backup Now**. Attendre la confirmation. Ne jamais changer l'image sans backup valide récent.

## 3. Basculer l'image vers pgvector et redémarrer

1. Coolify → service Postgres → **General** → champ **Image**.
2. Remplacer l'image standard par l'image pgvector correspondant à la version majeure :
   - PostgreSQL 16 → `pgvector/pgvector:pg16`
   - PostgreSQL 15 → `pgvector/pgvector:pg15` (adapter au besoin)
3. **Save**, puis **Restart** (redéploiement).
4. Attendre que tous les conteneurs repassent **healthy** dans les logs de démarrage.

> Les données restent sur le volume : le cluster existant est conservé (pas de réinitialisation). Vérifier quand même à l'étape 4 que les tables attendues sont là.

## 4. Activer l'extension et vérifier

```sql
CREATE EXTENSION IF NOT EXISTS vector;
SELECT extname, extversion FROM pg_extension WHERE extname = 'vector';
```

→ doit renvoyer une ligne `vector | <version>` (ex. 0.8.x).

## 5. Créer une table vectorielle + index de recherche

Adapter le nom de table et la dimension du vecteur au modèle d'embedding utilisé (ex. 1536 pour `text-embedding-3-small`) :

```sql
CREATE TABLE IF NOT EXISTS ma_table (
    id         BIGSERIAL PRIMARY KEY,
    content    TEXT NOT NULL,
    embedding  VECTOR(1536),
    created_at TIMESTAMPTZ DEFAULT now()
);

CREATE INDEX IF NOT EXISTS ma_table_embedding_idx
  ON ma_table USING hnsw (embedding vector_cosine_ops);
```

> Si la version de pgvector ne supporte pas HNSW (< 0.5.0), utiliser `ivfflat` à la place.

## 6. Vérification finale

```sql
\d ma_table
```

→ contrôler les colonnes, `embedding vector(1536)`, et la présence de l'index HNSW.

---

## Pièges connus

- **pgvector absent des images standard** → toujours l'image dédiée `pgvector/pgvector:pgXX`.
- **Faire correspondre le tag** `pgXX` à la version majeure réelle de la base.
- **Backup obligatoire** avant le changement d'image (surtout si une app dépend de cette base).
- **Identifiants** : dans le conteneur, le superuser est la valeur de `$POSTGRES_USER` (le rôle `postgres` peut ne pas exister). Utiliser `psql -U "$POSTGRES_USER" -d "$POSTGRES_DB"` pour éviter les erreurs de saisie (I/l, O/0).
- **Dimension du vecteur figée** : ne pas mélanger des embeddings de modèles différents dans une même colonne `VECTOR(n)`.
- **Coupure** : changer l'image redémarre la base → courte indisponibilité de toute app qui l'utilise.
