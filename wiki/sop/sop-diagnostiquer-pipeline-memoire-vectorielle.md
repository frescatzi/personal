---
type: wiki
title: SOP — Diagnostiquer/réparer un pipeline de mémoire vectorielle (n8n + pgvector)
status: active
publish: notion
vault: ai-automation
brand:
sources:
  - raw/2026-07-02--lumina-marche-a-suivre-generique-diagnostiquer-et-reparer-un-pipeline-memoire-vectorielle-n8n-etat-des-lieux-multi-agents.md
related:
  - wiki/concept-memoire-vectorielle-multi-marques.md
  - wiki/concept-memoire-vivante-agents.md
  - wiki/sop/sop-reparer-webhook-n8n-ingestion-pdf.md
  - wiki/sop/sop-ingestion-multi-format-banque-vectorielle.md
  - wiki/sop/SOP_installer-pgvector-sur-postgres-coolify.md
  - wiki/sop/sop-cablage-orchestrateur-subagents.md
updated: 2026-07-06
---

# SOP — Diagnostiquer/réparer un pipeline de mémoire vectorielle (n8n + pgvector)

## Portée

Patron réutilisable (tout pipeline RAG n8n : embeddings OpenAI → pgvector), exposé via webhook ou MCP, avec architecture hub-and-spoke.

## A. Diagnostiquer

### 1. Reproduire et qualifier l'erreur

- Appeler le point d'entrée (MCP/webhook) et noter le message **exact**.
- `NodeApiError` → un node interne plante.
- Réponse vide ou `{}` → le flux va au bout mais renvoie 0 résultats (table vide ou SQL cassé).

### 2. Remonter au node fautif

1. n8n → **Executions** → filtrer les **Error**.
2. Identifier le workflow **réellement** en échec (le serveur MCP peut réussir mais relayer une erreur).
3. Ouvrir l'exécution → lire l'erreur du **node rouge**.

### 3. Suspects classiques (du plus fréquent)

| Cause | Signal |
|-------|--------|
| **Credential morte/rotée** | ID orphelin dans le node |
| **Table vide** (ingestion cassée après TRUNCATE) | 0 résultats alors qu'avant OK |
| **SQL par concaténation** | Erreur sur contenu riche (apostrophes, backticks, markdown) |
| Seuil de similarité trop strict | 0 résultats avec score proche du seuil |
| Dimension d'embedding incohérente | Erreur au niveau de la comparaison cosinus |
| Auth/permissions DB | `permission denied` |

## B. Réparer

### 4. Credential morte

- Ouvrir le node → re-sélectionner la credential dans le dropdown (ne pas se fier au nom affiché : le binding interne peut pointer un ID supprimé).
- **Balayer tous les workflows** qui utilisent la même credential (retrieval ET ingestion).

### 5. Table vide (TRUNCATE-puis-reload interrompu)

- Vérifier l'historique : une ancienne exécution renvoyait-elle des lignes ? Si oui et plus maintenant → table vidée.
- Repérer un node **TRUNCATE** en tête de l'ingestion : tout échec en aval laisse la table vide.
- Réparer la cause amont → relancer l'ingestion → valider avec `SELECT count(*)`.

### 6. Passer aux requêtes paramétrées (règle d'or)

Ne **jamais** construire du SQL par concaténation quand le contenu est riche :

```js
// Node "Build Insert" (Code)
const query =
  "INSERT INTO <table> (colA, colB, embedding) " +
  "VALUES ($1, $2, $3::vector) ON CONFLICT (content_hash) DO NOTHING;";
const values = [valA, valB, '[' + embedding.join(',') + ']']; // vector = literal SANS quotes
return [{ json: { query, values } }];
```

```
// Node "Postgres" (Execute Query)
Query            = {{ $json.query }}
Query Parameters = {{ $json.values }}   // doit résoudre en Array
```

Pièges : activer l'option **Query Parameters** (sinon « there is no parameter $1 ») ; l'éditeur d'expression auto-ferme `{{ }}` (nettoyer les `}}` en trop).

### 7. Valider end-to-end

Toujours re-tester par le **point d'entrée réel** (MCP/webhook), pas seulement au niveau du node.

## C. État des lieux d'un système multi-agents

### 8. Inventaire

- Rechercher par préfixe ; lister les agents et leur **statut** (Published/Active).
- Distinguer les workflows archivés/deprecated.

### 9. Vérifier l'orchestrateur

- Trigger → AI Agent.
- Chat Model (credential valide ? modèle ?).
- Memory (historique conversationnel).
- Tools : outil mémoire (Knowledge) + un `Call-workflow` par sous-agent.

### 10. Vérifier chaque sous-agent

- Double trigger : *chat* (test) + *Executed by Another Workflow* (prod).
- Prompt tolérant : `{{ $json.query || $json.chatInput }}`.
- System message : identité, périmètre, « always query Knowledge first », langue, garde-fou validation humaine.

### 11. Points de contrôle récurrents

1. Cohérence des **noms d'outils** : nom du node-outil = ce que le prompt demande d'utiliser.
2. **Modèle réellement configuré** vs documenté.
3. Chat Memory en sous-workflow : prévoir un fallback `sessionId`.
4. **Langue & garde-fous** dans le prompt.
5. **Qualité du contenu indexé** : la mémoire contient-elle ce que les agents doivent lire ?
6. **Sécurité** : auth MCP ; gating humain sur actions critiques.

## D. Leçons clés

- Erreur opaque → toujours passer par Executions → node rouge.
- Credential affichée OK mais binding mort → réassigner explicitement + balayer tous les workflows.
- TRUNCATE-puis-reload → un échec post-TRUNCATE vide la table → préférer l'idempotence via hash.
- SQL concaténé → casse sur contenu riche → **requêtes paramétrées obligatoires**.
- Validation → end-to-end (appel MCP), pas node par node.

## Voir aussi

- [[concept-memoire-vectorielle-multi-marques]] — schéma banque, registre, multi-marques.
- [[concept-memoire-vivante-agents]] — primitives WRITE/READ/CONSOLIDATE.
- [[sop/sop-reparer-webhook-n8n-ingestion-pdf]] — réparer un webhook last-node + ingérer un PDF.
- [[sop/sop-ingestion-multi-format-banque-vectorielle]] — ingestion texte + PDF + récursion.
- [[sop/SOP_installer-pgvector-sur-postgres-coolify]] — setup de la base vectorielle.
- [[sop/sop-cablage-orchestrateur-subagents]] — audit du câblage orchestrateur.
