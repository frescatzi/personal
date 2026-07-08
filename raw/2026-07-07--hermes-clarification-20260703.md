---
type: raw
title: "Hermes_Clarification_20260703"
source_url: "drive:1YVMW6Owd3ZEFxmbmQKnNZZm3ZNXXv0yJ"
captured: 2026-07-07
vault: ai-automation
brand: null
immutable: true
---

# Hermès — Clarification (03.07.2026)

**Statut :** Résolu — confusion identifiée et corrigée, relayée à Maestro et à l'agent AFTRSN-Marketing, sauvegardée en mémoire épisodique.
**Contexte :** Pendant la discussion sur qui gère le monitoring publicitaire + exécution des optimisations approuvées, Karter a mentionné un agent "Hermès" (parfois transcrit "MS" par la dictée vocale). Investigation menée pour identifier ce que c'est réellement.

---

## 1 · La confusion initiale

Trois hypothèses successives, chacune partiellement fausse :

1. **Hypothèse 1** — Hermès serait l'un des agents du roster Maestro (Culture-Steward, Experience-Designer, Secretary, Marketing, Comptable-Finance, Qualité-Compliance). *Faux* : aucun agent de ce nom dans le roster.
2. **Hypothèse 2** (réponse de Maestro, première interrogation) — Hermès serait un sous-outil (`Call_Hermes_Ops_`) rattaché au périmètre du spécialiste **AFTRSN-Marketing**, pour de la recherche/veille marché. *Incomplet* : Maestro et l'agent Marketing n'avaient pas connaissance du système complet ; ils ne voyaient que leur propre point d'accès à Hermès (une fonction qu'ils peuvent appeler), pas ce qu'Hermès est réellement.
3. **Hypothèse 3** (confirmée) — Hermès est une **plateforme d'agent IA autonome séparée**, pas un sous-outil de qui que ce soit dans le roster AFTRSN.

## 2 · Ce qui a été vérifié, et comment

| Étape | Source | Résultat |
|---|---|---|
| 1 | Recherche Notion ("Agents") | Roster périmé (Gardien-marque / Contenu-canaux / Performance-acquisition) — pas de trace de Hermès |
| 2 | Interrogation directe de Maestro (n8n, chat) | Maestro + sous-agent Marketing : Hermès = sous-outil de recherche marché, rattaché à Marketing — réponse incomplète |
| 3 | Recherche Notion ("Hermès") | Une entrée dans le coffre **Security → Knowledge Base → PASSWORDS** : `hermes.aftersunpeople.com`, avec identifiants stockés — confirme l'existence d'un service séparé, mais aucune description fonctionnelle |
| 4 | Recherche workflows n8n ("Hermes") | Deux workflows trouvés : `ZZZ_20260702_tmp_hermes_reach_test` (tagué DEPRECATED — test jetable) et **`LUMINA-Hermes-Exec`** (tagué EXEC/PROD/SHARED, créé le 02.07.2026, dossier `LUMINA-03-BRAIN`) |
| 5 | Lecture du code de `LUMINA-Hermes-Exec` | Chaîne complète : `Prep` → `Embed Task` → `Build Skills Query` → `Skills Search` → **`Hermes Login`** (`POST https://hermes.aftersunpeople.com/api/session/new`) → `Hermes Exec` → `Build Log` → `Log Skills` → `Result` |
| 6 | Visite directe de `hermes.aftersunpeople.com` (session déjà authentifiée dans le navigateur de Karter) | Interface de chat humaine (modèle **Claude Opus 4.6**), historique de conversations mixte (travail AFTRSN réel + tests techniques), bibliothèque **Skills** partagée par catégories (Général, Autonomous-AI-Agents, Creative, etc.) |

## 3 · Ce qu'est réellement Hermès

**Une plateforme d'exécution d'agent IA autonome, partagée entre toutes les marques gérées via LUMINA — pas un outil propre à AFTRSN, et pas un des 7 agents du roster Maestro.**

- **Hébergement :** service séparé à `hermes.aftersunpeople.com`, accessible directement par Karter via une interface de chat (comme ChatGPT/Claude.ai), et aussi appelable par les workflows n8n (dont `LUMINA-Hermes-Exec`, qui sert de client/passerelle).
- **Rangement infra :** dossier n8n `LUMINA-03-BRAIN` — infrastructure commune, pas dans l'espace AFTRSN.
- **Fonctionnement :** on lui donne une tâche → recherche sémantique dans une bibliothèque de **Skills/Runbooks** partagée pour trouver les modes opératoires applicables → ouverture d'une session sur Hermès → exécution → journalisation des skills utilisés (boucle d'apprentissage/amélioration continue).
- **Gouvernance intégrée au code** (pas une simple convention, c'est écrit dans la logique d'exécution elle-même) :
  - Skill **`[autonomous]`** → exécute directement les étapes non critiques.
  - Skill **`[supervised]`** → propose seulement, n'exécute jamais.
  - **Action critique** (dépense, envoi, publication, suppression) → validation explicite de Karter obligatoire, **sans exception**, peu importe le niveau de maturité du skill.

## 4 · Ce qui reste ouvert

Aucun skill dédié "monitoring de performance publicitaire + optimisation" n'a été identifié dans la bibliothèque Skills lors de la visite rapide (catégories Général, Autonomous-AI-Agents et début de Creative parcourues). Deux possibilités :
- le skill existe plus loin dans la bibliothèque, pas encore localisé ;
- il reste à créer.

Ce point n'a pas encore été tranché — l'agent AFTRSN-Marketing, une fois corrigé, a lui-même recommandé de vérifier directement dans la bibliothèque Hermès si un tel skill existe avant de l'assumer.

## 5 · Actions effectuées suite à cette clarification

- Page Notion **🤖 Agents** mise à jour avec le roster réel (Maestro + 6 spécialistes), remplaçant l'ancienne dénomination.
- Maestro informé par compte-rendu complet ; a sauvegardé l'épisode en mémoire centrale (`aftrsn_memory`, collection `episodic`).
- Correction relayée par Maestro à l'agent **AFTRSN-Marketing**, qui a mis à jour sa propre compréhension et confirmé qu'il ne déclenchera jamais d'action critique via Hermès sans validation Karter — garde-fou tenu des deux côtés du système.

## 6 · Prochaine étape suggérée

Aller vérifier, dans la bibliothèque Skills de Hermès, si un runbook "optimisation publicitaire" existe déjà — sinon, le créer, en respectant la règle de gouvernance déjà en place (auto-brouillon + validation humaine avant toute action critique), cohérente avec ce qui a été défini pour le budget et les publicités dans le process métier W-1.
