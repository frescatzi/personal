---
type: wiki
title: "Architecture de la Connaissance — Modèle Git-hub (Obsidian + LLM Wiki)"
status: active
publish: notion
vault: ai-automation
brand: null
sources: []
related: ["synthese-lumina-systeme-reference", "architecture/brief-montage-vaults", "architecture/Memoire_Centrale_ASP_Brief_Construction"]
updated: 2026-06-20
---

# Architecture de la Connaissance — Modèle Git-hub (Obsidian + LLM Wiki)

**Date :** 2026-06-20
**Projet :** AFTER SUN PEOPLE (AFTRSN) — AI Operating System
**Statut :** Recommandation « version robuste » (v2.0) — supersède la v1.0 (chaîne Obsidian→Notion→pgvector)
**Inspiration :** Andrej Karpathy — « LLM Wiki » (gist) + pratique markdown/local-first.

---

## 1. Principe directeur

> **Le repo Git est la source de vérité unique. pgvector et Notion sont des SORTIES dérivées, parallèles et jetables, générées directement depuis Git.**

On ne fait **jamais transiter** la connaissance qui fait autorité *à travers* une sortie (Notion) pour atteindre une autre (les agents). Chaque consommateur lit depuis la source par le chemin le plus propre. Tout ce qui n'est pas la source doit être **régénérable** et **jamais édité à la main**.

C'est ce qui rend le système stable : une seule vérité, le reste est reconstructible.

---

## 2. Le coffre (Obsidian = repo Git = source de vérité)

Structure inspirée du « LLM Wiki » de Karpathy :

```text
AFTRSN-vault/            (un repo Git, ouvert comme coffre Obsidian)
├── raw/                 # sources CURÉES et IMMUABLES (articles, papiers, exports,
│                        #   refs trouvées en ligne). Le LLM les LIT, ne les modifie JAMAIS.
├── wiki/                # pages PROPRES — résumés, pages-concepts, synthèses, backlinks.
│                        #   ÉCRITES & MAINTENUES PAR LE LLM. Tu lis ; le LLM écrit.
├── CLAUDE.md            # LE SCHÉMA : règles de structure + workflows (ingest / Q&A / maintenance)
├── index.md            # carte de navigation du wiki (maintenue par le LLM)
└── log.md              # journal append-only, chronologique (ingests, requêtes, lints)
```

**Répartition des rôles humain / LLM (Karpathy) :**
- **Toi (humain) :** curer les sources (les ranger dans `raw/`), diriger l'analyse, poser les bonnes questions, décider du sens.
- **Le LLM :** tout le reste — compiler `wiki/`, relier, résumer, maintenir l'index, « linter ».

> ⚠️ `raw/` n'est **pas du vrac** : ce sont des sources que *tu* as choisies et rangées. Il n'y a pas d'« inbox en désordre » dans le coffre — `raw/` est curé, `wiki/` est propre.

---

## 2bis. Découpage en 3 vaults (décision 2026-06-20)

Pour rester simple (coûts/tokens maîtrisés), **3 vaults seulement** — chacun avec sa structure interne `raw/ + wiki/ + CLAUDE.md` :

- **Vault 1 — `ai-automation`** : apprentissage IA, API, MCP, agents, automation, ChatGPT / Claude / Gemini, etc.
- **Vault 2 — `brands`** : un **dossier par marque** (AFTRSN, …) contenant ses agents & skills. C'est **ce vault qui alimente la mémoire des agents** (pgvector).
- **Vault 3 — `personal`** : perso, business & connaissance générale (finances, santé, préceptes de livres, etc.).

**Routage (Claude Code via `CLAUDE.md`)** : tout topic entrant est classé dans l'un des 3 ; **s'il ne correspond à aucun, il n'est pas conservé** ; chaque décision journalisée dans `log.md`.

> Conséquence isolation : les marques étant des **dossiers** d'un même vault (et non des vaults séparés), l'étanchéité entre agents de marques se fait par **dossier/tag** à l'ingestion/récupération, pas par stores séparés. Acceptable pour des marques internes ; on pourra durcir plus tard si une marque l'exige.

---

## 3. Les deux sorties dérivées (parallèles, jamais en chaîne)

**pgvector** — index sémantique généré **directement depuis `wiki/`** (markdown propre → chunk → embedding → upsert). C'est l'outil de **récupération runtime des agents** (qui tournent côté serveur dans n8n et ne lisent pas le repo en direct). Régénérable à tout moment.

**Notion** — vue **humaine** générée **directement depuis `wiki/`** (sous-ensemble « à publier »). Lecture confortable, partage avec des non-techniciens, mobile, tableaux de bord. **Optionnelle et non porteuse** : le système tourne sans elle. On l'ajoute quand le besoin de partage est réel.

---

## 4. Schéma de flux

```text
                    HUMAINS curent les sources
                              │
   ┌──────────────────────────▼───────────────────────────┐
   │  OBSIDIAN = REPO GIT = SOURCE DE VÉRITÉ UNIQUE          │
   │    raw/ (immuable) · wiki/ (LLM) · CLAUDE.md · log.md   │
   └─────────┬───────────────────────────────┬─────────────┘
             │ généré depuis Git              │ généré depuis Git
             ▼                                ▼
   ┌────────────────────┐          ┌──────────────────────────┐
   │ pgvector            │          │ NOTION (optionnel)        │
   │ = mémoire AGENTS    │          │ = vue humaine             │
   │ (régénérable)       │          │ (régénérable, non porteur)│
   └────────────────────┘          └──────────────────────────┘
       ⤷ parallèles — Git est le moyeu, jamais une chaîne via Notion
```

---

## 5. Rôles révisés

| Couche | Rôle |
|---|---|
| **Obsidian (coffre = repo Git)** | **Source de vérité unique.** `raw/` curé par toi, `wiki/` maintenu par le LLM. Graphe / Marp / Dataview pour visualiser. |
| **GitHub** | Remote du coffre. Historique, rollback, branches, lecture par les agents/Claude Code. |
| **`CLAUDE.md` (schéma)** | Configuration qui discipline le LLM-bibliothécaire. Pièce à plus fort levier. |
| **PostgreSQL + pgvector** | Index sémantique **dérivé** du wiki → mémoire runtime des agents. Jamais la source. |
| **Notion** | Vue humaine **dérivée**, optionnelle, non porteuse. |
| **n8n** | Orchestration : ingestion Git→pgvector, et (plus tard) publication Git→Notion. |
| **Agents IA** | Lisent **pgvector + GitHub + APIs**. Jamais Notion comme source. |

---

## 6. Pourquoi c'est robuste

1. **Régénérabilité :** pgvector et Notion se reconstruisent depuis Git. Une seule vérité à protéger.
2. **Isolation des pannes :** Notion down/limité → agents intacts (ils dépendent de Git/pgvector, pas de Notion).
3. **Fidélité :** les agents reçoivent du markdown propre, sans aller-retour lossy par les blocs Notion.
4. **Versioning gratuit :** Git = historique, diff, rollback, branches.
5. **Ingestion idempotente :** ré-exécuter l'ingestion (upsert par `fichier + n° de chunk`) reproduit le même état → on peut toujours rebâtir pgvector proprement.

---

## 7. Ce qui change vs la « chaîne par Notion » — et ce qu'on garde

- **On re-pointe l'ingestion pgvector** pour lire **`wiki/` dans Git**, plus Notion.
- **On garde l'acquis :** l'extension `pgvector` + la table `asp_memory` restent (seul le *nœud source* de l'ingestion change). L'ingestion n'était de toute façon pas terminée (nœud n8n qui bloquait) → **aucune perte réelle**.
- **Notion sort du chemin des agents** et devient une vue humaine optionnelle.

---

## 8. Le fichier `CLAUDE.md` (schéma) — la pièce maîtresse

C'est le document qui dit au LLM : comment le wiki est structuré, les conventions (nommage, frontmatter, backlinks), et les workflows à suivre pour **ingérer une source**, **répondre à une question**, **maintenir/linter** le wiki. Il se co-évolue avec l'usage. Sans lui, le LLM est un chatbot générique ; avec lui, c'est un bibliothécaire discipliné. *(À rédiger en premier — voir séquence.)*

---

## 9. Opérations (best practices Karpathy)

- **Ingest :** déposer une source dans `raw/` → demander au LLM de mettre à jour `wiki/` (résumé, pages-concepts, backlinks) + une ligne dans `log.md`.
- **Q&A :** poser une question ; le LLM lit le wiki (via `index.md` + résumés) et répond, idéalement en produisant un fichier markdown / des slides Marp / une image, **refilé dans le wiki** pour qu'il s'enrichisse.
- **Lint :** périodiquement, « health-check » — contradictions, affirmations périmées, pages orphelines, concepts sans page, cross-refs manquantes, trous à combler par recherche web.
- **`log.md` :** append-only, préfixe régulier `## [AAAA-MM-JJ] ingest | Titre` → greppable.

---

## 10. Gouvernance & validation

- **Valider = promouvoir** une connaissance de `raw/` (brut curé) vers `wiki/` (propre, validé) — fait par le LLM, revu par toi.
- **Tâches critiques** (coûts, sécurité, accès API, suppression de mémoire) : **gate CEO** conservé.
- Statut léger via frontmatter (`status: draft | active | retired`) — pas de cycle de vie lourd au MVP.
- `raw/` immuable = provenance préservée.

---

## 11. Séquence de construction (MVP)

1. **Créer le repo `AFTRSN-vault`** (GitHub) + l'ouvrir comme coffre Obsidian. Installer le plugin **Obsidian Git** (auto-commit/push).
2. **Rédiger `CLAUDE.md`** (le schéma) + poser `raw/`, `wiki/`, `index.md`, `log.md`.
3. **Déposer 3-5 vraies sources** dans `raw/` (exports Claude/ChatGPT, refs).
4. **Faire compiler le premier `wiki/`** par Claude (Cowork/Code) à partir de `raw/`.
5. **Re-pointer l'ingestion `wiki/` (Git) → pgvector** (finir l'Étape 3, régler le nœud n8n qui bloquait). Upsert idempotent.
6. *(Plus tard, optionnel)* Publication **Git → Notion** pour le sous-ensemble humain.

---

## 12. Points de vigilance

- **Discipline Git** = condition de réussite (automatisée par le plugin Obsidian Git).
- **Ne jamais éditer pgvector ni Notion à la main** : ce sont des dérivés.
- **`raw/` reste immuable** : on n'y réécrit pas, on ajoute.
- **Ne pas sur-construire Notion** : vue humaine, fast-follow, pas le MVP.
- **Coût tokens** : compiler/linter le wiki consomme — OK à petite échelle, à surveiller.

---

## Version
v2.1 — Architecture Git-hub + découpage 3 vaults — 2026-06-20
*(supersède v1.0 Obsidian-centric en chaîne)*
