---
type: raw
title: "brief-montage-vaults"
source_url: "drive:1HvMAKMqVX2pnfLCcAQ85eia4_Pno1jIm"
captured: 2026-07-07
vault: ai-automation
brand: null
immutable: true
---

---
type: wiki
title: "Architecture — Montage des 3 vaults Obsidian (brief de référence)"
status: active
publish: notion
vault: ai-automation
brand: null
sources:
  - "raw/2026-06-24--brief-claudecode-montage-vaults-obsidian.md"
  - "raw/2026-06-29--brief-claudecode-montage-vaults-obsidian.md"
related:
  - "synthese-lumina-systeme-reference"
  - "architecture/Instructions_Projet_ChatGPT_n8n_v2"
  - "concept-intake-source-git"
updated: 2026-06-29
---

# Architecture — Montage des 3 vaults Obsidian

> Brief de référence pour la structure des 3 vaults Git/Obsidian constituant le système Lumina.
> Voir [[synthese-lumina-systeme-reference]] pour l'état complet du système.

---

## 1. Les 3 vaults et leur périmètre

```
ai-automation/      (IA, API, MCP, agents, automation, LLMs)
brands/             (1 dossier par marque ; nourrit pgvector des agents)
personal/           (perso, business & général : finances, santé, livres)
```

Chaque vault a la **même structure interne** :

```
<vault>/
  CLAUDE.md      ← schéma du vault (conventions + workflows)
  raw/           ← sources curées, IMMUABLES (le LLM lit, ne modifie jamais)
  wiki/          ← pages propres, écrites/maintenues par le LLM
  index.md       ← carte de navigation
  log.md         ← journal append-only
```

Vault `brands/` a en plus **un dossier par marque** :

```
brands/
  CLAUDE.md
  AFTRSN/
    raw/   wiki/
```

---

## 2. Routeur (logique de tri d'un topic entrant)

Un seul classement par topic — sinon non conservé :

1. IA / Automation (API, MCP, agents, n8n, ChatGPT/Claude/Gemini…) → **`ai-automation`**
2. Une marque précise (AFTRSN…) → **`brands/<marque>`**
3. Perso / business / général (finances, santé, préceptes de livres…) → **`personal`**
4. Aucun des 3 → **ignoré** (non conservé)

Journaliser chaque décision d'inclusion/exclusion dans `log.md`.

---

## 3. Conventions de nommage

- **`raw/`** : `AAAA-MM-JJ--source-titre-court.md`
- **`wiki/`** : `concept-nom.md` (page-concept) ou `synthese-sujet.md` (synthèse) ou `sop/sop-nom.md`
- Kebab-case, sans accents, sans majuscules.

---

## 4. Workflows du LLM (résumé)

| Workflow | Déclencheur | Sortie |
|---|---|---|
| **Ingest** | Nouveau fichier dans `raw/` | Page(s) `wiki/` créées/mises à jour + `index.md` + `log.md` |
| **Q&A** | Question → index.md → pages wiki → réponse + trace wiki si utile | |
| **Lint** | Périodique | Rapport contradictions/orphelines/trous + corrections |

---

## 5. Règles d'or

- `raw/` **immuable** ; `wiki/` = **domaine du LLM** ; pgvector et Notion = **dérivés régénérables**.
- **Vault `brands/`** est le seul qui alimente la **mémoire pgvector des agents**.
- Tout passe par **Git** (commit/push) — plugin **Obsidian Git** pour automatiser.
- Ne pas conserver un topic hors périmètre « pour ranger plus tard ».

---

## 6. Affectation des fichiers existants (session 2026-06-20)

**`wiki/architecture/`** : Architecture_Connaissance_Obsidian_Centric, Instructions_Projet_ChatGPT_n8n_v2, Claude_Review_Knowledge_Governance_Layer_v1, Memoire_Centrale_ASP_Brief_Construction, Plan_Demarrage_Memoire_Centrale_MVP

**`wiki/sop/`** : SOP_installer-pgvector-sur-postgres-coolify, SOP_systeme-multi-agents-memoire-centrale-mcp-n8n, Guide-Connexion-Agents-AI-n8n, n8n-Brancher-API-et-Premier-Workflow

**`sessions/`** : fichiers de sessions datées (lecture seule)

*Brief montage vaults v1.0 — 2026-06-20 (archivé dans wiki 2026-06-24).*
