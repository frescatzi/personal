---
type: wiki
title: SOP — Brancher un agent n8n sur un service web avec auth par cookie
status: active
publish: notion
vault: ai-automation
brand:
sources:
  - raw/2026-07-02--pos-generique-agent-n8n-vers-service-cookie-auth-sans-exposer-secret.md
  - raw/2026-07-06--pos-generique-agent-n8n-vers-service-cookie-auth-sans-exposer-secret.md
related:
  - wiki/sop/sop-cablage-orchestrateur-subagents.md
  - wiki/sop/sop-audit-edition-n8n-api-interne.md
  - wiki/sop/Guide-Connexion-Agents-AI-n8n.md
updated: 2026-07-06
---

# SOP — Brancher un agent n8n sur un service web avec auth par cookie

## Principe & sécurité

Donner à un agent n8n un outil qui agit sur un service web protégé par login+cookie, **sans jamais exposer le secret** dans les nodes Code, les expressions ou le JSON du workflow.

Règle absolue : **ne pas** activer `N8N_BLOCK_ENV_ACCESS_IN_NODE=false` pour un seul mot de passe — cela expose tous les secrets du serveur à tous les workflows.

## Architecture (sous-workflow réutilisable)

```
Trigger(s) → Login (HTTP Request + Custom Auth credential) → Code (cookie → appels métier → polling)
```

## 1. Stocker le secret : credential Custom Auth

- Type de credential : **Custom Auth** (injecte des données dans le corps ou les headers d'une requête HTTP).
- Contenu : `{"body": {"password": "…"}}`.
- **Allowed Domains** : restreindre au domaine du service cible (empêche l'exfiltration du secret).
- Les credentials ne sont jamais lisibles par les nodes Code ni les expressions.
- L'URL de base (non secrète) peut rester en dur dans le workflow.

## 2. Node Login

- HTTP Request POST vers l'endpoint de login.
- Authentication = `Generic` → `Custom Auth` → sélectionner le credential créé.
- Corps : **mode Champs nom/valeur** (objet) — en mode JSON brut, la fusion du credential n'opère pas.
- Activer **Full Response** pour lire l'en-tête `set-cookie` dans la réponse.

## 3. Node Code — capturer le cookie + appels métier

```js
// Extraire le cookie depuis set-cookie
const setCookie = $input.first().json.headers['set-cookie'];
const cookie = setCookie.split(';')[0]; // format "nom=valeur"

// Appels métier avec le cookie
const result = await $http.request({
  method: 'POST',
  url: 'https://service.example.com/api/action',
  headers: { Cookie: cookie },
  body: { instruction: $json.message }
});

// Polling pour API asynchrone (deux lectures identiques = stable)
// Entourer d'un try/catch : les early-polls peuvent renvoyer 4xx
```

Les cookies ne sont **pas partagés automatiquement** entre les nodes HTTP Request — il faut les capturer au login et les repasser manuellement.

## 4. Exposer comme outil d'agent

- Node `toolWorkflow` pointant ce sous-workflow.
- Entrée principale (`message` ou `instruction`) via `{{ $fromAI('message', 'description de ce que l\'agent doit envoyer') }}`.
- `description` du node : quand appeler cet outil, garde-fous (ex. « actions irréversibles = validation humaine »).
- Connexion `ai_tool` → AI Agent de l'orchestrateur.

## 5. Pièges

| Piège | Fix |
|-------|-----|
| Custom Auth `body` + corps JSON brut → fusion échoue silencieusement | Passer le corps en **mode champs nom/valeur** (objet) |
| Mode queue n8n (worker séparé) : `$env` bloqué/différent | S'appuyer sur les credentials, jamais sur `$env` |
| Cookies non partagés entre nodes HTTP | Capturer dans Code, repasser manuellement |
| Service cible hors ligne → erreurs d'auth trompeuses | Vérifier le backend avant de câbler |

## 6. Tester

1. **Sous-workflow seul** : `POST /rest/workflows/:id/run` avec `triggerToStartFrom:{name}` + `pinData` sur le trigger → vérifier la réponse du service.
2. **Agent → outil → service** : donner à l'agent une tâche forçant l'appel de l'outil ; vérifier la trace et le résultat dans Executions.

## Voir aussi

- [[sop/sop-cablage-orchestrateur-subagents]] — câbler un outil (toolWorkflow) à l'orchestrateur.
- [[sop/sop-audit-edition-n8n-api-interne]] — tester par API interne (`triggerToStartFrom`).
- [[sop/Guide-Connexion-Agents-AI-n8n]] — créer et gérer les credentials dans n8n.
