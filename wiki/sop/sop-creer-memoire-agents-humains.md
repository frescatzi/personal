---
type: wiki
title: SOP Lumina — Créer la mémoire (agents + humains) depuis le wiki Git
status: active
publish: notion
vault: ai-automation
brand:
sources:
  - raw/2026-06-29--sop-creer-la-memoire-agents-et-humains.md
related:
  - concept-pipeline-memoire-wiki-git
  - sop/SOP_systeme-multi-agents-memoire-centrale-mcp-n8n
  - sop/SOP_installer-pgvector-sur-postgres-coolify
  - synthese-lumina-systeme-reference
  - sop/sop-lumina-intake-et-publish
updated: 2026-06-29
---

# SOP Lumina — Créer la mémoire (agents + humains) depuis le wiki Git

> **Règle d'or :** Git = source unique. pgvector et Notion se **régénèrent** depuis Git. On ne les édite jamais directement.

Pour le pattern générique (sans stack Lumina), voir [[concept-pipeline-memoire-wiki-git]].
Pour la stack pgvector + MCP + agents hub-and-spoke, voir [[sop/SOP_systeme-multi-agents-memoire-centrale-mcp-n8n]].

---

## Objectif

À partir du wiki Git (source de vérité), alimenter **automatiquement** deux mémoires dérivées en parallèle :
- **pgvector** (mémoire des AGENTS) : recherche sémantique via OpenAI embeddings.
- **Notion** (vue des HUMAINS) : lecture, partage, mobile.

```
Obsidian (coffre + graphe)
     │  plugin Obsidian Git : commit + push auto
     ▼
GitHub  wiki/  ← SOURCE DE VÉRITÉ
     │
┌────┴──────────────────────────┐
▼ n8n + OpenAI embeddings        ▼ n8n + Notion API
pgvector (mémoire AGENTS)       Notion (vue HUMAINS)
TRUNCATE + reload               upsert par fichier
```

---

## Pré-requis

- Obsidian vault = repo Git (même dossier). Plugin **Obsidian Git** actif.
- PostgreSQL + pgvector ; table `asp_memory` — voir [[sop/SOP_installer-pgvector-sur-postgres-coolify]].
- Credentials n8n : **GitHub**, **OpenAI**, **Notion**.
- Embedding **figé** : `text-embedding-3-small` (1536 dimensions). Ne jamais changer = tout ré-encoder.

---

## Partie 1 — Obsidian & synchro GitHub

**Éléments :**
- **Vault Obsidian = repo Git** (`raw/`, `wiki/`, `CLAUDE.md`, `index.md`, `log.md`) — ouvert via *Open folder as vault*.
- **Plugin Obsidian Git** — commit + push auto (~10 min). C'est lui qui **déclenche** les pipelines.
- **`raw/` vs `wiki/`** — on ingère/publie le `wiki/`, pas `raw/`.
- **Accès distant (optionnel)** — Obsidian sur Coolify (`linuxserver/obsidian`, port 3000). ⚠️ Synchro via **Git** (pas CouchDB/LiveSync).

**Gouvernance :** accès restreint au fondateur aujourd'hui, ouvert progressivement aux chefs de département.

---

## Partie 2 — Mémoire des AGENTS : `LUMINA — Memory — Ingestion (Wiki→pgvector)`

Workflow idempotent : reconstruit `asp_memory` en entier à chaque run → miroir exact du wiki.

**Chaîne :**
1. **Trigger** — manuel (test) ou webhook (prod).
2. **Truncate** — `TRUNCATE asp_memory;` → garantit un miroir propre.
3. **HTTP Request** (GitHub Trees API `git/trees/main?recursive=1`) — liste tous les fichiers en un appel.
4. **Split Out** (`tree`) — 1 item par fichier.
5. **Filter** (`type=blob` ET `path` commence par `wiki/` ET finit par `.md`) — seules les pages wiki.
6. **Get a file** (`File Path = {{ $json.path }}`, **As Binary OFF**) — contenu réel.
7. **Set Document** — mise en forme + métadonnées (`document`, `title`, `source`, `source_ref`, `collection`, `knowledge_type`).
8. **Chunk** (Code, `for … of $input.all()`) — découpe en morceaux digestes. ⚠️ `$input.all()` et non `$input.first()`.
9. **Embed (OpenAI `text-embedding-3-small`)** — chaque morceau → vecteur 1536 dim.
10. **Build Insert** — prépare la requête SQL.
11. **Postgres Insert** (`{{ $json.query }}`) — écrit dans `asp_memory`.

**Vérif :** `SELECT source_ref, count(*) FROM asp_memory GROUP BY source_ref;` → uniquement `wiki/…`.

---

## Partie 3 — Vue des HUMAINS : `PUBLISH-NOTION`

Workflow upsert par fichier : archive l'ancienne page, recrée la fraîche → Notion reste propre.

**Chaîne :**
1. **Build Notion payload** (Code) — construit la page ; pose l'**URL Source** comme clé (`payload.properties.Source.url`).
2. **Query Notion DB BY SOURCE** — cherche si une page existe déjà. ⚠️ Corps du filtre en **Expression + `JSON.stringify`** ; header **`Notion-Version: 2022-06-28`**.
3. **Split Out** (`results`) — éclate les résultats.
4. **Archive page** (`PATCH /v1/pages/{id}` `{"archived": true}`) — archive l'ancienne version.
5. **Create page** — recrée la page à jour dans la base Notion.

**IDs Lumina :** Notion DB `6fac8cb891e44c749e37a86419826905` · data_source `657827e3-1e2a-48d5-8f52-5954e3618687` · intégration **AFTRN-n8n** (doit être en **Connection** sur la base Notion).

> Configs exhaustives node par node : voir [[synthese-lumina-systeme-reference]] (v1.2).

---

## Partie 4 — Automatisation : `LUMINA — Orchestrateur — Sync`

**Éléments :**
- **Webhook GitHub** (repo Settings → Webhooks, event = push, secret) → notifie n8n.
- **Webhook Trigger n8n** — filtre branche `main` + chemins `wiki/**`.
- **2× Execute Workflow** — lance en **parallèle** Ingestion (Partie 2) et Publish-Notion (Partie 3).

Idempotents → re-déclencher à chaque push est totalement sûr. Résultat : écriture dans Obsidian = agents et humains à jour quelques secondes après.

---

## Erreurs fréquentes (vécues)

| Symptôme | Cause | Solution |
|---|---|---|
| Un seul doc ingéré | Chunk en `$input.first()` | `for … of $input.all()` |
| 404 sur Get a file | `File Path` en *Fixed* | Passer en *Expression* |
| Pas de texte / base64 | `As Binary` resté ON | Binary OFF sur Get a file |
| Notion : 0 résultat au filtre | Corps non en Expression | `JSON.stringify` dans le body |
| Doublons Notion | Pas d'archivage ou mauvais champ | Archiver l'ancienne page ; matcher sur `Source.url` |
| Node Anthropic n8n = 404 | Bug nœud natif | Toujours passer par **HTTP Request** pour Claude |

---

## Points critiques

- **Idempotence** : pgvector = TRUNCATE+reload ; Notion = archive+recrée par fichier.
- **Embedding figé** à 1536 dim (`text-embedding-3-small`).
- Ne **jamais** éditer pgvector ou Notion à la main.
- Adaptation à une autre marque : changer owner/repo, base Notion, collection. Pipelines réutilisables.
