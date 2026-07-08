---
type: raw
title: "AFTRSN-LUMINA — POS EXACT — LUMINA-AI-Router (routage multi-LLM via OpenRouter) — 2026-07-01"
source_url: "drive:1CQBKtE88ebFBU2XfYaM2Y9z4FVQoYzxj"
captured: 2026-07-07
vault: ai-automation
brand: null
immutable: true
---

# POS EXACT — Runbook : LUMINA-AI-Router (routage multi-LLM par task_type via OpenRouter)

**Procédure Opérationnelle Standardisée (spécifique à ce système).**
**Cible :** n8n `n8n.aftersunpeople.com` · workflow **`LUMINA-AI-Router`** (id `Cu8SozYmondKM8RB`) · passerelle **OpenRouter**.
**Date :** 2026-07-01 · **Validateur final :** Katel (Karter).

> Référence d'exploitation. Détail pédagogique : « Marche à suivre EXACTE » associée.

---

## 1. Rôle

Point d'entrée unique qui **choisit le LLM selon `task_type`** puis appelle OpenRouter (passerelle OpenAI-compatible → Claude / GPT / Gemini avec **une seule clé**). Appelé par les agents (Maestro, spécialistes) via *Call n8n Workflow Tool*.

## 2. Chaîne (3 nodes)

`When Executed by Another Workflow (Accept all data)` → `Code in JavaScript` → `HTTP Request`.

- **Trigger** : Input data mode = **Accept all data** (accepte tout le payload JSON du contrat §6 de la stratégie).
- **Code** : lit `routing.task_type` → mappe vers un slug OpenRouter → sort `{ model, task_type, messages:[{system},{user}] }`. Garde-fou : `task_type = 'private'` → **stop** (renvoie `{error}`, **n'appelle pas** le cloud — règle PII non négociable).
- **HTTP Request** : `POST https://openrouter.ai/api/v1/chat/completions` · Authentication = **Predefined Credential Type → OpenRouter → "OpenRouter account"** · Body = JSON, **Expression** :
  `{{ JSON.stringify({ model: $json.model, messages: $json.messages }) }}`

## 3. Table task_type → modèle (dans le node Code, bloc `M`)

| Rôle | task_types | Slug OpenRouter (alias anti-churn `~…-latest`) |
|---|---|---|
| Claude (rédaction/raisonnement) | copy, reasoning, analysis, orchestrate | `~anthropic/claude-sonnet-latest` |
| GPT (idéation/défaut) | draft (+ défaut) | `~openai/gpt-mini-latest` |
| GPT (image) | image_gen | `~openai/gpt-latest` |
| Gemini Flash (recherche/volume) | research, summarize_long, translate, multimodal, verify, classify, extract, preprocess, support_l1, realtime | `~google/gemini-flash-latest` |
| PII | private | **stop** (pas de cloud) |

**Alias `~…-latest`** = floating OpenRouter (se mettent à jour seuls → anti-churn). Le `~` fait partie de l'id (confirmé via l'API `/api/v1/models`). **Fallback pinnés** si un alias casse : `anthropic/claude-opus-4.8`, `openai/gpt-5.5`, `google/gemini-3.5-flash`. **Pas de Haiku** (retiré de l'API + bugs).

## 4. Modifier / ajouter un modèle

1. Ouvrir le node **Code** → bloc `M`.
2. Changer/ajouter une ligne `task_type: 'provider/slug'`.
3. Vérifier le slug sur **`openrouter.ai/models`** (⚠️ churn — les slugs changent à chaque génération).
4. Save → **Publish**.
- Astuce anti-churn : pour rendre une tâche insensible au churn, utiliser `openrouter/auto` (OpenRouter choisit).

## 5. Ajouter un task_type

1. Node Code → ajouter la clé dans `M` (ou laisser tomber sur le défaut).
2. (Si logique spéciale, ex. `private`) ajouter un garde-fou avant le mapping.
3. Save → Publish.

## 6. Coût / clé

- OpenRouter = **service tiers payant** (compte + crédits). Modèles `:free` disponibles pour du gratuit.
- Credential n8n : **"OpenRouter account"** (type OpenRouter). La clé se gère dans le credential — **jamais en clair** dans un node.
- Le « moteur Hermes local gratuit » (coût≈0, LoRA) reste **différé** (besoin d'un GPU non dispo).

## 7. Tester

- **Vrai test = appeler le router depuis un autre workflow** (Maestro) avec un payload conforme au contrat (`routing.task_type`, `brand_profile`, `task_payload`). Vérifier `model` sorti par le Code + la réponse OpenRouter (`choices[0].message.content`).
- Test standalone (pin data du sub-trigger) = possible mais UI capricieuse ; préférer un caller réel.

## 8. Dépannage

- `model not found` → slug périmé → corriger dans `M` (voir `openrouter.ai/models`).
- `401` → credential OpenRouter (clé/ crédits).
- Sortie = requête au lieu de réponse → connexion Code → HTTP rompue (recréer).
- `private` renvoie `{error}` = **normal** (protection PII).

## 9. Dette / à faire

- [ ] Câbler Maestro/agents pour appeler le router (fin du test de routage réel).
- [ ] Système-prompts par rôle (au lieu du fallback « You are a helpful assistant »).
- [ ] Fallback/retry (ex. `openrouter/auto`) + validation croisée (livrables critiques).
- [ ] Brancher la mémoire (RAG pgvector) dans `task_payload.context_raw` avant l'appel.
- [ ] `private`/PII : moteur local (Hermes model) quand GPU dispo.

## 10. Leçons (mémo)

- Passerelle unique (OpenRouter) → 1 clé, 1 endpoint, routeur = mapping + 1 appel (vs N branches/3 clés).
- Slugs de modèles **churnent** → config en 1 bloc, à re-vérifier régulièrement ; jamais de Haiku.
- PII **jamais** au cloud → garde-fou explicite.
- Champs Expression n8n auto-ferment `{{ }}` → coller le code plutôt que taper.
