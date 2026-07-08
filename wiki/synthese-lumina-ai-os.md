---
type: wiki
title: LUMINA AI OS — Système multi-agents & multi-LLM
status: active
publish: notion
vault: ai-automation
brand:
sources:
  - raw/2026-07-02--lumina-ai-os-passation.md
  - raw/2026-07-02--lumina-playbook-v1-2026-07-02.md
  - raw/2026-07-06--lumina-playbook-v1-2026-07-02.md
  - raw/2026-07-02--milestone-cablage-maestro-router-2026-07-02.md
  - raw/2026-07-06--milestone-cablage-maestro-router-2026-07-02.md
  - raw/2026-07-02--aftrsn-lumina-pos-exact-lumina-ai-router-routage-multi-llm-via-openrouter-2026-07-01.md
related:
  - wiki/concept-routeur-multi-llm.md
  - wiki/concept-memoire-vivante-agents.md
  - wiki/concept-memoire-vectorielle-multi-marques.md
  - wiki/concept-classification-workflows-n8n.md
  - wiki/sop/sop-cablage-orchestrateur-subagents.md
  - wiki/sop/SOP_systeme-multi-agents-memoire-centrale-mcp-n8n.md
  - wiki/synthese-lumina-systeme-reference.md
updated: 2026-07-06
---

# LUMINA AI OS — Système multi-agents & multi-LLM

## Idée centrale

Couche d'orchestration qui **sélectionne automatiquement le meilleur LLM** selon la nature de chaque tâche (qualité, coût, vitesse, confidentialité). Contrainte structurante : **brand-agnostic** — un seul workflow tourne pour toutes les marques du portefeuille, piloté par un fichier de config par marque. Les agents restent spécialisés par métier ; un *LLM Orchestrator (AI Router)* choisit dynamiquement le moteur à chaque étape.

## Stack technique

| Brique | Choix |
|--------|-------|
| Orchestration | **n8n** (workflows, Switch/Code, crons) |
| Infra | **Coolify** sur VPS (Docker, expose les endpoints) |
| LLM premium | ChatGPT (OpenAI) · Gemini (Google) · Claude (Anthropic) |
| LLM local | **Hermes** (Nous, auto-hébergé via Coolify — ⚠️ pas Ollama) |
| Passerelle LLM | **OpenRouter** (1 clé, endpoint OpenAI-compatible, 300+ modèles) |
| Mémoire | **pgvector** sur Postgres/Coolify |

## Profils LLM (critère dominant)

- **Claude** — rédaction premium, raisonnement, analyse, orchestration.
- **ChatGPT** — idéation, image, écosystème outils-agents.
- **Gemini** — recherche web, multimodal, synthèse de gros volumes.
- **Hermes (local)** — classification/extraction en volume, PII, mémoire continue (coût ≈ 0).

Voir [[concept-routeur-multi-llm]] pour la matrice et les règles de routage complètes.

## Règles de routage (ordre strict)

1. **PII en jeu ?** → Hermes local. Non-négociable.
2. **Volume + faible complexité ?** → Hermes ; à défaut Gemini Flash.
3. **Nature de la tâche** → Rédiger/raisonner → Claude ; Chercher/multimodal → Gemini ; Idéation/image/outils → ChatGPT.
4. **Livrable complexe** → chaîne multi-LLM : Hermes (prépa) → ChatGPT (création) → Claude (ciselage) → Gemini (validation).
5. **Validation croisée** → 2ᵉ LLM uniquement si livrable externe/critique.
6. **Temps réel** → Hermes ou Gemini Flash.

## Architecture n8n

```
[Maestro (AI Agent, hub)]
    ├── Call Culture-Steward
    ├── Call Experience-Designer
    ├── Call Secretary
    ├── Call Marketing
    └── Call LUMINA-AI-Router (task_type + instruction + context_raw)
                    ↓
           [Code: task_type → slug OpenRouter]
                    ↓
           [HTTP: POST openrouter.ai/api/v1/chat/completions]
```

Deux contrats d'entrée du Router : **à plat** (`task_type`, `instruction`, `context_raw`) pour les agents ; **imbriqué** (`routing.task_type`, `task_payload.*`) pour le contrat JSON brand-agnostic.

## Playbook v1 — généralisation multi-marques

Principe : **1 workflow, N marques**. Ce qui distingue une marque = sa banque mémoire (`<code>_memory`), son profil (`brand_profiles`), son canon ingéré. Tout le reste est partagé.

**Couches partagées :** Router · mémoire partagée `lumina_memory` · skills génériques · mémoire vivante · Hermes.

**Couches par marque :** banque `<code>_memory` + index HNSW · ligne `memory_registry` + `brand_profiles` · canon ingéré · roster d'agents cloné (Maestro + spécialistes).

**Onboarding d'une nouvelle marque :**
1. `LUMINA-BRAND-PROVISION {code, brand_name, tone_voice, forbidden_words}` → crée tout.
2. Ingérer le canon (bible de marque) → `collection='canon'`.
3. Cloner le roster (`cloneRoster`) → rebind `brand=<code>`, adapter prompts.
4. Vérifier : `memory-search {brand:<code>}` renvoie le canon.

Voir [[sop/sop-clonage-roster-agents]] et [[concept-memoire-vectorielle-multi-marques]].

## Couche mémoire Hermes (hippocampe du système)

Hermes est le seul moteur qu'on peut faire tourner sur **100 % du trafic** (coût ≈ 0). Il gère :
- **WRITE** : résume chaque exécution, extrait entités, embed → stocke dans pgvector.
- **READ/RAG** : interroge la base, injecte le contexte dans `task_payload.context_raw`.
- **CONSOLIDATION** (cron nocturne) : condense l'épisodique en insights sémantiques.
- **Fine-tuning LoRA** (tous 1-3 mois) : absorbe les meilleures données validées → actif propriétaire.

Voir [[concept-memoire-vivante-agents]] pour le détail du pattern.

## Milestone câblage Maestro↔Router (2026-07-02 — PROD)

- **A1** : tools Maestro recâblés (4 sub-agents, pattern `$fromAI('query')`). ✅
- **A2** : Secrétaire réparé (prompt v2, credential Postgres Chat Memory). ✅
- **A6** : Maestro → LUMINA-AI-Router câblé. Test `task_type research` → `google/gemini-3.5-flash`. ✅
- Restants : A3 (Hermes outil Ops) · A4 (chat memory Experience-Designer) · A5 (prompt Channel-Content) · A7 (modèles par rôle).

**Piège confirmé :** trigger `executeWorkflowTrigger` en mode `jsonExample` = filtre silencieux (champs absents de l'exemple droppés). Fix : déclarer tous les champs.

## IDs composants (instance Karter)

| Composant | ID n8n |
|-----------|--------|
| LUMINA-AI-Router | `Cu8SozYmondKM8RB` |
| memory-search | `ZhUeKCo8nMp35gk1` |
| memory-write | `zu4jfZbmDz8trQLl` |
| consolidation nocturne | `dVZyCwGYqnqMpS3P` |
| Hermes-Exec | `Dwv4rcMqNAyQzlrF` |
| brand-provision | `Ul7u33t5VG8MDXWj` |
| Maestro | `OlrQO21u178SjgBK` |

## Points ouverts

1. Couloir code : Claude vs ChatGPT sur code applicatif (à valider empiriquement).
2. Base vectorielle : Qdrant vs pgvector (décision infrastructure).
3. Seuils boucle procédurale : taux d'édition pour rétrograder un moteur.
4. Politique de validation croisée : liste des livrables "critiques".
5. Modèle d'embedding : Hermes vs modèle dédié.

## Voir aussi

- [[concept-routeur-multi-llm]] — matrice task_type → modèle, runbook routeur.
- [[concept-memoire-vivante-agents]] — WRITE/READ/CONSOLIDATE, fine-tuning LoRA.
- [[concept-memoire-vectorielle-multi-marques]] — pattern banque partagée + tables par marque.
- [[concept-classification-workflows-n8n]] — standard de rangement des workflows n8n.
- [[sop/sop-cablage-orchestrateur-subagents]] — recette câblage Maestro↔sub-agents.
- [[sop/sop-clonage-roster-agents]] — cloner un roster d'agents pour une nouvelle marque.
- [[sop/sop-audit-edition-n8n-api-interne]] — auditer/modifier les workflows par API interne.
- [[sop/sop-agent-n8n-cookie-auth]] — brancher un outil vers un service protégé par cookie.
- [[sop/sop-outreach-backfill]] — intégrer l'historique d'outreach manuel dans le pipeline.
- [[sop/sop-calendrier-contenu-agent]] — générer un calendrier de contenu en draft via agent.
- [[sop/SOP_systeme-multi-agents-memoire-centrale-mcp-n8n]] — SOP mémoire centrale MCP.
- [[synthese-lumina-systeme-reference]] — vue d'ensemble du système Lumina (4 couches).
