---
type: wiki
title: "SOP générique — Runner LLM agentique headless déclenché par webhook"
status: draft
publish: none
vault: ai-automation
brand: null
sources: ["raw/2026-07-07--pos-generique-runner-llm-headless-declenche-par-webhook-2026-07-07.md"]
related: ["sop/sop-generique-pipeline-source-vers-vues", "concept-pipeline-memoire-wiki-git", "sop/sop-lumina-archive-raw-vers-drive", "concept-archivage-n8n-idempotent", "sop/sop-lumina-auto-ingest-raw-vers-wiki"]
updated: 2026-07-07
---

# SOP générique — Runner LLM agentique headless déclenché par webhook

> **Patron réutilisable, hors marque.** Pour automatiser une étape qui a besoin du **jugement d'un LLM agentique** (lire un dépôt, décider, écrire, committer) — trop riche pour un simple appel API stateless. Implémentation concrète chez Lumina : le maillon `raw/ → wiki/` de [[concept-pipeline-memoire-wiki-git]] (étape 3, historiquement manuelle).

---

## 1. Quand utiliser ce patron

- L'étape a besoin d'un **contexte large** (état d'un repo, conventions) + de **décisions** (créer vs mettre à jour, dédup, exclusions) → LLM **agentique** (type Claude Code), pas un simple prompt→texte.
- On veut du **full-auto** déclenché par un événement (push Git, dépôt de fichier, message).
- La sortie écrit dans une **source de vérité** → il faut des garde-fous (brouillon, idempotence, anti-boucle) — cf. [[concept-archivage-n8n-idempotent]] pour le pendant côté archivage.

## 2. Architecture (3 briques)

```
Événement (push/webhook) → Orchestrateur no-code (n8n) → HTTP → Runner LLM headless (conteneur)
                            filtre + anti-boucle           sync source
                            extrait la cible précise        LLM en mode non-interactif
                            appelle le runner (secret)      écrit + commit/push
```

- **Orchestrateur** : reçoit le webhook, filtre (pertinence + anti-boucle), transmet au runner la **cible précise** — ne pas laisser le LLM tout scanner.
- **Runner** : petit service HTTP (`/health`, `/ingest`) qui détient les droits (token, clé LLM), synchronise, lance le LLM ciblé, écrit/commit/push.

## 3. Recommandations — le runner

- Conteneur avec le CLI du LLM agentique + `git` + un serveur HTTP léger.
- **Auth** par secret partagé en header ; sinon 403.
- **Verrou** un seul job à la fois (`busy`) → 202 si occupé.
- **Sync dur** avant traitement (`git reset --hard origin/<branche>`).
- **Ciblage** : injecter dans le prompt la liste précise des éléments changés — plus rapide, moins cher, plus déterministe.
- **Modèle figé** via flag ; milieu de gamme pour la synthèse/compilation, haut de gamme réservé au raisonnement lourd.
- **Écriture** : commit + `pull --rebase` (absorbe un commit concurrent) + push. Le runner doit être le **seul writer** de la cible dérivée.
- Réponse structurée (`ok`/`noop`/`error` + résumé) que l'orchestrateur logue.

## 4. Recommandations — l'orchestrateur

- Réponse **immédiate** au webhook (ne pas bloquer l'émetteur pendant le job LLM).
- Filtre sur la bonne branche + les bons chemins (regex profondeur-agnostique si la source range en sous-dossiers).
- **Anti-boucle indispensable** : le push du runner ne doit pas re-déclencher le job (filtrer par auteur/bot, type de fichier, ou convention `[skip ci]`).
- Timeout large côté HTTP (le job LLM prend des minutes), retry **off** (éviter les doubles runs).

## 5. Garde-fous (écrire dans une source de vérité sans humain)

1. Brouillon par défaut — n'atteint les consommateurs (agents, base vectorielle, doc) qu'après promotion manuelle.
2. Idempotence/dédup — mettre à jour plutôt que dupliquer.
3. Anti-boucle (§4).
4. Verrou + plafond d'éléments par run.
5. Allowlist des cibles autorisées.
6. Immuabilité de la source brute — le runner ne modifie jamais l'entrée, seulement la sortie dérivée.

## 6. Sécurité & secrets

- Secrets uniquement dans l'environnement (conteneur/orchestrateur), jamais dans le repo ni les logs.
- ⚠️ Certaines PaaS (ex. Coolify) passent les variables d'env en **build-args** → exposées dans les logs de build. Après exposition : révoquer + régénérer le token, changer le secret partagé, redéployer, revalider.
- Token à portée minimale (write, seulement sur les dépôts visés).

## 7. Déploiement (conteneur no-git / PaaS type Coolify)

1. Dockerfile auto-suffisant (code inline) → zéro repo dédié pour le runner.
2. Variables d'env (secrets) en runtime.
3. Port exposé, domaine HTTPS, endpoint `/health`.
4. Toujours **Save avant Deploy** ; redéployer après tout changement d'env.
5. Si utilisateur non-root imposé : réutiliser l'utilisateur `node` de l'image, **pas de volume** sur le dossier de travail (sinon conflits de permissions root/non-root).

## 8. Pièges génériques

| Piège | Solution |
|---|---|
| CLI LLM refuse root | user non-root (`node`) + `chown` |
| Volume root-only → app non-root ne peut plus écrire | dossier de travail éphémère, pas de volume |
| Token trop restreint → 403 au push | portée du token = write sur les dépôts visés |
| Changement d'env ignoré | redéployer |
| Source rangée en sous-dossiers | filtre regex profondeur-agnostique |
| Le push du runner se re-déclenche (boucle) | anti-boucle par auteur/type/convention |
| Entrée créée au mauvais format/endroit | valider le format en amont ; intake automatisé (cf. [[concept-intake-source-git]]) |

## 9. Tests de recette

1. Événement valide → orchestrateur déclenche → runner répond `ok` → sortie en brouillon.
2. Idempotence : ré-émettre le même événement → pas de doublon.
3. Anti-boucle : le push du runner ne relance pas de job.
4. Sécurité : mauvais secret → 403.

---

## Voir aussi

- [[sop/sop-generique-pipeline-source-vers-vues]] — pendant générique côté ingestion inbox→store / publication idempotente (pas d'agent LLM autonome, juste des appels LLM stateless).
- [[concept-pipeline-memoire-wiki-git]] — implémentation Lumina : ce patron est la version « auto » de l'étape 3 (`raw/ → wiki/`, Claude Code), historiquement manuelle.
- [[sop/sop-lumina-auto-ingest-raw-vers-wiki]] — implémentation Lumina concrète de ce patron (runner Coolify, secrets, pièges, tests), multi-coffres.
- [[sop/sop-lumina-archive-raw-vers-drive]] — le workflow n8n aval qui consomme la sortie de ce runner (`_archive_queue.json`).
- [[concept-archivage-n8n-idempotent]] — garde-fous équivalents côté archivage.
