---
type: raw
title: "POS-GENERIQUE_Cablage-orchestrateur-subagents-router-n8n"
source_url: "drive:1cHW-ArRKO3PcHqi0pYKKeCiTUqL-ey-5"
captured: 2026-07-07
vault: ai-automation
brand: null
immutable: true
---

# LUMINA — POS GÉNÉRIQUE — Câbler un orchestrateur AI Agent à des sub-agents et un routeur LLM (n8n)

Patron réutilisable (indépendant de la marque) : un agent-chef délègue à N sub-agents spécialisés et route les tâches lourdes vers le bon LLM via un routeur.

## 1. Architecture

- **Orchestrateur** : AI Agent (chat trigger) + chat memory + outils = N `Call n8n Workflow Tool` + mémoire vectorielle (`Knowledge`) + option routeur.
- **Sub-agent** : workflow séparé avec **double trigger** — chat (test manuel) + `When Executed by Another Workflow` (appel par l'orchestrateur). Prompt = `{{ $json.query || $json.chatInput }}`.
- **Routeur** : sub-workflow `task_type → modèle` (voir POS routeur multi-LLM).

## 2. Recette d'un tool de délégation

Node `toolWorkflow` :
1. `workflowId` → le sub-agent cible.
2. `workflowInputs` : une entrée `query` = `{{ $fromAI('query', 'description de ce qu'il faut envoyer') }}` — l'agent remplit lui-même le champ.
3. `description` du node = **contrat de délégation** (quand appeler cet agent, ce qu'il rend). C'est ce que lit l'orchestrateur pour choisir.
4. Connexion `ai_tool` vers le node AI Agent.
5. Le prompt de l'orchestrateur doit nommer ses outils et donner les règles (appels ciblés, parallèle, chaîne courte, validation humaine des actions critiques).

## 3. Brancher le routeur comme outil

- Entrées à plat (`task_type`, `instruction`, `context_raw`) via `$fromAI` — lister les `task_type` valides dans la description.
- Côté routeur, accepter le contrat à plat ET le contrat imbriqué (rétro-compatibilité programmatique).
- Ajouter une section `ROUTING:` au prompt de l'orchestrateur (quoi router, quoi ne jamais router : PII).

## 4. Pièges (appris en conditions réelles)

1. **Trigger `executeWorkflowTrigger` en mode jsonExample = filtre silencieux** : tout champ absent de l'exemple JSON est droppé sans erreur. Symptôme : le sub-workflow tourne avec des valeurs par défaut. Fix : déclarer tous les champs des contrats acceptés dans l'exemple.
2. **Publication ≠ sauvegarde** : une modif (API ou éditeur) crée un draft ; sans Publish, la PROD exécute l'ancienne version.
3. **Publication refusée si credential manquante** sur n'importe quel node (message explicite). Copier la credential d'un node équivalent sain.
4. **Nom du node = référence** : l'orchestrateur choisit ses outils par nom + description ; nommer clairement (`Call '<Agent>'`).
5. Alias flottants de passerelle (`~…-latest`) : à valider une fois en conditions réelles (vérifier le `model` résolu dans la réponse), puis on peut s'y fier contre le churn.

## 5. Tester (ordre)

1. Sub-agent seul via son chat trigger.
2. Délégation : demander à l'orchestrateur une tâche qui force UN sub-agent → node vert + trace dans la réponse.
3. Routage : demander un `task_type` précis → vérifier le `model` résolu dans l'output du tool.
4. Vérifier l'exécution dans Executions (statut, entrées réellement reçues).

---
*POS GÉNÉRIQUE — 2026-07-02. Version exacte : POS-AFTRSN câblage Maestro. Complète : POS routeur multi-LLM, POS hub-and-spoke.*
