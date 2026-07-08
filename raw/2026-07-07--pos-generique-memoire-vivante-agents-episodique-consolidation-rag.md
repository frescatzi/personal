---
type: raw
title: "POS-GENERIQUE_Memoire-vivante-agents-episodique-consolidation-RAG"
source_url: "drive:1jFmZkXskXcr6uM1ceCY--e67O5vBKqvL"
captured: 2026-07-07
vault: ai-automation
brand: null
immutable: true
---

# LUMINA — POS GÉNÉRIQUE — Mémoire « vivante » pour agents (épisodique + consolidation + RAG)

Patron réutilisable, indépendant de la marque : donner à un système multi-agents une mémoire qui s'écrit, se consolide et se réinjecte. Repose sur une banque vectorielle (pgvector) déjà en place.

## 1. Modèle de données (une banque, plusieurs `collection`)

Une table par marque/domaine avec `title, content, collection, knowledge_type, source, source_ref, content_hash (md5 → idempotence), embedding`. Séparer par `collection` : `canon` (référence figée), `episodic` (traces d'exécution), `insights` (consolidés), `docs`. Un même modèle d'embedding partout.

## 2. Trois primitives

- **WRITE** : sous-workflow `{brand,title,content,collection,…}` → embed → INSERT paramétré idempotent. Exposé en **executeWorkflowTrigger** (appelé comme outil par les agents) — plus fiable qu'un webhook si l'instance est en mode queue.
- **READ** : webhook/sous-workflow `{brand, question, collection?, limit?}` → embed question → recherche cosinus. Filtre `collection` : si absent, **exclure les épisodes bruts** pour ne pas polluer les recherches de référence.
- **CONSOLIDATE** : cron nocturne → lit les épisodes récents → un LLM en tire 3-5 enseignements → écrit 1 entrée `insights` datée (garder l'historique, ne pas purger).

## 3. Boucle mémoire

1. **Écrire** : l'orchestrateur a un outil « Save Episode » et une consigne de prompt : après chaque requête traitée, enregistrer un résumé (demande, décision, résultat).
2. **Consolider** : la nuit, condenser les épisodes en insights réutilisables.
3. **Réinjecter (RAG)** : avant un appel LLM créatif (rédaction/analyse/raisonnement), récupérer la mémoire pertinente et la préfixer au contexte. Best-effort (ne jamais bloquer si la mémoire est indisponible). Ne jamais réinjecter/router de PII vers le cloud.

## 4. Garde-fous & pièges

1. **Webhooks API non enregistrés en mode queue** → préférer `executeWorkflowTrigger`.
2. **Séparer collections** : épisodes bruts hors des recherches de référence (filtre par défaut).
3. **Idempotence** via `md5(content)` ; consolidation = 1 entrée datée/nuit pour borner le volume.
4. **Embedding unique** partout (sinon distances incohérentes).
5. **RAG ciblé** (types de tâches créatives) plutôt que systématique → maîtrise du coût en tokens.
6. Prévoir un `created_at` en base pour une consolidation **incrémentale** (sinon fenêtre par `id DESC LIMIT N`).

## 5. Tester

Écrire un épisode → le relire avec le filtre `collection=episodic` ; lancer la consolidation manuellement → vérifier l'entrée `insights` ; envoyer à l'orchestrateur une tâche créative → confirmer que la mémoire est injectée dans le prompt.

---
*POS GÉNÉRIQUE — 2026-07-02. Version exacte : POS-AFTRSN mémoire épisodique+consolidation+RAG. Brique du futur LUMINA-PLAYBOOK.*
