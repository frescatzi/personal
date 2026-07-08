---
type: wiki
title: SOP — Câbler un orchestrateur AI Agent à des sub-agents et un routeur LLM (n8n)
status: active
publish: notion
vault: ai-automation
brand:
sources:
  - raw/2026-07-02--pos-generique-cablage-orchestrateur-subagents-router-n8n.md
  - raw/2026-07-06--pos-generique-cablage-orchestrateur-subagents-router-n8n.md
related:
  - wiki/concept-routeur-multi-llm.md
  - wiki/synthese-lumina-ai-os.md
  - wiki/sop/sop-audit-edition-n8n-api-interne.md
  - wiki/sop/SOP_systeme-multi-agents-memoire-centrale-mcp-n8n.md
  - wiki/concept-classification-workflows-n8n.md
updated: 2026-07-06
---

# SOP — Câbler un orchestrateur AI Agent à des sub-agents et un routeur LLM (n8n)

## Architecture cible

```
[Orchestrateur] (AI Agent + chat trigger + chat memory)
    ├── toolWorkflow → Sub-agent A
    ├── toolWorkflow → Sub-agent B
    ├── toolWorkflow → LUMINA-AI-Router (task_type + instruction + context_raw)
    └── Knowledge (mémoire vectorielle)
```

## 1. Sub-agent : double trigger

Chaque sub-agent doit avoir **deux triggers** :
- **Chat trigger** (test manuel depuis l'éditeur).
- **`When Executed by Another Workflow`** (appel par l'orchestrateur en prod).

Prompt `{{ $json.query || $json.chatInput }}` pour tolérer les deux contrats.

## 2. Recette d'un tool de délégation (node `toolWorkflow`)

1. `workflowId` → l'ID du sub-agent cible.
2. `workflowInputs` : entrée `query` = `{{ $fromAI('query', 'description de ce qu\'il faut envoyer') }}` — l'orchestrateur remplit lui-même le champ.
3. `description` du node = **contrat de délégation** (quand appeler cet agent, ce qu'il rend) — c'est ce que lit l'orchestrateur pour choisir.
4. Connexion `ai_tool` → node AI Agent de l'orchestrateur.
5. Le prompt de l'orchestrateur nomme ses outils + donne les règles (appels ciblés, parallèle si indépendants, chaîne courte, validation humaine sur le critique).

## 3. Brancher le routeur LLM comme outil

- Entrées à plat : `task_type`, `instruction`, `context_raw` via `$fromAI` — lister les `task_type` valides dans la description du node.
- Côté routeur : accepter les deux contrats (à plat + imbriqué) pour la rétro-compatibilité.
- Ajouter une section `ROUTING:` au prompt de l'orchestrateur : quoi router, quoi ne **jamais** router (PII → local uniquement).

Voir [[concept-routeur-multi-llm]] pour la table de routage complète.

## 4. Pièges confirmés en conditions réelles

| Piège | Symptôme | Fix |
|-------|----------|-----|
| `executeWorkflowTrigger` en mode jsonExample | Champs absents de l'exemple droppés silencieusement | Déclarer tous les champs des contrats dans l'exemple |
| Publication non faite après PATCH | La PROD exécute l'ancienne version | Toujours `Save → Publish` |
| Credential manquante sur un node | Publication refusée (message explicite) | Copier la ref `{id, name}` d'un node équivalent sain |
| Nom du node = référence implicite | L'orchestrateur rate l'outil | Nommer clairement : `Call 'LUMINA-AI-Router'` |
| Alias flottants `~…-latest` | À valider une fois en réel | Vérifier le `model` résolu dans la réponse |

## 5. Prompt de l'orchestrateur (blocs clés)

```
IDENTITY: Tu es <Maestro/Rôle>, orchestrateur de <Marque>. ...
MEMORY: Requêtes Knowledge (brand=<code>) avant toute réponse.
TOOLS: [nommer chaque outil + quand l'utiliser]
ROUTING: Router via LUMINA-AI-Router pour les tâches lourdes (copy, reasoning, research...). Ne JAMAIS router du PII (task_type=private → bloqué).
NAMING: Voix de marque, mots interdits depuis brand_profile.
AUTHORITY: Actions critiques (dépense, envoi, suppression) → validation humaine.
```

## 6. Ordre de test

1. Sub-agent seul via son chat trigger.
2. Délégation : demander à l'orchestrateur une tâche qui force UN sub-agent → node vert + trace dans la réponse.
3. Routage : demander un `task_type` précis → vérifier le `model` résolu dans l'output du tool.
4. Vérifier dans Executions : statut, entrées réellement reçues par le sub-workflow.

## Voir aussi

- [[concept-routeur-multi-llm]] — table task_type → modèle, passerelle OpenRouter.
- [[synthese-lumina-ai-os]] — architecture Maestro + sub-agents en contexte Lumina.
- [[sop/sop-audit-edition-n8n-api-interne]] — éditer les workflows par API si l'éditeur est capricieux.
- [[sop/sop-agent-n8n-cookie-auth]] — brancher un outil vers un service web protégé par cookie.
- [[sop/SOP_systeme-multi-agents-memoire-centrale-mcp-n8n]] — SOP mémoire centrale MCP.
- [[concept-classification-workflows-n8n]] — ranger Maestro en `03-BRAIN`, sub-agents en `03-BRAIN/Sub-Agents`.
