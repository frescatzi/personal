---
type: wiki
title: "Lumina — Système de connaissance (référence complète)"
status: active
publish: notion
vault: ai-automation
brand: null
sources:
  - raw/2026-06-24--synthese-lumina-systeme-reference.md
  - raw/2026-07-02--lumina-ai-os-passation.md
  - raw/2026-07-02--lumina-playbook-v1-2026-07-02.md
  - raw/2026-07-02--milestone-cablage-maestro-router-2026-07-02.md
related:
  - sop/sop-lumina-intake-et-publish
  - sop/SOP_systeme-multi-agents-memoire-centrale-mcp-n8n
  - concept-limites-api-claude
  - architecture/Instructions_Projet_ChatGPT_n8n_v2
  - synthese-lumina-ai-os
  - concept-routeur-multi-llm
  - concept-memoire-vectorielle-multi-marques
  - concept-memoire-vivante-agents
  - concept-classification-workflows-n8n
updated: 2026-07-06
---

# LUMINA — Système de connaissance (référence complète)

> AI Operating System d'AFTER SUN PEOPLE (AFTRSN). Modèle **hub parallèle** : `Git (source de vérité) → pgvector` **ET** `Git → Notion`, en parallèle, jamais en chaîne.
> État : **complet et fonctionnel de bout en bout** (2026-06-23).

---

## 1. Vue d'ensemble (4 couches)

```
   Drive Inbox ──(robot Gemini)──▶ GitHub raw/ ──(Claude Code)──▶ wiki/
                                                      │
                        ┌──────────────────────────────┴──────────────┐
                        ▼ n8n + OpenAI                                 ▼ n8n + Notion API
                   pgvector (mémoire AGENTS)                    Notion (vue HUMAINS)
                   MCP search_brand_memory / webhook            contenu formaté, par marque
```

- **Source de vérité = Git** (repos `frescatzi/ai-automation`, `brands`, `personal` + `lumina-meta`). pgvector et Notion = **dérivés régénérables, jamais édités à la main**.
- **Embedding figé** : OpenAI `text-embedding-3-small` (1536 dim).
- **Piège** : node Anthropic natif n8n interdit (bug 404) → HTTP Request. Voir [[concept-limites-api-claude]].

---

## 2. Couche 1 — Intake (robot d'ingestion) ✅

Workflow n8n « Lumina – Ingestion » : Drive Inbox → LLM classe → écrit dans `<coffre>/raw/` sur GitHub → vide l'inbox.
- Trigger Google Drive (folder Lumina Inbox `1gjuB438pW7NKo4A-wbjaud_PH7ijnNUW`) → Download → Extract from File → **LLM CALL** (classification JSON) → **Code_Parse** → IF (reject) → **GitHub Create file** (repo `By URL` = `https://github.com/frescatzi/{{ $json.repo }}`) → **Drive Delete**.
- Sécurité : GitHub node `On Error = Stop Workflow` (Delete ne tourne jamais sans écriture réussie).

→ Procédure détaillée : [[sop/sop-lumina-intake-et-publish]]

## 3. Couche 2 — Rédaction wiki (Claude Code) ✅

`raw/ → wiki/` : Claude Code en local (`~/Documents/Lumina AI/GitHub/ai-automation`), guidé par `CLAUDE.md`. Produit `concept-*` / `synthese-*`, backlinks, `index.md`, `log.md`. Commit + push via Obsidian Git. Validation humaine : `status: draft → active`.

## 4. Couche 3 — Mémoire des agents (pgvector) ✅

Workflow `LUMINA — Memory — Ingestion (Wiki→pgvector)` : Trigger → **Truncate `asp_memory`** → HTTP GitHub Trees API → Split Out → Filter `wiki/*.md` → Get a file → Set Document → Chunk → Embed (OpenAI 1536) → Build Insert → Postgres Insert. **Idempotent** (TRUNCATE + reload). Récupération : MCP `search_brand_memory` + webhook.
- Table `asp_memory` : `id, content, source, source_ref, metadata JSONB, embedding VECTOR(1536), created_at`. Index HNSW cosinus.

→ Procédure détaillée : [[sop/SOP_systeme-multi-agents-memoire-centrale-mcp-n8n]]

## 5. Couche 4 — Vue humaine (Notion) ✅

Base Notion **« AI Automation — Knowledge »** (teamspace AI-AUTOMATION).
- DB id : `6fac8cb891e44c749e37a86419826905` · data_source : `657827e3-1e2a-48d5-8f52-5954e3618687`
- Propriétés : `Page` (title), `Type` (select concept/synthese/sop), `Source` (url GitHub), `Tags` (multi), `Vault` (select), `Updated` (date).
- ⚠️ Intégration n8n **AFTRN-n8n** doit être **ajoutée en Connection** sur la base (sinon « database not found »).

Workflow `LUMINA-MEMORY-INGESTION/PUBLISH-NOTION` — **idempotent (upsert par fichier)** :
```
Trigger → Query Notion DB → HTTP GitHub Trees → Split Out1 (tree) → Filter (wiki/*.md, exclut README)
        → Get a file → Set Document → Code (Parsing)
              ├─→ Build Notion payload → POST Notion          (créer la version fraîche)
              └─→ Query BY SOURCE → Split Out (results) → ARCHIVE PAGE   (archiver l'ancienne)
```

→ Procédure détaillée : [[sop/sop-lumina-intake-et-publish]] § SOP B. Patron générique : [[sop/sop-generique-pipeline-source-vers-vues]].

### Pièges résolus (2026-06-24)
1. **Fil parasite `Query Notion DB → Code (Parsing)`** → page fantôme. Fix : Code (Parsing) ne reçoit QUE Set Document.
2. **Expression `{{ }}` non évaluée** dans body JSON → fix : mode Expression + `JSON.stringify`.
3. **`sourceUrl` tronqué** → fix : filtrer sur `payload.properties.Source.url` (Build payload).
4. **README.md inclus** → fix : condition `path does not contain README`.
5. **Race archive/create** : ne jamais lancer la branche archive seule ; re-remplir = 1 run publish complet.

---

## 6. Évolution — Lumina AI OS v2 (multi-LLM, multi-brand, multi-agents)

*(Sources ingérées 2026-07-02. Synthèse complète → [[synthese-lumina-ai-os]])*

### Architecture cible

```
[Orchestrateur Maestro] ─── toolWorkflow ──→ Sub-agents (secrétariat, recherche, copy…)
        │                ─── toolWorkflow ──→ LUMINA-AI-Router (task_type → modèle)
        └────────────── Knowledge ──────────→ <code>_memory (pgvector, par marque)
```

### Routeur LLM multi-modèle

- Un seul credential OpenRouter (passerelle OpenAI-compatible) → 1 node HTTP → N modèles.
- Table task_type → modèle : `reasoning` → o3-mini · `copy_writing` → Claude Sonnet · `research` → Perplexity · `quick_answer` → GPT-4o-mini · `local_sensitive` → Hermes (local).
- Alias anti-churn (`~…-latest`) pour ne pas rééditer à chaque release modèle.
- Ne jamais router du PII vers le cloud (`local_sensitive` bloqué à Hermes).

→ Détail : [[concept-routeur-multi-llm]]

### Mémoire vectorielle multi-marques

- **Frontière physique** : `lumina_memory` (partagé) + `<code>_memory` (par marque) dans la même instance pgvector.
- **`memory_registry`** : table de routage source de vérité (brand_code → table + collection).
- Provisioning : `LUMINA-BRAND-PROVISION {code, brand_name, tone_voice, forbidden_words}` → crée table + index HNSW + ligne registry.
- Onboarding d'une nouvelle marque : provisionner → ingérer le canon → cloner le roster d'agents.

→ Détail : [[concept-memoire-vectorielle-multi-marques]] · [[sop/sop-clonage-roster-agents]]

### Mémoire vivante

- **WRITE** (Hermes, gratuit) : épisodique systématique après chaque exécution.
- **READ/RAG** : embed la question → cosinus → injecte dans `context_raw` avant appel LLM premium.
- **CONSOLIDATE** : cron nocturne → 3-5 insights distillés → `collection=insights`.

→ Détail : [[concept-memoire-vivante-agents]]

### Classification n8n

- Dossiers numérotés `01-INTAKE → 05-CONNECT` · 4 axes de tags (domaine, statut, déclencheur, marque).
- Maestro en `03-BRAIN` · sous-agents en `03-BRAIN/Sub-Agents` · infra partagée en `01-LUMINA`.

→ Détail : [[concept-classification-workflows-n8n]]

---

## 7. ⏭️ À faire (prochaines étapes)

1. ✅ **Dédup / idempotence du publish Notion** — FAIT (2026-06-24).
2. **Automatiser les déclencheurs** : webhook GitHub push → ingestion pgvector + publish Notion auto.
3. **Répliquer** `brands` et `personal` (vaults + bases Notion) quand ils auront du contenu.
4. **Filtre publication** : n'envoyer vers Notion que `status: active` (décommenter la ligne dans le Code parse).

---

## 7. Décisions figées (mémo)

- Source de vérité = Git ; dérivés régénérables, jamais édités à la main.
- Embedding = `text-embedding-3-small` (1536). Jamais node Anthropic natif n8n.
- Idempotence : pgvector = TRUNCATE + reload ; Notion = **upsert par fichier** (archive ancienne + recrée).
- Notion : body du filtre toujours en **mode Expression** + `JSON.stringify` ; matcher sur `payload.properties.Source.url`.
- Un node Code à entrée unique : ne jamais y brancher un 2ᵉ flux (sinon items parasites).
- Nommage : `LUMINA — <Process> — <Détail>` (infra) ; `AFTRSN — …` (marque).
- Sécurité : workflow planifié+publié obsolète = **dépublier**. Tâches critiques → gate CEO.

*v1.1 — 2026-06-24 (dédup Notion idempotente terminée).*
