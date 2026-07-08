---
type: raw
title: "SOP_systeme-multi-agents-memoire-centrale-mcp-n8n"
source_url: "drive:1W8TRDpd0_5PWO2b-r-cP3YtEN1sVBsYf"
captured: 2026-07-07
vault: ai-automation
brand: null
immutable: true
---

---
type: wiki
title: "SOP — Système multi-agents avec mémoire centrale & serveur MCP (n8n)"
status: active
publish: notion
vault: ai-automation
brand: null
sources:
  - "raw/2026-06-24--sop-systeme-multi-agents-memoire-centrale-mcp-n8n.md"
  - "raw/2026-06-29--sop-systeme-multi-agents-memoire-centrale-mcp-n8n.md"
related:
  - "synthese-lumina-systeme-reference"
  - "sop/SOP_installer-pgvector-sur-postgres-coolify"
  - "sop/sop-lumina-intake-et-publish"
  - "sop/sop-creer-memoire-agents-humains"
  - "concept-pipeline-memoire-wiki-git"
updated: 2026-06-29
---

# SOP — Système multi-agents avec mémoire centrale & serveur MCP (n8n)

**Type :** Procédure générique réutilisable (template)
**Stack visée :** n8n self-hosted + PostgreSQL/pgvector + OpenAI embeddings + Claude (MCP)
**Convention :** le contenu des champs (system prompts, noms de nodes, descriptions d'outils, valeurs) est en **anglais** ; les explications restent dans ta langue.

> Remplace les `<placeholders>` par tes valeurs : `<DOMAIN>` (ex. n8n.exemple.com), `<TABLE>` (table mémoire), `<WEBHOOK_PATH>`, etc.

---

## Vue d'ensemble

Objectif : **une seule mémoire centrale**, alimentée par tes sources, interrogeable par des agents n8n **et** par Claude (web/Cowork/Code) via un serveur MCP. Topologie agents = **hub-and-spoke** : 1 superviseur qui délègue à N spécialistes.

```
Sources → Ingestion → [Mémoire centrale: Postgres + pgvector]
                              ↑                ↑
                    Webhook de recherche   (même base)
                       ↑           ↑
              Agents n8n        Serveur MCP → Claude
```

---

## Étape 1 — Socle mémoire (pgvector)

1. Image Postgres **avec pgvector** (ex. `pgvector/pgvector:pg16`). **Backup avant** tout changement d'image.
2. `CREATE EXTENSION IF NOT EXISTS vector;`
3. Table mémoire :

```sql
CREATE TABLE <TABLE> (
  id            BIGSERIAL PRIMARY KEY,
  title         TEXT,
  content       TEXT NOT NULL,
  collection    TEXT DEFAULT 'core',
  knowledge_type TEXT,
  status        TEXT DEFAULT 'draft',
  source        TEXT,
  source_ref    TEXT,
  content_hash  TEXT UNIQUE,
  metadata      JSONB,
  embedding     VECTOR(1536),
  created_at    TIMESTAMPTZ DEFAULT now(),
  updated_at    TIMESTAMPTZ DEFAULT now()
);
CREATE INDEX <TABLE>_embedding_idx ON <TABLE> USING hnsw (embedding vector_cosine_ops);
```

> `VECTOR(1536)` = dimensions du modèle `text-embedding-3-small`. `content_hash UNIQUE` = anti-doublon.

---

## Étape 2 — Ingestion

Workflow : **Trigger** → **Set** (champ `document`) → **Chunk** (Code, ~3000 car / 200 overlap) → **Embed** (HTTP Request OpenAI) → **Build Insert** (Code) → **Postgres Execute Query**.

- **Embed (HTTP Request)** : POST `https://api.openai.com/v1/embeddings`, credential `openAiApi`, JSON body :
  ```
  {{ JSON.stringify({ model: 'text-embedding-3-small', input: $json.content }) }}
  ```
- **Build Insert** — mode **Run Once for Each Item**, retourne **un objet** `{ query }` :
  ```js
  const emb = $json.data && $json.data[0] && $json.data[0].embedding;
  if (!emb) { throw new Error('No embedding'); }
  const content = $json.content || '';
  const esc = s => "'" + String(s == null ? '' : s).replace(/'/g, "''") + "'";
  const vec = "'[" + emb.join(',') + "]'";
  const query =
    "INSERT INTO <TABLE> (title, content, source, content_hash, embedding) VALUES (" +
    esc(content.slice(0,60)) + ", " + esc(content) + ", 'manual', md5(" + esc(content) + "), " +
    vec + "::vector) ON CONFLICT (content_hash) DO NOTHING;";
  return { query };
  ```
- **Postgres** : Execute Query = `{{ $json.query }}`.

> Pour une source comme Notion : Trigger Notion/Schedule → récup blocs → **Filter** (`content` not empty, sinon OpenAI rejette les blocs vides) → Embed → Build Insert (`source='notion'`) → Postgres.

---

## Étape 3 — Récupération (recherche sémantique)

Workflow : **Set** (champ `question`) → **Embed** (même config, `input: $json.question`) → **Build Search** (Code) → **Postgres Execute Query**.

```js
const emb = $json.data[0].embedding;
const vec = "'[" + emb.join(',') + "]'";
const query =
  "SELECT id, title, left(content,200) AS extrait, " +
  "round((1 - (embedding <=> " + vec + "::vector))::numeric, 4) AS similarite " +
  "FROM <TABLE> ORDER BY embedding <=> " + vec + "::vector LIMIT 5;";
return { query };
```

> `<=>` = distance cosinus. `1 - distance` = score de similarité.

---

## Étape 4 — Webhook (API de la mémoire)

Workflow : **Webhook** (POST `<WEBHOOK_PATH>`) → **Embed** (`input: $json.body.question`) → **Build Search** → **Postgres** → **Respond to Webhook**.
Active/Publie → endpoint live : `POST https://<DOMAIN>/webhook/<WEBHOOK_PATH>` body `{"question":"..."}`.

---

## Étape 5 — Un agent (le superviseur)

Workflow : **Chat Trigger** → **AI Agent**, avec 3 sous-nodes :
- **Chat Model** (ex. OpenAI gpt-4o-mini).
- **Tool « Knowledge »** = HTTP Request Tool vers le webhook de l'Étape 4 :
  - Method POST, URL `https://<DOMAIN>/webhook/<WEBHOOK_PATH>`
  - Body : `{ "question": "{{ $fromAI('question', 'the question to search in the central memory') }}" }`
  - Description : `Search the central memory before answering.`
- **Chat memory** = Postgres Chat Memory (table d'historique séparée), *Context Window* ~15.
- **System Message** (anglais) : identité + mission + « always query the Knowledge tool before answering » + périmètre + « Respond in the user's language ».

---

## Étape 6 — Orchestration (hub-and-spoke)

**Sur chaque spécialiste :**
1. Trigger **« When Executed by Another Workflow »** avec champ `query` (String).
2. AI Agent → **Source for Prompt** = `{{ $json.query || $json.chatInput }}`.
3. Les spécialistes doivent être **actifs**.
4. (Recommandé) Pas de Chat memory sur les spécialistes — seul le superviseur garde le fil. Sinon, Chat memory → Session ID = `{{ $json.sessionId || 'subagent' }}`.

**Sur le superviseur :** un node **« Call n8n Workflow Tool »** par spécialiste :
- Sélectionne le workflow cible → la ligne `query` apparaît → bouton **✨** (insère `$fromAI`).
- Description claire de quand déléguer à ce spécialiste.
- Ajoute au System Message du superviseur la liste des outils de délégation.

> Pièges : (a) `query` n'apparaît qu'après sélection du workflow ; (b) « not active » = active le sous-workflow ; (c) erreur Chat memory en sous-workflow = fix Session ID ci-dessus.

---

## Étape 7 — Serveur MCP (exposer à Claude)

1. Nouveau workflow : node **MCP Server Trigger**.
2. Attache un **HTTP Request Tool** (ou Call Workflow Tool) qui appelle ton webhook mémoire :
   - Name : `search_memory`
   - Description : `Search the central memory. Returns the top matching entries.`
   - POST `https://<DOMAIN>/webhook/<WEBHOOK_PATH>`, body `{ "question": "{{ $fromAI('question', '...') }}" }`
3. **Active** le workflow → copie la **Production URL (SSE)**.
4. Dans Claude : **Customize → Connectors → + → Add custom connector** → colle l'URL (HTTPS). (Claude Code : `claude mcp add --transport sse <name> <URL>`.)
5. Teste : demande à Claude une info qui n'existe que dans ta mémoire.

**Auth :** MVP = sans auth (URL à identifiant aléatoire). Production = **OAuth2** côté MCP Server Trigger (l'UI connecteur de Claude gère OAuth ; Claude Code accepte aussi un header Bearer).

---

## Pièges génériques n8n (mémo)

1. **Credentials non liés** après import JSON / parfois après duplication → re-sélectionner OpenAI/Postgres dans chaque node.
2. **Code « Run Once for All Items »** ne sort qu'1 item → « Run Once for Each Item » + `return { query };` (objet, pas tableau).
3. **Blocs vides** d'une source → **Filter** avant Embed.
4. **Pool de connexions Postgres figé** après changement d'image → redémarrer n8n, ou exécuter le DDL via le terminal psql.
5. **Hostnames partagés** (ex. `redis`, `postgres`) sur réseau Docker mutualisé → utiliser le **nom de conteneur unique**.
6. **« Publish »** dans n8n = activer la prod.
