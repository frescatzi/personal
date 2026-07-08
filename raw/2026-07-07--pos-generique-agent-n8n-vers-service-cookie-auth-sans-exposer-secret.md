---
type: raw
title: "POS-GENERIQUE_Agent-n8n-vers-service-cookie-auth-sans-exposer-secret"
source_url: "drive:14gVi4G1CjB2UfGIChTgGl6k4zn_2FKWx"
captured: 2026-07-07
vault: ai-automation
brand: null
immutable: true
---

# LUMINA — POS GÉNÉRIQUE — Brancher un agent n8n sur un service web (auth par cookie) sans exposer de secret

Patron réutilisable : donner à un agent n8n un outil qui exécute des actions sur un service web protégé par login+cookie (type WebUI à mot de passe), en gardant le secret hors de portée des nodes Code et du JSON du workflow.

## 1. Principe & sécurité

- **Ne jamais** débloquer l'accès aux variables d'env (`N8N_BLOCK_ENV_ACCESS_IN_NODE=false`) juste pour un mot de passe : ça expose TOUS les secrets du serveur à tout workflow.
- Stocker le secret dans un **credential** (type « Custom Auth » pour injecter dans le corps/headers d'une requête). Les credentials ne sont jamais lisibles par les nodes Code ni les expressions.
- Restreindre le credential aux **Allowed Domains** du service (empêche l'exfiltration).
- Les valeurs non secrètes (URL de base) peuvent rester en dur dans le workflow.

## 2. Architecture (sous-workflow réutilisable)

`Trigger(s)` → `Login (HTTP + credential)` → `Code (cookie → appels métier → polling)`.

- **Login** : HTTP Request POST vers l'endpoint de login, Authentication = Generic → Custom Auth. Le credential contient `{"body":{"password":"…"}}`. Mettre le corps du node en **mode champs nom/valeur** (corps = objet) sinon la fusion du credential n'opère pas. Activer « Full Response » pour lire l'en-tête `set-cookie`.
- **Code** : extraire le cookie (`set-cookie` → `nom=valeur`), puis enchaîner les appels métier en passant l'en-tête `Cookie`. Pour une API asynchrone : **poller** l'endpoint de résultat jusqu'à stabilité (deux lectures identiques) avec un `try/catch` (les early-polls peuvent renvoyer 4xx).

## 3. Exposer comme outil d'agent

Node `toolWorkflow` → le sous-workflow ; entrée principale (ex. `message`/`instruction`) via `{{ $fromAI('…') }}` ; `description` claire (quand l'utiliser, garde-fous). Connexion `ai_tool` → AI Agent.

## 4. Pièges figés

1. **Custom Auth `body` + corps JSON brut = pas de fusion** → corps en mode champs nom/valeur (objet).
2. **Mode queue n8n** (worker séparé) : le Code s'exécute sur le worker ; l'env peut être bloqué/différent → s'appuyer sur les credentials, pas sur `$env`.
3. **Cookies non partagés entre nodes HTTP** → capturer le cookie au login et le repasser manuellement (Code ou en-tête).
4. **Tester par API** : `POST /rest/workflows/:id/run` avec `triggerToStartFrom:{name}` + `pinData` sur le trigger (fiable, contrairement au chat de l'éditeur).
5. Vérifier que le **service cible est authentifié/opérationnel** avant de câbler (un backend en panne renvoie des erreurs d'auth trompeuses).

## 5. Tester (ordre)

1. Sous-workflow seul via `triggerToStartFrom` → vérifier la réponse du service.
2. Agent → outil → service : donner à l'agent une tâche qui force l'appel de l'outil, vérifier la trace + le résultat dans l'exécution.

---
*POS GÉNÉRIQUE — 2026-07-02. Version exacte : POS-AFTRSN Hermes outil Ops. Complète : POS câblage orchestrateur, POS audit/édition n8n par API.*
