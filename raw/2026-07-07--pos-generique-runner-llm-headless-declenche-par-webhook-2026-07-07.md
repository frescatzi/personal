---
type: raw
title: "POS-GENERIQUE_Runner-LLM-headless-declenche-par-webhook_2026-07-07"
source_url: "drive:1Vatb1b_AqmhZaQY1yDG6OWWq7Wfhi_vF"
captured: 2026-07-07
vault: ai-automation
brand: null
immutable: true
---

# POS GÉNÉRIQUE — Automatiser une étape LLM (transformation/compilation) déclenchée par webhook, via un runner headless

> **Patron réutilisable, hors marque.** Cas d'usage : une étape de votre pipeline nécessite le **jugement d'un LLM agentique** (lire un dépôt, décider, écrire des fichiers, committer) — trop riche pour un simple appel API stateless. On l'automatise sans intervention grâce à un **runner LLM headless conteneurisé**, déclenché par un **webhook** et orchestré par un outil **no-code** (n8n, Make…).
> **Exemple d'implémentation Lumina :** compilation `raw/ → wiki/` — voir `POS-LUMINA_Auto-Ingest-Raw-vers-Wiki`.

---

## 1. Quand utiliser ce patron

- L'étape demande un **contexte large** (état d'un repo, conventions) et des **décisions** (créer vs mettre à jour, dédup, exclusions) → un LLM **agentique** (type Claude Code) plutôt qu'un appel API « prompt → texte ».
- On veut du **full-auto** déclenché par un événement (push Git, dépôt de fichier, message).
- La sortie écrit dans une **source de vérité** → il faut des **garde-fous** (brouillon, idempotence, anti-boucle).

---

## 2. Architecture (3 briques)

```
Événement (push / webhook)  ──▶  Orchestrateur no-code (n8n)  ──HTTP──▶  Runner LLM headless (conteneur)
                                  · filtre / anti-boucle                 · sync source
                                  · extrait la cible (quoi traiter)      · lance le LLM agentique (ciblé)
                                  · appelle le runner (secret)           · écrit + commit/push
```

- **Orchestrateur** : reçoit le webhook, **filtre** (pertinence + anti-boucle), transmet au runner la **cible précise** (ne pas faire tout scanner au LLM).
- **Runner** : petit service HTTP (`/health`, `/ingest`) qui synchronise la source, **lance le LLM en mode non-interactif** sur la cible, puis écrit/commit/pousse. C'est lui qui détient les droits (token, clé LLM).

---

## 3. Le runner (recommandations)

- **Conteneur** avec : le CLI du LLM agentique, `git`, un petit serveur HTTP (Node/Express, Python/FastAPI…).
- **Auth** : chaque appel porte un **secret partagé** (header) ; rejeter sinon (403).
- **Verrou** : un seul job à la fois (`busy`) → répondre 202 si occupé.
- **Sync dur** avant traitement : `git reset --hard origin/<branche>` (état miroir).
- **Ciblage** : injecter dans le prompt **la liste des éléments changés** (« ne traite que ceux-ci ») → plus rapide, moins cher, plus déterministe.
- **Modèle** : choisir selon la tâche. Pour de la **synthèse/compilation**, un modèle **milieu de gamme** (rapide, bon marché) suffit ; réserver le haut de gamme au raisonnement lourd. **Figer** le modèle via un flag.
- **Écriture puis push** : `commit` + `pull --rebase` (absorber un commit concurrent) + `push`. Le runner doit être le **seul writer** de la cible dérivée.
- **Réponse** : renvoyer un statut structuré (`ok` / `noop` / `error` + résumé) que l'orchestrateur logue.

---

## 4. L'orchestrateur (recommandations)

- **Webhook** en réponse **immédiate** (ne pas faire attendre l'émetteur pendant le job LLM).
- **Filtre** : ne déclencher que sur la bonne branche + les bons chemins ; **regex profondeur-agnostique** si la source range par sous-dossiers.
- **Anti-boucle indispensable** : le push du runner lui-même ne doit **pas** re-déclencher le job (filtrer par auteur/bot, par type de fichier, ou par convention de commit `[skip ci]`).
- **Timeout large** côté appel HTTP (le job LLM prend des minutes) ; **retry off** (éviter les doubles runs).
- Transmettre au runner **la cible** (repo, fichiers) dans le corps.

---

## 5. Garde-fous (écrire dans une source de vérité sans humain)

1. **Brouillon par défaut** : produire en `draft` / non publié → n'atteint les consommateurs (agents, base vectorielle, doc) qu'après **promotion manuelle**.
2. **Idempotence / dédup** : ne pas recréer si le contenu existe déjà (empreinte) ; mettre à jour plutôt que dupliquer.
3. **Anti-boucle** (cf. §4).
4. **Verrou + plafond** (nombre d'éléments par run).
5. **Allowlist** des cibles autorisées (refuser tout le reste).
6. **Immuabilité de la source brute** : le runner ne modifie jamais l'entrée, seulement la sortie dérivée.

---

## 6. Sécurité & secrets

- Secrets **uniquement** dans l'environnement du conteneur / l'orchestrateur, **jamais** dans le repo ni les logs.
- ⚠️ Certaines plateformes (ex. Coolify) passent les variables d'env en **build-args** → **exposées dans les logs de build**. Après toute exposition : **révoquer + régénérer** le token, changer le secret partagé, **redéployer**, revalider.
- Token à **portée minimale** (juste les droits d'écriture nécessaires, sur les seuls dépôts visés).

---

## 7. Déploiement (conteneur no-git / PaaS type Coolify)

1. Écrire un **Dockerfile auto-suffisant** (embarque le code inline) → déploiement « Dockerfile sans Git », zéro repo pour le runner.
2. Renseigner les **variables d'env** (secrets) — runtime.
3. Exposer le **port** de l'app, un **domaine** HTTPS, un **`/health`**.
4. **Toujours “Save” avant “Deploy”** ; **redéployer** après tout changement d'env.
5. Pour un utilisateur non-root imposé par l'outil : réutiliser l'utilisateur **`node`** de l'image, **pas de volume** sur le dossier de travail (sinon conflits de permissions root/non-root).

---

## 8. Pièges génériques (transposables)

| Piège | Solution |
|---|---|
| CLI LLM refuse root | user non-root (`node`) + `chown` |
| Volume root-only → l'app non-root ne peut plus écrire | dossier de travail éphémère (pas de volume) |
| Droits d'écriture insuffisants du token → 403 au push | portée du token = write sur les dépôts visés |
| Changement d'env ignoré | redéployer |
| Source qui range en sous-dossiers | filtre regex profondeur-agnostique |
| Le push du runner se re-déclenche (boucle) | anti-boucle par auteur/type/convention |
| Entrée créée sans la bonne extension / au mauvais endroit | valider le format en amont ; l'intake automatisé évite ça |

---

## 9. Tests de recette (canevas)

1. Émettre un événement valide → l'orchestrateur déclenche → le runner répond `ok` → sortie produite (en brouillon).
2. **Idempotence** : re-émettre le même → pas de doublon.
3. **Anti-boucle** : le push du runner ne relance pas de job.
4. **Sécurité** : mauvais secret → 403.

---

*POS GÉNÉRIQUE — Runner LLM headless déclenché par webhook — 2026-07-07. Version exacte / implémentation : `POS-LUMINA_Auto-Ingest-Raw-vers-Wiki`. Double sauvegarde : projet Claude « Claude Lessons » + Google Drive (règle CLAUDE.md).*
