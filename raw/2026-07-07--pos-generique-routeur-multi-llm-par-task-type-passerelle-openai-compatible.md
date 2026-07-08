---
type: raw
title: "POS-GENERIQUE_Routeur-multi-LLM-par-task_type-passerelle-OpenAI-compatible"
source_url: "drive:1X7bxUDQ0Zzt17jjHd84YN52O6PWomdW4"
captured: 2026-07-07
vault: ai-automation
brand: null
immutable: true
---

# POS GÉNÉRIQUE — Runbook : routeur multi-LLM par `task_type` (passerelle OpenAI-compatible)

Procédure Opérationnelle Standardisée (réutilisable, indépendante de la marque). Cible : n8n + une passerelle OpenAI-compatible (OpenRouter, ou endpoint local) exposant plusieurs modèles via une clé unique.

Principe : on ne colle pas un agent à un seul LLM. Un routeur choisit le moteur selon la nature de la tâche.

## 1. Architecture (3 nodes)

`Trigger (Execute by Another Workflow, Accept all data)` → `Code (mapping)` → `HTTP Request (passerelle)`.

- Contrat d'entrée (JSON) : `routing.task_type`, `brand_profile` (voix, mots interdits, kb_ref), `task_payload` (context_raw, instruction).
- Code (mapping) : `task_type` → slug modèle ; construit `messages:[{system},{user}]` ; garde-fous (ex. PII → stop). Sort `{ model, messages }`.
- HTTP : `POST <gateway>/chat/completions`, credential unique, body Expression `{{ JSON.stringify({ model: $json.model, messages: $json.messages }) }}`.

## 2. Table de routage (par nature/critère dominant)

- Rédiger / raisonner / nuancer / orchestrer → modèle « rédacteur » (ex. Claude Sonnet).
- Idéation / défaut → modèle « généraliste-agent » (ex. GPT).
- Recherche / synthèse volume / multimodal / classify / extract / temps réel → modèle « rapide & bon marché » (ex. Gemini Flash).
- PII / données sensibles → local uniquement ; si pas de local sûr → stop (jamais au cloud).

## 3. Procédure « ajouter/modifier un modèle »

1. Node Code → bloc de mapping → éditer 1 ligne `task_type: 'provider/slug'`.
2. Vérifier le slug sur la page « models » de la passerelle (les slugs churnent).
3. Save → Publish.

- Option anti-churn : slug « auto » de la passerelle (elle choisit le modèle).

## 4. Coût & clé

- Passerelle = souvent payante (prix modèle + marge) ; vérifier l'existence de modèles gratuits.
- Une seule clé/credential pour tous les modèles. Clé jamais en clair dans un node.
- Alternative « gratuite » : servir un modèle open-weight local (endpoint OpenAI-compatible) — nécessite du compute (GPU).

## 5. Tester

- Vrai test = appeler le routeur depuis un caller (agent orchestrateur) avec un payload conforme. Vérifier le `model` choisi + la réponse (`choices[0].message.content`).
- Éviter le pin-data manuel du sub-trigger (fragile).

## 6. Garde-fous & dépannage

- `model not found` → slug périmé.
- `401` → clé/crédits.
- Sortie = requête au lieu de réponse → lien Code→HTTP rompu.
- Ne jamais router du PII vers le cloud → garde-fou explicite dans le Code.
- Validation croisée (2ᵉ modèle) uniquement pour livrables externes/critiques (sinon coût x2).

## 7. Leçons (mémo)

- Passerelle unique >> N branches (1 clé, maintenance en 1 bloc).
- Slugs churnent → config centralisée + revue régulière.
- Deux couches orthogonales : agents métier (QUI) vs routeur (QUEL modèle) ; elles se composent.
- Champs Expression n8n auto-ferment `{{ }}` → coller, pas taper.
