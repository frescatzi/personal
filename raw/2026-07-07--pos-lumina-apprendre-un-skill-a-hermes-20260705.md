---
type: raw
title: "POS-LUMINA_Apprendre-Un-Skill-A-Hermes_20260705"
source_url: "drive:1cybX0UYVhQlZG3SN_MrTMxgES60LV3oK"
captured: 2026-07-07
vault: personal
brand: null
immutable: true
---

# POS LUMINA — Apprendre un skill à Hermès (écriture directe dans la bibliothèque)

Procédure vérifiée le 05.07.2026 — premier skill appris à Hermès sur décision Karter (« Hermès est l'exécutant et l'apprenant ; les spécialistes sont là pour le guider ») : skill n° 990 « Recherche d'historique email envoyé (Gmail) ». Générique par nature (architecture LUMINA OS) — pas de paire.

## 1. Principe

La capacité d'Hermès n'est pas définie par ses nodes n8n mais par sa **bibliothèque de skills** : `LUMINA-Hermes-Exec` embedde la tâche reçue, cherche les skills pertinents par similarité (Postgres/pgvector), exécute, puis journalise (`Build Log → Log Skills`). Apprendre un skill à Hermès = **écrire un runbook dans la table mémoire, collection `skills`** — il le trouvera seul dès que la tâche s'y prête. Conforme au cycle de supervision (§1 de l'architecture processus) : les agents/spécialistes poussent leurs connaissances vers Hermès, la bibliothèque s'enrichit des succès comme des échecs.

## 2. Procédure

1. **Rédiger le runbook** au format des skills existants (regarder un skill en place via `search_brand_memory` avant d'écrire) : `RUNBOOK:` (une ligne de définition) / `BUT:` (les usages) / `ETAPES:` numérotées **à partir de 1** / section `AMELIORATION CONTINUE:` (ce qu'Hermès doit consigner à chaque usage) / `REGLE:` (les invariants, ex. lecture seule, jamais d'envoi) / le **cas d'origine** daté comme exemple concret. Y référencer les outils d'exécution concrets (ex. un workflow utilitaire n8n avec son ID).
2. **Écrire via `LUMINA-MEMORY-WRITE/WEBHOOK`** (`zu4jfZbmDz8trQLl`) — malgré son nom, son seul trigger est `Execute Workflow` (pas de webhook) : il faut un **appelant** (node Execute Workflow) avec le contrat exact `{brand, title, content, collection:"skills", knowledge_type:"skill", source, source_ref}`. `brand:"lumina"` pour les skills partagés (les runbooks d'outreach y sont), `"aftrsn"` pour du pur métier de marque. Appelant jetable → nommage `ZZZ_` + décision de suppression (DA) pour Karter.
3. **Vérifier la sortie** : le write renvoie `{ok:true, table, collection}` — contrôler `collection:"skills"`.
4. **Vérifier la trouvabilité** (étape décisive) : interroger `search_brand_memory` avec une **requête formulée comme une vraie tâche** (pas comme le titre du skill). Le skill doit sortir en tête avec une similarité nette au-dessus des voisins — c'est ce test qui prédit qu'Hermès le sélectionnera.

## 3. Difficultés rencontrées

1. `LUMINA-MEMORY-WRITE/WEBHOOK` n'expose pas de webhook malgré son nom — impossible de POSTer directement.
2. Taxonomie du contrat incertaine (`knowledge_type` pour un skill ? quelle `brand` ?) — aucun schéma documenté.
3. Risque de skill « invisible » : un runbook mal formulé peut être sémantiquement loin des requêtes réelles d'Hermès.

## 4. Solutions implémentées

1. Appelant jetable `ZZZ_` avec node Execute Workflow (contrat jsonExample reproduit exactement) ; suppression déléguée à Karter (DA-006).
2. Alignement sur l'existant : symétrie `episodic`/`episode` → `skills`/`skill` ; `brand` déduite de l'emplacement des skills frères (les runbooks d'outreach sont dans `lumina`).
3. Test de trouvabilité post-écriture avec une requête d'usage réel (« Vérifier si un lieu a déjà été contacté par email et retrouver le fil… ») → sorti 1er à 0.66, devant les skills voisins (0.60 et moins).

## 5. Leçons apprises

1. **Enseigner à Hermès = écrire dans la banque, pas câbler du n8n** — le workflow utilitaire n'est qu'un outil que le skill référence.
2. Le nom d'un workflow ne décrit pas son trigger : toujours vérifier le node trigger avant de choisir le mode d'invocation (leçon jumelle du « Respond to Webhook » de la passation W-1).
3. Un contrat non documenté se déduit **des données existantes** (skills frères), pas d'une supposition — et se vérifie par la réponse du write.
4. L'écriture ne suffit pas : un skill n'existe pour Hermès que s'il **remonte en tête d'une recherche formulée comme une tâche**. Le test de trouvabilité fait partie de la procédure, pas de l'option.
