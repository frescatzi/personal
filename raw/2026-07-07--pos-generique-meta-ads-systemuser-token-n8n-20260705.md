---
type: raw
title: "POS-GENERIQUE_Meta-Ads-SystemUser-Token-n8n_20260705"
source_url: "drive:1oJlZM03sTTNq1wUjeZaZ9yJQ-W-9fLaf"
captured: 2026-07-07
vault: ai-automation
brand: null
immutable: true
---

# POS-GÉNÉRIQUE — Token System User Meta Ads pour n8n (création de campagnes)

**Date :** 2026-07-05 (snapshot d'un état vérifié en production)
**Portée :** générique (toute marque gérée via LUMINA OS voulant piloter Meta Ads depuis n8n).
**Objet :** obtenir un **access token System User** longue durée avec `ads_management`, le stocker dans un credential n8n `facebookGraphApi`, et créer des **campagnes en PAUSE** par API. L'agent ne saisit jamais le secret : l'utilisateur crée le credential lui-même.

---

## 1 · Procédure (ordre exact)
1. **App Meta** (developers.facebook.com) : créer une app, cas d'utilisation **« Créer et gérer les publicités avec l'API Marketing »**, la rattacher au **portefeuille business** qui détient le compte publicitaire. Vérifier que `ads_management` apparaît (statut « Prête pour le test » suffit).
2. **Publier l'app (mode Live)** : Tableau de bord → **Publier**. Prérequis pour activer le bouton : **URL de politique de confidentialité** + **catégorie** renseignées dans Paramètres de base. (Aucune vérification d'entreprise requise pour gérer ses propres comptes.)
3. **Utilisateur système** : Business Settings → Utilisateurs système → Ajouter (rôle Employee suffit). Lui **affecter des éléments** : le **compte publicitaire** (permission « Gérer les campagnes ») + la **Page**.
4. **⭐ Donner à l'utilisateur système un RÔLE dans l'app** (l'étape qui débloque tout) : Business Settings → **Applications → [l'app] → Affecter des personnes → sélectionner l'utilisateur système → activer « Gérer l'app » (Contrôle total) → Enregistrer.**
5. **Générer le token** : Utilisateurs système → [user] → **Générer un token** → choisir l'app → expiration **Jamais** → les scopes `ads_management` (+ `business_management`) apparaissent alors → générer. **L'utilisateur copie le token une seule fois.**
6. **Credential n8n** : Credentials → New → **« Facebook Graph API »** (PAS « (App) ») → coller l'access token.
7. **Créer une campagne** : HTTP Request `POST https://graph.facebook.com/v21.0/act_<ID>/campaigns`, credential prédéfini `facebookGraphApi`, corps JSON :
   `{"name":"…","objective":"OUTCOME_TRAFFIC","status":"PAUSED","special_ad_categories":[],"is_adset_budget_sharing_enabled":false}`

## 2 · Difficultés rencontrées
1. Génération de token : **« Aucune autorisation disponible — affectez un rôle d'application à l'utilisateur système »**, même après publication de l'app et affectation du compte pub.
2. Bouton **Publier** grisé.
3. Création de campagne : erreur générique **100 « Invalid parameter »**.
4. Confusion de credential n8n : « Facebook Graph API » vs « Facebook Graph API (App) ».

## 3 · Solutions implémentées
1. Donner le rôle **« Gérer l'app »** à l'utilisateur système via **Applications → [app] → Affecter des personnes** (étape 4). Ni la publication ni l'affectation du compte pub ne suffisent.
2. Renseigner **URL de politique de confidentialité + catégorie** dans Paramètres de base, puis Publier.
3. Lire la **réponse FB complète** (`fullResponse` + `neverError` sur le node HTTP) pour voir `error_user_title` → champ manquant **`is_adset_budget_sharing_enabled`** → l'ajouter (`false`).
4. Prendre **« Facebook Graph API »** (token seul) — c'est le bon ; « (App) » attend un App ID/Secret.

## 4 · Leçons apprises
- Un **token System User** n'expose ses permissions que si l'utilisateur système possède un **rôle dans l'app** (« Gérer l'app »). C'est le point de blocage n°1, non évident.
- L'app doit être **publiée (Live)** ; publier exige **privacy policy URL + catégorie**.
- Pour toute erreur Meta 100 générique, **capturer la réponse FB brute** (`fullResponse`+`neverError`) révèle `error_user_title` — ici l'exigence `is_adset_budget_sharing_enabled` quand la campagne n'utilise pas de budget CBO.
- **Créer les campagnes en `PAUSED`** ; adsets/creatives/budget peuvent rester manuels (Ads Manager) — la coquille campagne automatisée suffit déjà à cadrer la revue humaine.
- Secret **jamais saisi par l'agent** : l'utilisateur crée le credential ; passkey/2FA = humain uniquement.
- Mettre le node Meta en **onError = continue** dans un workflow de contenu : un échec pub ne doit jamais bloquer la génération de contenu.
