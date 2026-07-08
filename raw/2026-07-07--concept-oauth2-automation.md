---
type: raw
title: "concept-oauth2-automation"
source_url: "drive:1DLtdVDczVu5xVwKOCZydvQUwMFQEGv8T"
captured: 2026-07-07
vault: ai-automation
brand: null
immutable: true
---

---
type: wiki
title: "OAuth2 — patron universel automation ↔ service cloud"
status: draft
publish: none
vault: ai-automation
brand: null
sources:
  - raw/2026-06-22--concept-configurer-oauth2-automation.md
related:
  - wiki/synthese-oauth2-n8n-google.md
  - wiki/sop/Guide-Connexion-Agents-AI-n8n.md
  - wiki/concept-n8n-credentials.md
updated: 2026-06-29
---

# OAuth2 — patron universel automation ↔ service cloud

## Idée centrale

Connecter un outil d'automation (n8n, Make, Zapier…) à un service cloud (Google, Microsoft, GitHub…) en OAuth2 suit **toujours le même schéma en 3 étapes** : créer une application côté fournisseur, générer Client ID + Client Secret, les coller dans la plateforme. Les erreurs classiques sont prévisibles et diagnosticables à partir du seul code HTTP.

## Les 3 étapes universelles

1. **Créer l'app côté fournisseur** (ex. Google Cloud Console) → écran de consentement OAuth, type *Externe*, mode *Test* → ajouter son email en **utilisateur de test**.
2. **Générer les identifiants** → type *Application Web* → coller l'**URI de redirection** fournie par la plateforme (ex. `https://<domaine>/rest/oauth2-credential/callback`) → récupérer **Client ID** + **Client Secret**.
3. **Lier dans la plateforme** → coller Client ID + Secret → **Sign in** → autoriser le bon compte.

## Les 3 erreurs classiques (et leur cause)

| Erreur | Cause racine | Correctif |
|---|---|---|
| **401 / invalid_client** | Email collé dans le champ Client ID à la place du jeton machine (`….apps.googleusercontent.com`) | Mettre le vrai Client ID technique |
| **403 / access_denied** | App en mode Test, compte **non déclaré** comme testeur | Ajouter l'email dans *Utilisateurs de test* |
| **Redirect URI mismatch** | URL de callback incorrecte (vérif au caractère près, HTTPS, segment `/rest/`) | Recopier l'URI exacte des deux côtés |

## Règle de lecture rapide

- **401** → problème d'**identité de l'application** (Client ID/Secret erroné).
- **403** → problème de **droits** (utilisateur non autorisé ou scope manquant).

## Règles à retenir

- **Client ID ≠ email** : c'est une clé machine-à-machine générée par le fournisseur.
- **Jetons en mode Test** expirent souvent **tous les 7 jours** pour les testeurs externes → ré-authentifier si un flux s'arrête après une semaine. Pour la prod, publier l'app.
- **Vérification URI au caractère près** — l'oubli de `/rest/` dans le callback n8n est un piège fréquent.
- Si le popup connecte le mauvais compte, ouvrir en **navigation privée** pour forcer la sélection de compte.
- Ne jamais exposer le **Client Secret** en clair ; le stocker dans le gestionnaire chiffré de la plateforme (voir [[concept-n8n-credentials]]).

## Exemple concret (Lumina)

La connexion **n8n (`n8n.aftersunpeople.com`) ↔ Google Drive** qui alimente le robot d'ingestion Lumina a utilisé exactement ce patron. Voir [[synthese-oauth2-n8n-google]] pour le détail spécifique Google et [[sop/Guide-Connexion-Agents-AI-n8n]] pour la configuration des credentials dans n8n.
