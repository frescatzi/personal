---
type: raw
title: "Brief_ClaudeCode_Montage_Vaults_Obsidian"
source_url: "drive:1D8PISuTuNOl2rSABCwW5er0QqSvJmkuR"
captured: 2026-07-07
vault: ai-automation
brand: null
immutable: true
---

# Brief Claude Code — Monter les 3 vaults Obsidian

**But :** scaffolder 3 vaults Obsidian (= 3 repos Git), créer le routeur + les schémas `CLAUDE.md`, et y ranger les documents existants. À lancer toi-même dans **Claude Code** (apprentissage).

---

## 1. Structure des 3 vaults

```
ai-automation/      (Vault 1 — IA, API, MCP, agents, automation, LLMs)
brands/             (Vault 2 — 1 dossier par marque ; nourrit pgvector des agents)
personal/           (Vault 3 — perso, business & général : finances, santé, livres)
```

Chaque vault a la **même structure interne** :

```
<vault>/
  CLAUDE.md      # schéma du vault (conventions + workflows)
  raw/           # sources curées, IMMUABLES (le LLM lit, ne modifie jamais)
  wiki/          # pages propres, écrites/maintenues par le LLM
  index.md       # carte de navigation
  log.md         # journal append-only
```

Vault 2 a en plus un **dossier par marque** :

```
brands/
  CLAUDE.md
  AFTRSN/
    raw/   wiki/
```

---

## 2. Routeur (un `CLAUDE.md` racine, au-dessus des 3 vaults)

Logique de tri d'un topic entrant — **classer dans un seul des 3, sinon ne pas conserver** :

1. IA / Automation (API, MCP, agents, automation, ChatGPT/Claude/Gemini…) → **`ai-automation`**
2. Une marque précise (AFTRSN…) → **`brands/<marque>`**
3. Perso / business / général (finances, santé, préceptes de livres…) → **`personal`**
4. Aucun des 3 → **ignoré** (non conservé)

Journaliser chaque décision dans `log.md`.

---

## 3. Gabarit `CLAUDE.md` par vault

```md
# CLAUDE.md — Schéma du vault <nom>

## Rôle
Source de vérité (markdown + Git) pour : <périmètre du vault>.

## Ce qui appartient ici / ce qui n'y appartient pas
- Appartient : <ex.>
- N'appartient pas (rediriger) : <ex.>

## Couches
- raw/  : sources curées, IMMUABLES. Le LLM lit, ne modifie jamais.
- wiki/ : pages propres. Domaine du LLM (tu lis, il écrit).

## Workflows
- Ingest : nouvelle source → résumer, créer/mettre à jour pages-concepts + backlinks → 1 ligne dans log.md.
- Q&A : lire via index.md + résumés ; répondre ; refiler la sortie utile dans wiki/.
- Lint : repérer contradictions, pages orphelines, concepts sans page, trous à combler.

## Conventions
- Frontmatter obligatoire (voir §5).
- Noms de fichiers en kebab-case.
- Liens internes [[wikilinks]].
```

---

## 4. Frontmatter standard (en-tête de chaque note `wiki/`)

```yaml
---
title:
type: reference | sop | session | concept | agent
status: draft | active | retired
tags: []
---
```

## 5. Format `log.md` (greppable)

```
## [2026-06-20] ingest | Titre de la source
## [2026-06-20] lint   | Résumé du passage
```
(`grep "^## \[" log.md | tail -5` → 5 dernières entrées.)

---

## 6. Rangement des documents existants → tous dans `ai-automation`

Les 12 docs actuels concernent la **construction du système IA/n8n** → Vault 1. (Aucun n'est encore une connaissance de marque ou perso.)

**`ai-automation/wiki/architecture/`**
- Architecture_Connaissance_Obsidian_Centric.md
- Instructions_Projet_ChatGPT_n8n_v2.md
- Claude_Review_Knowledge_Governance_Layer_v1.md
- Memoire_Centrale_ASP_Brief_Construction.md
- Plan_Demarrage_Memoire_Centrale_MVP.md

**`ai-automation/wiki/sop/`**
- SOP_installer-pgvector-sur-postgres-coolify.md
- SOP_systeme-multi-agents-memoire-centrale-mcp-n8n.md
- Guide-Connexion-Agents-AI-n8n.md
- n8n-Brancher-API-et-Premier-Workflow.md

**`ai-automation/sessions/`**
- 2026-06-19_session_memoire-centrale_etapes-1-2.md
- 2026-06-20_session_agents-orchestration-mcp.md
- Session-Recap-n8n-Agents-AI.md

`brands/` et `personal/` restent vides au départ — ils se rempliront ensuite via le routeur.

---

## 7. Règles d'or

- `raw/` **immuable** ; `wiki/` = **domaine du LLM** ; **pgvector et Notion = dérivés régénérables**, jamais édités à la main.
- **Vault 2 `brands/`** est le seul qui alimente la **mémoire pgvector des agents** (étanchéité par dossier de marque).
- Tout passe par **Git** (commit/push) — installe le plugin **Obsidian Git** pour automatiser.

---

## 8. Prompt de départ (à coller dans Claude Code)

> « Crée 3 dossiers `ai-automation/`, `brands/`, `personal/`, chacun avec `raw/`, `wiki/`, `index.md`, `log.md` et un `CLAUDE.md` selon le gabarit fourni. Dans `brands/`, ajoute un sous-dossier `AFTRSN/` avec `raw/` et `wiki/`. Crée un `CLAUDE.md` racine implémentant le routeur (4 règles). Puis range les fichiers listés dans `ai-automation/wiki/architecture/`, `ai-automation/wiki/sop/`, `ai-automation/sessions/`. Initialise un repo Git par vault. Explique-moi chaque étape au fur et à mesure. »

---

## Version
v1.0 — Brief de montage vaults — 2026-06-20
