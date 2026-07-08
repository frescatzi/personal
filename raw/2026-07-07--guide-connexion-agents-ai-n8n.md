---
type: raw
title: "Guide-Connexion-Agents-AI-n8n"
source_url: "drive:1KmUInR8RrXeVZiamZbTIOuGOU1wCXz93"
captured: 2026-07-07
vault: ai-automation
brand: null
immutable: true
---

---
type: wiki
title: "Guide — Connecter Claude, ChatGPT et Gemini dans n8n"
status: active
publish: notion
vault: ai-automation
brand: null
sources:
  - "raw/2026-06-24--guide-connexion-agents-ai-n8n.md"
  - "raw/2026-06-29--guide-connexion-agents-ai-n8n.md"
related:
  - "sop/n8n-Brancher-API-et-Premier-Workflow"
  - "concept-limites-api-claude"
  - "synthese-lumina-systeme-reference"
updated: 2026-06-29
---

# Guide : connecter Claude, ChatGPT et Gemini dans n8n

*Pour les faire dialoguer entre eux et tourner en autonomie, sans PC allumé*

Mis à jour le 18 juin 2026 — pour un n8n **auto-hébergé sur serveur**, en partant **sans aucune clé API**.

---

## 1. L'idée en une image

n8n est un **chef d'orchestre**. Lui-même ne « pense » pas : il appelle les modèles d'IA (Claude, ChatGPT, Gemini) via leurs **API**, et fait circuler les messages de l'un à l'autre.

```
   ┌─────────────────────────────────────────────┐
   │                  n8n (serveur)              │
   │                                             │
   │  Déclencheur → Claude → ChatGPT → Gemini →  │
   │   (horaire,    (rédige) (critique) (synthé-  │
   │    webhook)                        tise)     │
   │                                             │
   │           ↓ résultat (email, fichier,       │
   │             message, base de données…)      │
   └─────────────────────────────────────────────┘
```

« Faire discuter les agents entre eux » = un **workflow** où la sortie d'un modèle devient l'entrée du suivant.

---

## 2. Le point qui change tout : abonnement ≠ API

Votre abonnement Claude, ChatGPT Plus ou Gemini **ne donne pas** automatiquement d'accès API. L'API est un service séparé, avec sa **propre clé** et sa **propre facturation** (au volume de texte échangé, « tokens »).

| Plateforme | Ce qu'on connecte dans n8n | Où créer la clé | Format de la clé |
|---|---|---|---|
| **Claude** (Anthropic) | 1 clé API | platform.claude.com | `sk-ant-…` |
| **ChatGPT** (OpenAI) | 1 clé API | platform.openai.com | `sk-…` |
| **Gemini** (Google) | 1 clé API | aistudio.google.com | `AIza…` |

> **Important sur Claude** — « Chat », « Cowork » et « Code » ne sont **pas** trois connexions séparées. Ce sont trois *interfaces* posées sur le même moteur (Claude). Pour n8n, on connecte **une seule fois l'API Anthropic**. Cowork et Code sont des applis que *vous* utilisez ; elles ne s'exposent pas à n8n.

Chaque compte API démarre à **0 €** et ne facture que ce que vous consommez. Prévoyez tout de même une carte bancaire (obligatoire pour activer l'accès).

---

## 3. Étape A — Créer les 3 clés API

Faites-les dans cet ordre, en collant chaque clé dans un fichier sûr au fur et à mesure (gestionnaire de mots de passe idéalement).

### A.1 — Claude (Anthropic)
1. Aller sur **platform.claude.com** et se connecter (le même compte que votre Claude fonctionne).
2. **Settings → Billing** → ajouter un moyen de paiement et un petit crédit (5–10 € suffisent pour démarrer).
3. **Settings → API Keys → Create Key** → nommez-la (ex. `n8n`).
4. **Copiez-la immédiatement** : Anthropic ne l'affiche **qu'une seule fois**. Si vous la perdez, il faut en recréer une.
5. Modèles disponibles aujourd'hui : **Opus 4.8** (le plus puissant), **Sonnet 4.6** (équilibré, recommandé pour démarrer), **Haiku 4.5** (rapide et économique).

### A.2 — ChatGPT (OpenAI)
1. Aller sur **platform.openai.com**, se connecter.
2. **Settings → Billing** → ajouter un moyen de paiement et un crédit.
3. **API keys → Create new secret key** → nommez-la, copiez-la (affichée une seule fois).

### A.3 — Gemini (Google)
1. Aller sur **aistudio.google.com**.
2. Menu **« Get API key » → Create API key**.
3. Copiez la clé. Gemini propose un palier gratuit, mais ajoutez la facturation si vous voulez éviter les limites.

> **Règle d'or sécurité** : une clé API, c'est comme un mot de passe qui peut dépenser de l'argent. Ne la mettez jamais dans un email, un chat, ou un fichier partagé. Dans n8n, elle se range dans les **Credentials** (chiffrées).

---

## 4. Étape B — Héberger n8n sur votre serveur

Vous avez choisi l'auto-hébergement : c'est la version **gratuite** (Community Edition, sans limite d'exécutions) et **autonome** (tourne 24/7, PC éteint). En échange, il faut un minimum de mise en place technique.

### Ce qu'il vous faut
- Un **VPS** (serveur virtuel) chez un hébergeur : Hetzner, OVH, DigitalOcean, Scaleway… Un petit modèle (~2 Go de RAM, 5–8 €/mois) suffit pour démarrer.
- Un **nom de domaine** (ex. `n8n.mondomaine.com`) — optionnel mais fortement conseillé pour le HTTPS et les webhooks.

### La méthode la plus simple : Docker
n8n se déploie en quelques commandes avec **Docker Compose**. Le schéma type :

1. Installer Docker et Docker Compose sur le VPS.
2. Créer un fichier `docker-compose.yml` qui lance n8n + une base PostgreSQL + un reverse-proxy (Caddy ou Traefik) pour le HTTPS automatique.
3. Lancer `docker compose up -d`.
4. Ouvrir `https://n8n.mondomaine.com`, créer votre compte administrateur.

> n8n publie un guide officiel « Docker Compose » et il existe des images « tout-en-un » (Caddy + n8n + Postgres) qui automatisent les certificats HTTPS. Si la ligne de commande vous intimide, deux raccourcis : (a) certains hébergeurs proposent n8n en **déploiement 1-clic**, (b) on peut faire cette mise en place ensemble, étape par étape, dans une autre session.

### Le minimum à régler
- **Sauvegardes** : activez les snapshots du VPS (au cas où).
- **Variable de fuseau horaire** (`GENERIC_TIMEZONE=Europe/Paris`) pour que les horaires des déclencheurs soient justes.
- **Mises à jour** : `docker compose pull && docker compose up -d` de temps en temps.

---

## 5. Étape C — Brancher les 3 clés dans n8n

Une fois n8n ouvert dans votre navigateur :

1. Menu **Credentials → New**.
2. Cherchez **« OpenAI »** → collez la clé `sk-…` → Save.
3. **New → « Anthropic »** → collez la clé `sk-ant-…` → Save.
4. **New → « Google Gemini (PaLM) »** → collez la clé `AIza…` → Save.

Les credentials sont stockés chiffrés. Vous les sélectionnez ensuite dans les nœuds d'un simple clic, sans jamais recoller la clé.

---

## 6. Étape D — Faire dialoguer les agents

Deux approches, du plus simple au plus puissant.

### Approche 1 — La chaîne (idéale pour débuter)
Vous enchaînez les modèles comme un relais. Exemple « rédaction → critique → synthèse » :

```
[Déclencheur] → [Anthropic: Claude rédige un texte]
              → [OpenAI: ChatGPT critique le texte de Claude]
              → [Google Gemini: synthétise les deux]
              → [Envoi: email / Slack / fichier]
```

Chaque nœud reçoit la sortie du précédent via une référence (dans n8n : `{{ $json.output }}`). C'est tout : c'est ça, « les agents qui discutent ».

### Approche 2 — Le nœud « AI Agent » (plus avancé)
n8n a un nœud **AI Agent** dédié : un modèle « cerveau » qui décide tout seul quelles actions/outils utiliser (recherche web, lecture de fichier, appel d'un autre modèle…). On peut même créer un agent **« orchestrateur »** qui appelle les autres comme des sous-outils. Plus puissant, mais à garder pour la phase 2, une fois la chaîne maîtrisée.

### Conseil de démarrage
Commencez **petit** : un workflow à 2 modèles, déclenché à la main (bouton « Execute »). Vérifiez que les messages passent bien de l'un à l'autre. Ajoutez le 3ᵉ modèle, puis l'automatisation, ensuite seulement.

---

## 7. Étape E — Rendre tout ça autonome (sans être devant l'ordinateur)

C'est là que l'auto-hébergement prend tout son sens : **votre serveur tourne en permanence**, donc le workflow s'exécute même quand votre ordinateur est éteint. Il suffit de remplacer le déclencheur manuel par un **déclencheur automatique** :

| Déclencheur n8n | Se lance quand… | Exemple d'usage |
|---|---|---|
| **Schedule (Cron)** | à heure fixe | « tous les matins à 8h, faire dialoguer les agents et m'envoyer le résultat » |
| **Webhook** | une URL est appelée | un formulaire, un autre logiciel ou un bouton déclenche le workflow |
| **Email / Drive / etc.** | un événement arrive | « quand un email arrive, les agents le traitent » |

Une fois le workflow **activé** (bouton « Active » en haut à droite), n8n s'en occupe seul, 24/7. Vous ne touchez plus rien.

---

## 8. Récapitulatif — votre feuille de route

1. ☐ Créer la clé **Claude** (platform.claude.com)
2. ☐ Créer la clé **ChatGPT** (platform.openai.com)
3. ☐ Créer la clé **Gemini** (aistudio.google.com)
4. ☐ Louer un **VPS** + nom de domaine
5. ☐ Installer **n8n via Docker** (HTTPS + Postgres)
6. ☐ Coller les **3 clés** dans les Credentials de n8n
7. ☐ Construire un **1ᵉʳ workflow à 2 modèles** en manuel
8. ☐ Ajouter le **3ᵉ modèle**
9. ☐ Remplacer le déclencheur par un **Cron** et **activer** le workflow

---

## 9. Réponses directes à vos deux questions

**« Est-ce possible de les connecter et de les faire discuter ? »**
Oui, sans difficulté. n8n est précisément conçu pour orchestrer plusieurs modèles d'IA dans un même workflow, et il a des nœuds prêts à l'emploi pour OpenAI, Anthropic et Gemini. « Discuter entre eux » = enchaîner leurs entrées/sorties.

**« Est-ce possible de le faire en autonomie, sans être devant l'ordinateur ? »**
Oui, **à condition que n8n tourne sur un serveur** (votre choix d'auto-hébergement) — pas en local sur votre PC. Avec un déclencheur horaire (Cron) et le workflow activé, tout s'exécute seul, en continu, machine personnelle éteinte.

---

## 10. Coûts à prévoir (ordre de grandeur)

- **n8n auto-hébergé** : gratuit (le logiciel).
- **VPS** : ~5–10 €/mois.
- **APIs IA** : à l'usage. Quelques euros par mois pour un usage léger ; ça grimpe avec le volume. Surveillez les tableaux de bord de consommation de chaque plateforme et fixez des **plafonds de dépense** (possible chez OpenAI et Anthropic) pour éviter les surprises.

---

*Prochaine étape possible : on peut faire ensemble, pas à pas, soit la création des clés API, soit l'installation de n8n sur le serveur (fichier Docker Compose prêt à copier), soit la construction du premier workflow à 2 agents.*
