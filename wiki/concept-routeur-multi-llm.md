---
type: wiki
title: Routeur multi-LLM par task_type (passerelle OpenAI-compatible)
status: active
publish: notion
vault: ai-automation
brand:
sources:
  - raw/2026-07-02--lumina-ai-os-passation.md
  - raw/2026-07-02--pos-generique-routeur-multi-llm-par-task-type-passerelle-openai-compatible.md
  - raw/2026-07-06--pos-generique-routeur-multi-llm-par-task-type-passerelle-openai-compatible.md
  - raw/2026-07-02--lumina-pos-generique-routeur-multi-llm-par-task-type-via-passerelle-openai-compatible.md
  - raw/2026-07-02--aftrsn-lumina-pos-exact-lumina-ai-router-routage-multi-llm-via-openrouter-2026-07-01.md
related:
  - wiki/synthese-lumina-ai-os.md
  - wiki/sop/sop-cablage-orchestrateur-subagents.md
  - wiki/concept-memoire-vivante-agents.md
  - wiki/sop/Guide-Connexion-Agents-AI-n8n.md
updated: 2026-07-06
---

# Routeur multi-LLM par task_type (passerelle OpenAI-compatible)

## Idée centrale

**On ne colle pas un agent à un seul LLM.** Un *LLM Router* choisit dynamiquement le moteur selon la nature (`task_type`) de la tâche. Avantage : maintenance centralisée (changer de modèle = 1 ligne), séparation propre entre *qui fait quoi* (agents métier) et *quel moteur* (routeur).

## Architecture (3 nodes n8n)

```
Trigger (executeWorkflowTrigger, Accept all data)
    ↓
Code JavaScript (task_type → slug modèle + construction messages)
    ↓
HTTP Request (POST <gateway>/chat/completions)
```

**Contrat d'entrée :** `routing.task_type` · `brand_profile` (voix, mots interdits, `kb_ref`) · `task_payload` (context_raw, instruction). Variante à plat : `task_type`, `instruction`, `context_raw` (pour les appels-outils d'agents).

**Code :** lit `task_type` → mappe vers slug → construit `messages:[{system},{user}]` → garde-fou PII → sort `{model, messages}`.

**HTTP :** `POST <gateway>/chat/completions`, body expression `{{ JSON.stringify({ model: $json.model, messages: $json.messages }) }}`, credential unique.

## Matrice de routage (task_type → moteur)

| `task_type` | Moteur primaire | Critère dominant |
|-------------|-----------------|-----------------|
| `copy`, `reasoning`, `analysis`, `orchestrate` | Claude | Rédaction / raisonnement |
| `draft` (+ défaut) | ChatGPT | Idéation / divergence |
| `image_gen` | ChatGPT | Capacité native |
| `research`, `verify`, `summarize_long`, `translate`, `multimodal` | Gemini Flash | Recherche / contexte / multimodal |
| `classify`, `extract`, `preprocess`, `support_l1`, `realtime` | Hermes (local) | Coût + confidentialité |
| `private` | **Stop** (local uniquement) | PII — non négociable |
| `code` | Claude (complexe) / ChatGPT (scripts n8n/SQL) | Qualité / écosystème |

## Règles d'arbitrage (arbre de décision)

1. **PII ?** → Hermes local. Non-négociable.
2. **Volume + faible complexité ?** → Hermes ; sinon Gemini Flash.
3. **Nature :** Rédiger/raisonner → Claude ; Chercher/multimodal → Gemini ; Idéation/image → ChatGPT.
4. **Livrable critique** → chaîne multi-LLM (Hermes prépa → ChatGPT création → Claude ciselage → Gemini validation).
5. **Validation croisée** → 2ᵉ LLM uniquement sur livrables externes/critiques (sinon coût ×2).

## Passerelle OpenRouter (implémentation Lumina)

- **1 credential** (type OpenRouter) couvre tous les modèles → 1 seule clé.
- Alias **anti-churn** : `~anthropic/claude-sonnet-latest`, `~openai/gpt-mini-latest`, `~google/gemini-flash-latest` (OpenRouter met à jour automatiquement). Le `~` fait partie du slug.
- Fallbacks pinnés : `anthropic/claude-opus-4.8` · `openai/gpt-5.5` · `google/gemini-3.5-flash`.
- ⚠️ Pas de Haiku (retiré de l'API + bugs signalés).

## Modifier/ajouter un modèle

1. Node **Code** → bloc de mapping → éditer 1 ligne `task_type: 'provider/slug'`.
2. Vérifier le slug sur `openrouter.ai/models` (les slugs churnent à chaque génération).
3. Save → **Publish** (sans Publish, la PROD exécute l'ancienne version).

## Garde-fous & dépannage

| Symptôme | Cause | Fix |
|----------|-------|-----|
| `model not found` | Slug périmé | Corriger dans le bloc de mapping |
| `401` | Clé/crédits épuisés | Recharger le compte OpenRouter |
| Sortie = requête SQL (pas la réponse) | Lien Code→HTTP rompu | Recréer la connexion |
| `private` renvoie `{error}` | Normal | Protection PII active |

## Leçons clés

- Passerelle unique >> N branches (1 seule clé, 1 endpoint, maintenance en 1 bloc de mapping).
- Slugs churnent → config centralisée, revue régulière.
- Deux couches orthogonales : **agents métier** (QUI fait quoi) vs **routeur** (QUEL moteur) — elles se composent librement.
- Champs Expression n8n auto-ferment `{{ }}` → coller le code, pas taper.

## Voir aussi

- [[synthese-lumina-ai-os]] — contexte complet du système Lumina et du rôle du routeur.
- [[sop/sop-cablage-orchestrateur-subagents]] — comment brancher le routeur comme outil d'agent.
- [[concept-memoire-vivante-agents]] — la mémoire RAG injectée dans `task_payload.context_raw`.
- [[sop/Guide-Connexion-Agents-AI-n8n]] — créer et gérer les clés API dans n8n.
