---
type: raw
title: "POS-GENERIQUE_configuration-obsidian-ios-self-hosted"
source_url: "drive:1WTvEKvpoGv1Hl-6av-E9bqqQrONkBPmx"
captured: 2026-07-07
vault: personal
brand: null
immutable: true
---

---
type: POS
title: "POS générique — Configurer Obsidian sur iPhone avec un coffre self-hosted (Git)"
scope: générique
statut: actif
date: 2026-07-03
---

# POS — Configurer Obsidian sur iPhone (coffre self-hosted, sans Obsidian Sync)

## Contexte

Le coffre Lumina (`ai-automation`, `brands`, `personal`) est un **dépôt Git** (GitHub `frescatzi/ai-automation`), ouvert comme coffre Obsidian sur desktop et synchronisé via le plugin **Obsidian Git**. Il n'y a **pas d'abonnement Obsidian Sync** (le service officiel payant) : la synchronisation est « self-hosted », donc entièrement basée sur Git.

Sur iPhone, le plugin Obsidian Git seul est **instable** (implémentation JavaScript de Git, pas de vrai Git natif). Deux façons fiables de contourner ce problème :

| Option | Coût | Fiabilité | Effort de mise en place |
|---|---|---|---|
| **A — Working Copy + plugin Obsidian Git** | App gratuite (fonctions avancées en achat in-app) | Bonne, éprouvée par la communauté | Moyen (plusieurs apps à relier) |
| **B — GitSync.md** | ~9,99 $ (achat unique) | Bonne, app dédiée et maintenue activement | Faible (tout-en-un) |

**Recommandation** : commencer par **GitSync.md** (option B) — plus simple, un seul point de friction (le paiement unique). Basculer sur Working Copy si un besoin plus avancé apparaît (scripts, automatisations iOS).

---

## Prérequis communs aux deux options

1. **Obsidian** installé depuis l'App Store.
2. Un **Personal Access Token (PAT) GitHub** dédié à cet usage :
   - github.com → Settings → Developer settings → Personal access tokens → Generate new token (fine-grained recommandé).
   - Droits minimum : accès en lecture/écriture au repo `frescatzi/ai-automation` uniquement (pas d'accès à tous les repos).
   - Copier le token immédiatement (il n'est affiché qu'une fois) et le garder dans un gestionnaire de mots de passe — **jamais** dans une note du vault.
3. Savoir quel dossier du repo tu veux ouvrir sur mobile (tout le repo, ou seulement un des 3 vaults `ai-automation` / `brands` / `personal`).

⚠️ Sécurité : ne jamais coller ce token dans une conversation Claude ou un chat.

---

## Option A — Working Copy + plugin Obsidian Git

1. Installer **Working Copy** (App Store).
2. Dans Working Copy : cloner le repo en collant l'URL du dépôt (`https://github.com/frescatzi/ai-automation.git`) plutôt qu'en passant par la connexion GitHub intégrée ; utiliser le PAT comme mot de passe quand demandé.
3. Ouvrir l'app **Fichiers** (Files) → Emplacements → Working Copy → repérer le dossier du repo cloné.
4. Copier ce dossier (ou le sous-dossier du vault voulu) dans **Sur mon iPhone → Obsidian**, pour qu'il apparaisse comme un coffre dans l'app Obsidian.
5. Dans Obsidian : Réglages → Plugins communautaires → activer, puis installer le plugin **Git**.
6. Dans les réglages du plugin Git → section Authentification/Commit Author : renseigner le nom d'utilisateur GitHub et le PAT comme mot de passe.
7. Pour synchroniser : faire le pull/push soit depuis Working Copy directement, soit depuis les commandes du plugin Git dans Obsidian (Commit-and-Sync). En cas d'instabilité du plugin, privilégier Working Copy comme source de vérité pour push/pull, et le plugin Git seulement pour lire l'état.

---

## Option B — GitSync.md (recommandé pour démarrer)

1. Installer **GitSync.md** (App Store, ~9,99 $ achat unique, sans abonnement).
2. Se connecter avec GitHub (OAuth) ou coller le PAT créé plus haut.
3. Cloner le repo `frescatzi/ai-automation` directement dans le dossier Obsidian de l'iPhone — GitSync.md le place automatiquement au bon endroit pour qu'Obsidian le reconnaisse comme coffre.
4. Ouvrir Obsidian : le coffre apparaît sans configuration supplémentaire.
5. Utiliser GitSync.md pour : pull avant de lire/modifier, commit + push après modification. L'app affiche un badge de statut Git sur les fichiers modifiés.

---

## Bonnes pratiques une fois configuré

- **Toujours pull avant d'éditer** sur mobile pour éviter les conflits avec les modifications faites sur desktop.
- Ne pas laisser Obsidian Git (mobile) faire de l'auto-commit en tâche de fond si l'app dédiée (Working Copy / GitSync.md) est déjà responsable du push — un seul outil doit avoir la main sur la synchro à la fois pour éviter les conflits silencieux.
- Respecter la règle du coffre : `raw/` = immuable (on n'édite pas), `wiki/` = seul espace d'écriture régulière depuis mobile (notes rapides, corrections). Voir `Architecture_Connaissance_Obsidian_Centric.md` et `brief-montage-vaults.md` dans le wiki pour les conventions complètes.
- En cas de conflit Git non résolu automatiquement, régler depuis le desktop (Obsidian Git plugin + terminal) plutôt que sur mobile.

---

## Sources consultées (2026-07-03)

- [GitHub — Vinzent03/obsidian-git](https://github.com/Vinzent03/obsidian-git)
- [Obsidian Publish — Git Documentation, Getting Started](https://publish.obsidian.md/git-doc/Getting+Started)
- [Forum Obsidian — Setting Up Obsidian Git on iOS without iSH or Working Copy](https://forum.obsidian.md/t/setting-up-obsidian-git-on-ios-without-ish-or-working-copy/97800)
- [GitSync.md — Real Git for Obsidian and code on iPhone/iPad](https://gitsyncmd.isolated.tech/)
- [GitSync.md — How to Set Up GitSync.md with Obsidian Git on iOS](https://gitsyncmd.isolated.tech/blog/obsidian-git-ios-setup)
- [App Store — GitSync](https://apps.apple.com/us/app/gitsync/id6744980427)
