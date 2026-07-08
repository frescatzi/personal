---
type: raw
title: "synthese-oauth2-n8n-google"
source_url: "drive:1HVJ-4dRYGZiIP6Bp_8qqTFrifS1Nunu1"
captured: 2026-07-07
vault: ai-automation
brand: null
immutable: true
---

---
type: wiki
title: Configurer OAuth2 Google dans n8n
status: active
publish: none
vault: ai-automation
brand:
sources:
  - raw/2026-06-22--sop-configuration-oauth2-n8n-google.md
related:
  - wiki/concept-oauth2-automation.md
  - sop/Guide-Connexion-Agents-AI-n8n
  - sop/sop-lumina-intake-et-publish
updated: 2026-06-29
---

# Configurer OAuth2 Google dans n8n

## Idée centrale

Connecter n8n (auto-hébergé) aux API Google Workspace (Drive, Gmail…) passe par un flux OAuth2 standard côté Google Cloud Console, mais la quasi-totalité des échecs viennent de deux confusions évitables : coller un email à la place d'un Client ID, et oublier de déclarer un compte comme testeur tant que l'app Google Cloud reste en mode Test.

## Points utiles

- **Flux** : créer un projet GCP → configurer l'écran de consentement OAuth (type *Externe*, mode *Test*) → créer un identifiant **OAuth 2.0 Application Web** avec comme URI de redirection exacte `https://<domaine-n8n>/rest/oauth2-credential/callback` → coller Client ID / Client Secret dans le credential n8n du nœud → "Sign in with Google".
- **Erreur 401 (`invalid_client`)** = toujours un problème d'identité de l'app : le piège classique est de coller l'email du compte Google à la place du vrai Client ID (`....apps.googleusercontent.com`).
- **Erreur 403 (`access_denied`)** = toujours un problème de droits : en mode Test, seuls les emails ajoutés en **Utilisateurs de test** sur l'écran de consentement peuvent s'authentifier.
- **Erreur de redirection** : Google vérifie l'URI au caractère près — l'omission du segment `/rest/` est une cause fréquente d'échec sur les versions récentes de n8n.
- **Astuce navigation privée** : si le pop-up Google s'auto-connecte au mauvais compte, ouvrir n8n en fenêtre privée force Google à redemander les identifiants.
- **Expiration en mode Test** : tant que l'app n'est pas publiée/vérifiée par Google, les refresh tokens des utilisateurs de test externes expirent généralement tous les **7 jours** → ré-authentification périodique à prévoir si un workflow s'arrête sans raison apparente après une semaine.

## Règle de lecture des erreurs (réutilisable au-delà de n8n)

- **401** → problème d'identité de l'application (mauvais Client ID/Secret).
- **403** → problème de droits/permissions (utilisateur non autorisé, scope manquant).

## Limites / conditions d'usage

- Procédure documentée pour une instance n8n auto-hébergée (`n8n.aftersunpeople.com`) ; l'URI de callback est à adapter au domaine réel.
- La limite des 7 jours de refresh token est spécifique au mode Test non vérifié — disparaît une fois l'app publiée/vérifiée par Google (processus non couvert par cette source).
- Ne jamais committer le Client Secret en clair ; utiliser les variables d'environnement n8n en équipe.

## Voir aussi

- [[concept-oauth2-automation]] — patron OAuth2 universel (indépendant de n8n et de Google) : 3 étapes, 3 erreurs, règles de sécurité.
- [[concept-n8n-credentials]] — gestion des credentials n8n (API Key, OAuth2, rotation).
- [[sop/Guide-Connexion-Agents-AI-n8n]] — configurer les credentials Anthropic/OpenAI/Gemini dans n8n (même contexte d'intégration).
- [[sop/sop-lumina-intake-et-publish]] — le robot d'intake Lumina utilise Google Drive avec ces mêmes credentials OAuth2.
