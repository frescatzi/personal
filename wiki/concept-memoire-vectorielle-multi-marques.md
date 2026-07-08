---
type: wiki
title: Mémoire vectorielle multi-marques (banque partagée + tables par marque)
status: active
publish: notion
vault: ai-automation
brand:
sources:
  - raw/2026-07-02--lumina-marche-a-suivre-generique-concevoir-une-memoire-vectorielle-multi-marques-banque-partagee-tables-par-marque.md
  - raw/2026-07-02--lumina-ai-os-passation.md
  - raw/2026-07-02--lumina-playbook-v1-2026-07-02.md
related:
  - wiki/concept-memoire-vivante-agents.md
  - wiki/sop/SOP_installer-pgvector-sur-postgres-coolify.md
  - wiki/sop/sop-diagnostiquer-pipeline-memoire-vectorielle.md
  - wiki/sop/sop-clonage-roster-agents.md
  - wiki/synthese-lumina-ai-os.md
updated: 2026-07-06
---

# Mémoire vectorielle multi-marques (banque partagée + tables par marque)

## Idée centrale

Séparer **deux natures de connaissance** par une **frontière physique**, pas par un simple filtre :
- **Savoir inter-entités** (process, solutions, patterns) → **banque partagée** (ex. `lumina_memory`).
- **Savoir propre à une entité** (canon, identité, ops) → **table par entité** (`<code>_memory`).

Un filtre oublié = fuite de contexte entre marques. La frontière physique supprime cette classe de bug par construction.

## Quand choisir « une table par marque »

Retenir ce pattern quand au moins un critère tient :
- **Horizon long** (années) — clarté de maintenance.
- **Volumes importants** — perf de recherche + purge/backup par entité indépendants.
- **Stack entier spécifique** par entité — la mémoire doit suivre la même logique.

Sinon (peu d'entités, faible volume) : une table + tag d'entité peut suffire — mais c'est un choix d'économie, pas de clarté.

## Schéma type d'une banque

Colonnes minimales :

```sql
id, title, content,
collection,        -- sous-catégorie : canon | episodic | insights | docs | skills
knowledge_type,    -- standard | procedural | ...
source, source_ref,
content_hash UNIQUE,  -- md5(content) → idempotence upsert
embedding VECTOR(d),
created_at
```

Index : **HNSW cosine** sur `embedding` ; btree sur `collection` / `knowledge_type`.

**Règle :** si la frontière est la table par entité, **pas besoin de colonne d'entité** (la table = l'entité). Schéma **identique** entre toutes les banques → migrations et templates triviaux.

## Sous-catégorisation interne (collections)

Frontières **dures** = la table (entité). Frontières **douces** = tags `collection` à l'intérieur :

| `collection` | Contenu |
|---|---|
| `canon` | Bible de marque, voix, personas, mots interdits (figé) |
| `episodic` | Traces d'exécution horodatées (brut) |
| `insights` | Connaissances consolidées (distillées du brut) |
| `docs` | Documents de référence ingérés |
| `skills` | Patterns appris, procédures validées |

Filtrer `collection` dans les recherches : exclure par défaut `episodic` pour ne pas polluer les recherches de référence.

## Registre de routage (`memory_registry`)

Table de config = source unique de vérité :

```sql
memory_registry(code, table_name, display_name, kind, active)
```

Les workflows reçoivent un **code** → résolvent la **table** via le registre. **Sécurité :** valider le nom de table contre le registre (allowlist) avant tout SQL. Ne jamais interpoler un nom de table arbitraire → risque d'injection SQL.

## Workflows paramétrés (éviter N pipelines)

Un **modèle d'ingestion** et un **modèle de recherche** paramétrés par la table résolue via registre. Ajouter une entité = paramétrer/cloner, pas reconstruire. Idéalement 1 workflow unique qui route selon `code`.

## Câblage des agents

- Agents propres à une entité → banque de cette entité.
- Agents transverses (dev/ops, skills) → banque partagée.
- Certains croisent les deux (deux outils `search`).

Recherche : k-NN `ORDER BY embedding <=> q LIMIT k`, éventuellement filtré par `collection`.

## Conventions de nommage

- Banque partagée : `<infra>_memory` (ex. `lumina_memory`).
- Banque par entité : `<code>_memory` (code lowercase court et stable).
- Outil MCP/n8n : 1 outil routé `search_entity_memory(entity, question)` pour toutes les entités.

## Pièges & leçons

- **Mémoire mélangée sans filtre** → l'agent d'une marque est noyé par le hors-sujet. Frontière physique d'abord.
- **Table par entité = risque N pipelines** → templates paramétrés + registre + outil routé.
- **Nom de table paramétré** → allowlist via registre, jamais d'interpolation directe.
- **Inserts** → requêtes paramétrées (`$1..$n`), jamais de concaténation.
- **Noms d'outils** → identiques entre node n8n et prompts des agents.

## Voir aussi

- [[concept-memoire-vivante-agents]] — le cycle WRITE/READ/CONSOLIDATE sur cette banque.
- [[sop/SOP_installer-pgvector-sur-postgres-coolify]] — installation pgvector, index HNSW.
- [[sop/sop-diagnostiquer-pipeline-memoire-vectorielle]] — réparer un pipeline mémoire cassé.
- [[sop/sop-clonage-roster-agents]] — provisionner une nouvelle banque marque + cloner les agents.
- [[synthese-lumina-ai-os]] — le contexte système Lumina.
