---
type: wiki
title: Mémoire vivante pour agents (épisodique + consolidation + RAG)
status: active
publish: notion
vault: ai-automation
brand:
sources:
  - raw/2026-07-02--pos-generique-memoire-vivante-agents-episodique-consolidation-rag.md
  - raw/2026-07-06--pos-generique-memoire-vivante-agents-episodique-consolidation-rag.md
  - raw/2026-07-02--lumina-ai-os-passation.md
  - raw/2026-07-02--howto-generique-creer-la-memoire-agents-et-humains.md
  - raw/2026-07-02--sop-creer-la-memoire-agents-et-humains.md
related:
  - wiki/concept-memoire-vectorielle-multi-marques.md
  - wiki/sop/sop-creer-memoire-agents-humains.md
  - wiki/sop/SOP_installer-pgvector-sur-postgres-coolify.md
  - wiki/concept-routeur-multi-llm.md
  - wiki/synthese-lumina-ai-os.md
updated: 2026-07-06
---

# Mémoire vivante pour agents (épisodique + consolidation + RAG)

## Idée centrale

Donner à un système multi-agents une mémoire qui **s'écrit, se consolide et se réinjecte** automatiquement. Repose sur une banque vectorielle (pgvector). Le modèle local (Hermes) opère en continu (coût ≈ 0) ; les modèles premium ne lisent que la mémoire pertinente distillée.

## Les trois types de mémoire

| Type | Contient | `collection` pgvector |
|------|----------|-----------------------|
| **Épisodique** | Chaque exécution, échange, décision (log horodaté) | `episodic` |
| **Sémantique** | Faits stables : voix de marque, personas, produits, patterns | `canon` / `insights` |
| **Procédurale** | Sorties acceptées/éditées/rejetées par moteur → scoring | `skills` |

## Les trois primitives (workflows n8n)

### WRITE — écriture systématique

Sous-workflow `{brand, title, content, collection, …}` → embed → INSERT paramétré idempotent (`ON CONFLICT content_hash DO NOTHING`).

- Exposé en **executeWorkflowTrigger** (fiable en mode queue ; préférer à un webhook qui n'est pas enregistré).
- L'orchestrateur a un outil « Save Episode » ; consigne dans son prompt : *après chaque requête, enregistrer un résumé (demande, décision, résultat)*.
- Gratuit via Hermes → systématique sur 100 % du trafic.

### READ / RAG — réinjection avant action créative

Webhook ou sous-workflow `{brand, question, collection?, limit?}` → embed la question → recherche cosinus.

- Filtre `collection` : si absent, **exclure `episodic` brut** pour ne pas polluer les recherches de référence.
- Résultat injecté dans `task_payload.context_raw` avant l'appel au LLM premium.
- **Best-effort** : ne jamais bloquer si la mémoire est indisponible.
- **Ne jamais router du PII vers le cloud** via RAG.

### CONSOLIDATE — cron nocturne

1. Lit les épisodes récents (`created_at > seuil`).
2. Un LLM en tire 3-5 enseignements.
3. Écrit 1 entrée `collection=insights` datée (ne pas purger — l'historique est la valeur).
4. Résultat : l'épisodique devient sémantique. Fenêtre incrémentale si `created_at` existe en base.

## Boucle procédurale (mémoire qui s'améliore)

- **Feedback scoring (continu)** : Hermes observe le taux d'acceptation par moteur/`task_type` → met à jour les scores de la matrice de routage. Le benchmark devient vivant.
- **Fine-tuning LoRA (tous 1-3 mois)** : Hermes absorbe les meilleures données validées dans ses poids → **actif propriétaire non réplicable**.

## Garde-fous & pièges

1. **Webhooks non enregistrés en mode queue** → préférer `executeWorkflowTrigger` pour WRITE.
2. **Séparer les collections** : exclure `episodic` des recherches de référence par défaut.
3. **Idempotence** via `md5(content)` ; consolidation = 1 entrée datée/nuit pour borner le volume.
4. **Embedding unique** partout (écriture ↔ recherche) — sinon distances incohérentes.
5. **RAG ciblé** (tâches créatives : copy, reasoning, analysis) — pas systématique pour maîtriser le coût en tokens.
6. `created_at` en base → consolidation incrémentale (sinon fenêtre par `id DESC LIMIT N` qui redouble).

## Tester

1. Écrire un épisode → le relire avec `collection=episodic` → présent.
2. Lancer la consolidation manuellement → vérifier l'entrée `insights`.
3. Envoyer à l'orchestrateur une tâche créative → confirmer que la mémoire est injectée dans le prompt.

## Voir aussi

- [[concept-memoire-vectorielle-multi-marques]] — schéma de la banque, registre, multi-marques.
- [[sop/sop-creer-memoire-agents-humains]] — SOP complète de construction de la couche mémoire.
- [[sop/SOP_installer-pgvector-sur-postgres-coolify]] — setup de la base vectorielle.
- [[concept-routeur-multi-llm]] — le routeur lit la mémoire via `task_payload.context_raw`.
- [[synthese-lumina-ai-os]] — Hermes comme hippocampe du système Lumina.
