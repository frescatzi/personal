---
type: raw
title: "Memoire_Centrale_ASP_Brief_Construction"
source_url: "drive:1rSEwlcRSRKcyzMnB60r23to-HJmftkuZ"
captured: 2026-07-07
vault: ai-automation
brand: null
immutable: true
---

---
type: wiki
title: "Mémoire Centrale ASP — Brief de construction (MVP)"
status: active
publish: notion
vault: ai-automation
brand: null
sources: []
related: ["synthese-lumina-systeme-reference", "sop/SOP_systeme-multi-agents-memoire-centrale-mcp-n8n", "architecture/Plan_Demarrage_Memoire_Centrale_MVP", "architecture/Architecture_Connaissance_Obsidian_Centric"]
updated: 2026-06-19
---

# Mémoire Centrale ASP — Brief de construction (MVP)

> **Document autonome.** À utiliser dans une nouvelle discussion Cowork dédiée à la construction de la mémoire. Tout le contexte nécessaire est dans ce fichier — pas besoin d'avoir suivi les échanges précédents.

**Date :** 2026-06-19
**Projet :** AFTER SUN PEOPLE (ASP) — AI Operating System
**Objectif :** créer **une mémoire centrale unique sur le serveur**, interrogeable à la demande par n8n et les agents, vers laquelle on migrera progressivement la connaissance qui vit aujourd'hui dans Claude et ChatGPT.

---

## 1. Contexte (à lire en premier)

- **Qui :** Karter (alias « Kater » / « Katel » = la même personne, le CEO). **Débutant sur n8n**, en montée en compétence sur l'outillage dev (Git, CLI, Node.js).
- **Préférences :** explications **concrètes et conversationnelles**, en **français**, sans complexité inutile. Approche **learn-by-doing** : on construit petit, on apprend en faisant, on a le droit de casser des choses.
- **Méthode :** **MVP d'abord**, on itère ensuite. On ne vise pas le parfait. Les agents observeront plus tard et proposeront les améliorations / workflows spécialisés.
- **Cadrage de la 1ʳᵉ session :** viser **Étape 0 (détection) + Étape 1 (activer pgvector + créer la table)**. Le reste s'enchaîne sur les sessions suivantes. Ne pas tout faire d'un coup.

---

## 2. Stack réelle

- **Serveur :** Hetzner VPS, géré via **Coolify**.
- **n8n** auto-hébergé : `n8n.aftersunpeople.com` (couche d'orchestration / exécution).
- **PostgreSQL** : déjà présent sur le serveur (via Coolify).
- **Plateformes IA :** ChatGPT, Claude, Gemini.
- **Notion** : banque de connaissances actuelle. **GitHub** : versioning technique.

**⚠️ Piège connu — node Anthropic dans n8n :** le node **Anthropic natif renvoie une 404** car n8n a codé en dur un modèle Haiku retiré (`claude-3-haiku-20240307`). **Ne pas l'utiliser.** Pour appeler Claude, passer par un node **HTTP Request** vers `https://api.anthropic.com/v1/messages` avec un modèle à jour. *(Pour la mémoire, Anthropic ne fournit de toute façon pas d'embeddings — voir §4.)*

---

## 3. Décisions figées (ne pas re-débattre)

1. **Store mémoire = `pgvector` sur le PostgreSQL existant.** Aucun nouveau service, aucun coût ajouté. C'est ce que n8n et les agents interrogent.
2. **Notion = base centrale d'écriture et référence humaine (« fait foi »).** Circulation **à sens unique** : on écrit dans Notion → n8n **pousse vers Postgres**. **Postgres n'est jamais édité à la main.** Pas de synchro bidirectionnelle (source de bugs).
3. **Un seul modèle d'embedding, figé dès le départ :** `text-embedding-3-small` (OpenAI, 1536 dimensions). En changer plus tard oblige à **tout ré-encoder**.
4. **Démarrer avec un petit lot trié** (les docs de ce projet + 2-3 SOP clés), prouver que la recherche marche, *puis* élargir. **Pas de migration massive d'un coup** (une mémoire en vrac = des réponses en vrac).
5. **Pas de cycle de vie formel** des connaissances au stade MVP.
6. **Validation :** autonomie par défaut aux agents ; les tâches critiques (coûts, sécurité, accès API, suppression de mémoire) passent par une **validation CEO** ; le reste, le Superviseur valide ou l'agent agit seul.

> Note : « SOP » = *Standard Operating Procedure* = procédure écrite étape par étape pour une tâche récurrente. Dans ce système, une SOP = un **How-To générique réutilisable** (sans mention de marque, suivable par un débutant, avec les pièges intégrés).

---

## 4. Architecture cible (MVP)

```text
   ┌────────────────────────┐
   │  NOTION                 │  ← base centrale : on écrit / collecte ici
   │  (référence humaine,    │     « fait foi »
   │   point d'autorité)     │
   └───────────┬─────────────┘
               │  (sens unique)
               ▼
   ┌────────────────────────┐
   │  INGESTION (n8n)        │   texte → découpage (chunks) →
   │                         │   embedding (OpenAI) → insertion
   └───────────┬─────────────┘
               ▼
   ┌────────────────────────┐
   │  PostgreSQL + pgvector  │  ← mémoire serveur (jamais éditée à la main)
   │  table asp_memory       │     interrogée par n8n et les agents
   └───────────┬─────────────┘
               ▼
   ┌────────────────────────┐
   │  RÉCUPÉRATION (n8n)     │   question → embedding →
   │                         │   recherche sémantique → réponse aux agents
   └────────────────────────┘
```

**En amont (hors périmètre de ce build) :** le CEO réfléchit dans **Obsidian** (espace perso). Rien n'entre dans Notion ni Postgres tant que ce n'est pas validé. **Les agents et l'ingestion ne lisent jamais Obsidian** → ça ne change rien à la construction de la mémoire ci-dessous.

**Embeddings :** via node **HTTP Request** → `POST https://api.openai.com/v1/embeddings`, body `{"model":"text-embedding-3-small","input":"..."}`. Le vecteur est dans `data[0].embedding` (1536 nombres).

---

## 5. Roadmap (du plus simple au plus complet)

### Étape 0 — Détection de l'environnement (1ʳᵉ session)
But : savoir sur quoi on bâtit (et premier contact n8n + Postgres).
1. Dans n8n : nouveau workflow → node **Postgres (Execute Query)** → connecter à la base (identifiants Coolify : host, port, db, user, password).
2. Exécuter :
   ```sql
   SELECT version();
   SELECT * FROM pg_available_extensions WHERE name = 'vector';
   ```
3. Lire : version de PostgreSQL + si l'extension `vector` est disponible / déjà installée (`installed_version`).
4. **Si `vector` est absent de l'image** : basculer la base Coolify sur l'image `pgvector/pgvector` (Claude guidera). Ne pas paniquer.

### Étape 1 — Socle (1ʳᵉ session)
1. Activer l'extension :
   ```sql
   CREATE EXTENSION IF NOT EXISTS vector;
   ```
2. Créer la table mémoire :
   ```sql
   CREATE TABLE IF NOT EXISTS asp_memory (
     id          BIGSERIAL PRIMARY KEY,
     content     TEXT NOT NULL,
     source      TEXT,              -- 'notion' | 'chatgpt' | 'claude' | 'manual'
     source_ref  TEXT,             -- id/url de la page Notion d'origine
     metadata    JSONB DEFAULT '{}'::jsonb,
     embedding   VECTOR(1536),     -- text-embedding-3-small
     created_at  TIMESTAMPTZ DEFAULT now()
   );
   ```
3. Index de recherche sémantique (cosinus) :
   ```sql
   CREATE INDEX IF NOT EXISTS asp_memory_embedding_idx
     ON asp_memory USING hnsw (embedding vector_cosine_ops);
   ```
   *(Si la version de pgvector ne supporte pas HNSW, utiliser `ivfflat` — Claude adaptera.)*

### Étape 2 — Workflow d'ingestion (session suivante)
Workflow n8n : **déclencheur manuel** → récupérer un texte → **découper en chunks** (Code node, ~500-1000 tokens, léger chevauchement) → **HTTP Request** embeddings OpenAI → **Postgres Insert** dans `asp_memory`.
- ⚠️ Le vecteur doit être inséré au **format pgvector** : une chaîne `'[0.0123, -0.045, ...]'`.
- Tester avec **un seul document** d'abord.

### Étape 3 — Workflow de récupération (session suivante)
Workflow n8n : **input = une question** → embedding de la question → requête de similarité :
```sql
SELECT content, source, 1 - (embedding <=> $1) AS similarity
FROM asp_memory
ORDER BY embedding <=> $1
LIMIT 5;
```
→ renvoyer les passages les plus pertinents (à exposer ensuite aux agents).

### Étape 4 — Doublon Notion + import progressif (sessions suivantes)
- Brancher Notion comme source d'écriture ; n8n lit Notion → ingère (sens unique).
- **Migrer par petits lots triés** la connaissance Claude / ChatGPT, en vérifiant la qualité de récupération à chaque ajout.

---

## 6. Pièges à garder en tête

- **pgvector pas dans l'image par défaut** → image `pgvector/pgvector`.
- **Dimension figée à 1536** : ne pas mélanger des embeddings de modèles différents dans la même table.
- **Format du vecteur** à l'insertion : chaîne `'[...]'`, pas un tableau JSON brut.
- **Jamais éditer Postgres à la main** : la vérité humaine est dans Notion.
- **Node Anthropic interdit** (bug 404) ; embeddings via OpenAI (ou Gemini), pas Anthropic.
- **Coûts** : `text-embedding-3-small` est bon marché, mais ne ré-encode pas tout en boucle pour rien.

---

## 7. À livrer en fin de session (conventions ASP)

1. Un **MD de session** (ce qui a été fait, décisions, challenges, lessons learned, prochaines étapes).
2. Un **How-To / SOP générique** si une étape est répétable ailleurs (sans marque, pas-à-pas, pièges inclus) — ex. « Installer pgvector sur un Postgres Coolify », « Créer un workflow d'ingestion mémoire dans n8n ».
3. Les **points critiques à valider** par le CEO.

---

## Version
v1.0 — Brief de construction Mémoire Centrale ASP — 2026-06-19
