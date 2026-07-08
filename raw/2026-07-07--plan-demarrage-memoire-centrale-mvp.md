---
type: raw
title: "Plan_Demarrage_Memoire_Centrale_MVP"
source_url: "drive:1CXQrPr6cWuwoarz4wqHBvzpOy4FEfsnZ"
captured: 2026-07-07
vault: ai-automation
brand: null
immutable: true
---

---
type: wiki
title: "Plan de démarrage — Mémoire Centrale (MVP)"
status: active
publish: notion
vault: ai-automation
brand: null
sources: []
related: ["architecture/Memoire_Centrale_ASP_Brief_Construction", "synthese-lumina-systeme-reference", "sop/SOP_systeme-multi-agents-memoire-centrale-mcp-n8n"]
updated: 2026-06-19
---

# Plan de démarrage — Mémoire Centrale (MVP)

**Date :** 2026-06-19
**Projet :** AFTER SUN PEOPLE — AI Operating System
**Objectif du jour :** poser la **mémoire centrale unique sur le serveur**, vers laquelle on rapatriera ensuite ce qui vit aujourd'hui dans Claude et ChatGPT, et qu'on pourra interroger à la demande.

---

## Décisions CEO actées (on ne re-débat plus)

- **Source of Truth conservée** pour l'instant.
- **Notion = base centrale d'écriture et de référence** ; **n8n pousse vers PostgreSQL (sens unique)**. On **n'édite jamais Postgres à la main**. En cas de doute, **Notion fait foi**.
- **Démarrage avec un petit lot trié** (les docs de ce projet + 2-3 SOP clés), on prouve que la recherche marche, *puis* on élargit. **Pas de migration massive d'un coup.**
- **Un seul modèle d'embedding figé** dès le départ : `text-embedding-3-small` (en changer plus tard = tout ré-encoder).
- **Pas de cycle de vie formel** au stade MVP.
- **Validation :** autonomie par défaut aux agents ; à la création de chaque workflow on marque ce qui est **critique → validation Kater**, le reste → **Superviseur** ou autonome.
- **Approche MVP** : minimum qui marche, on apprend en faisant ; les agents observeront et proposeront les améliorations / workflows au fil du temps.
- **Cadrage du jour :** objectif = **Étape 0 (détection) + Étape 1 (activer pgvector + créer la table)**. Le reste s'enchaîne sur les prochaines sessions.

---

## Architecture MVP de la mémoire (« ASP Memory »)

Une seule mémoire, deux représentations synchronisées :

```text
   ┌────────────────────────┐
   │  NOTION                 │  ← base centrale : tu écris / collectes ici
   │  (référence humaine,    │     « fait foi »
   │   point d'autorité)     │
   └───────────┬─────────────┘
               │  (sens unique)
               ▼
   ┌────────────────────────┐
   │  INGESTION (n8n)        │
   │  texte → découpage →    │
   │  embedding → insertion  │
   └───────────┬─────────────┘
               ▼
   ┌────────────────────────┐
   │  PostgreSQL + pgvector  │  ← mémoire serveur (jamais éditée à la main)
   │  interrogée par n8n     │
   │  et les agents          │
   └───────────┬─────────────┘
               ▼
   ┌────────────────────────┐
   │  RÉCUPÉRATION (n8n)     │
   │  question → embedding → │
   │  recherche sémantique → │
   │  réponse aux agents     │
   └────────────────────────┘
```

**Choix techniques (et pourquoi) :**

- **PostgreSQL + extension `pgvector`** comme mémoire serveur → tu as **déjà** PostgreSQL via Coolify : **aucun nouveau service, aucun coût ajouté**. pgvector permet la recherche sémantique directement dans ta base.
- **Notion** est la **base centrale d'écriture** : n8n lit Notion et pousse vers Postgres (jamais l'inverse). Postgres n'est jamais édité à la main → pas de conflit de synchro.
- **Embeddings** via **HTTP Request** vers une API peu coûteuse (OpenAI `text-embedding-3-small`, ou Gemini). ⚠️ Pas d'embeddings côté Anthropic, et on évite de toute façon le node Anthropic natif (bug 404 du modèle Haiku retiré).

---

## Les étapes (du plus simple au plus complet)

| Phase | But | Statut |
|---|---|---|
| **0 — Détection** | Identifier la version de PostgreSQL et si `pgvector` est disponible | 👉 **à faire aujourd'hui** |
| **1 — Socle** | Activer `pgvector` + créer la table `asp_memory` | à suivre |
| **2 — Ingestion** | 1er workflow n8n : texte → embedding → insertion (tester avec 1 document) | à suivre |
| **3 — Récupération** | 1er workflow n8n : question → recherche sémantique → réponse | à suivre |
| **4 — Doublon + import** | Sync Notion + rapatriement de la connaissance Claude/ChatGPT | à suivre |

---

## ÉTAPE 0 — à faire maintenant (détection de l'environnement)

But : savoir sur quoi on bâtit avant de bâtir (principe « on délègue la détection plutôt que de supposer »). C'est aussi ton **premier contact pratique avec n8n + PostgreSQL**.

1. Dans **n8n**, crée un nouveau workflow vide.
2. Ajoute un node **Postgres** (action « Execute Query ») et connecte-le à ta base (identifiants depuis Coolify : host, port, db, user, password).
3. Exécute cette requête :

```sql
SELECT version();
SELECT * FROM pg_available_extensions WHERE name = 'vector';
```

**Ce qu'on cherche dans le résultat :**
- la **version de PostgreSQL** (ligne `version()`),
- une ligne pour l'extension `vector` : si elle apparaît, pgvector est **installable** ; la colonne `installed_version` te dit si elle est déjà activée ou pas.

4. Copie-moi le résultat (ou dis-moi si tu bloques sur la connexion). Selon ce qu'on voit, on enchaîne sur l'Étape 1 (activer pgvector + créer la table).

> Si pgvector n'est pas disponible dans l'image PostgreSQL actuelle, pas de panique : on basculera ta base Coolify sur l'image `pgvector/pgvector` — je te guiderai.

---

## Version
v1.0 — Plan de démarrage Mémoire Centrale — 2026-06-19
