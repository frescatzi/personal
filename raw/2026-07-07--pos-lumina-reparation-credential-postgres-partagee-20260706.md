---
type: raw
title: "POS-LUMINA_Reparation-Credential-Postgres-Partagee_20260706"
source_url: "drive:1w2RZm3gbbFOYaUZdKml5PKAIYHwUKKDD"
captured: 2026-07-07
vault: ai-automation
brand: null
immutable: true
---

# POS-LUMINA — Réparation d'une credential Postgres partagée (mémoire + agents)

**Portée :** LUMINA OS (infrastructure). Procédure réutilisable pour toute marque hébergée. **À pousser au canon** (`wiki/` → collection `canon`) : c'est un runbook d'infra critique.
**Contexte d'origine :** 05.07.2026 — panne totale du « cerveau » (mémoire + tous les agents) suite à la corruption d'une unique credential Postgres.
**Date :** snapshot 06.07.2026 (état vérifié).

---

## 1 · Symptôme

Erreur runtime sur tout node Postgres (`n8n-nodes-base.postgres`, `memoryPostgresChat`) :
```
Credential with ID "<id>" does not exist for type "postgres".
```
La credential **apparaît dans la liste** (`/rest/credentials`) mais :
- `GET /rest/credentials/<id>` → **404** ;
- dans l'UI n8n : « **Problem loading credential — could not be found** ».

C'est une **credential fantôme** : métadonnées listées, entité (blob chiffré) illisible. Un redémarrage Docker ne la restaure PAS (le défaut est dans la donnée, pas dans le process).

## 2 · Rayon de la panne (à mesurer d'abord)

Une seule credential Postgres est typiquement câblée dans **beaucoup** de workflows (ici 24, dont 17 actifs) : écriture mémoire, retrieval, ingestion, consolidation, et **la Chat Memory de tous les agents**. Donc quand elle tombe, **tout le cerveau tombe** — y compris Maestro/Hermès (ils plantent au node Chat Memory avant de répondre).

Scanner l'impact :
```js
const OLD='<id>';
const list=(await (await fetch('/rest/workflows?limit=250',{headers})).json()).data;
for(const wf of list){ const w=(await (await fetch('/rest/workflows/'+wf.id,{headers})).json()).data;
  const hit=w.nodes.filter(n=>n.credentials?.postgres?.id===OLD).map(n=>n.name);
  if(hit.length) console.log(w.active?'ACTIF':'inactif', w.name, hit); }
```

## 3 · Diagnostic — ce n'est PAS la base de données

L'erreur est `functionality:'configuration-node'`, levée **au chargement de la credential**, avant toute requête SQL. Donc **les données pgvector sont intactes** (`aftrsn_memory`, `lumina_memory`, `n8n_chat_histories`). Seul l'objet credential n8n est à recréer.

## 4 · Trouver les paramètres de connexion (host interne)

La credential fantôme est illisible → il faut retrouver host/port/db/user ailleurs. Dans un stack Coolify/Dokploy « n8n + postgres » :
- Terminal du conteneur **Postgres** : `env` → `COOLIFY_CONTAINER_NAME=postgresql-<hash>` (= **host interne**), `POSTGRES_DB` (= base par défaut), `POSTGRES_USER`.
- **Vérifier où sont les tables** (souvent PAS la base `postgres` mais `n8n`) :
  ```
  psql -U <user> -d n8n -c "\dt" | grep -iE 'aftrsn_memory|lumina_memory|chat_histories'
  ```
- Host = nom de conteneur (jamais `localhost` : n8n et Postgres sont deux conteneurs). Port 5432. SSL off (réseau interne).

## 5 · Recréer la credential (action HUMAINE — jamais l'agent)

L'utilisateur crée une nouvelle credential Postgres dans n8n (Host = conteneur, Port 5432, Database = celle des tables, User + **mot de passe saisis par lui**). Test connection → Save. Elle reçoit un **nouvel id**. *(L'agent ne saisit jamais de secret.)*

## 6 · Repointer tous les nodes (action AGENT, sans secret)

Changer l'id de credential dans un node = config, pas un secret → l'agent le fait en masse via l'API :
```js
const OLD='<ghost>', NEW='<newId>', NAME='<newName>';
for(const wf of list){ const w=(await (await fetch('/rest/workflows/'+wf.id,{headers})).json()).data;
  let n=0; for(const node of w.nodes){ if(node.credentials?.postgres?.id===OLD){ node.credentials.postgres={id:NEW,name:NAME}; n++; } }
  if(n) await fetch('/rest/workflows/'+wf.id,{method:'PATCH',headers,body:JSON.stringify({name:w.name,nodes:w.nodes,connections:w.connections,settings:w.settings})}); }
```

## 7 · ⚠️ PIÈGE CENTRAL — republier les workflows actifs

**Un PATCH de credential met à jour le *brouillon*, pas la version *publiée/active*.** Résultat : les **webhooks et déclencheurs** actifs continuent d'exécuter l'ancienne credential (erreur inchangée) tant qu'on n'a pas **republié**. Les appels via **executeWorkflow**, eux, lisent le brouillon frais (d'où le piège : l'écriture semble marcher, la lecture par webhook non).

Republier chaque workflow **actif** repointé :
```js
const w=(await (await fetch('/rest/workflows/'+id,{headers})).json()).data;
await fetch('/rest/workflows/'+id+'/activate',{method:'POST',headers,
  body:JSON.stringify({versionId:w.versionId,name:w.name,description:w.description||''})});
```
(Un simple toggle active off/on ne suffit pas ; c'est bien `/activate` avec le `versionId` courant qui promeut le brouillon.)

## 8 · Vérifier la reprise

1. **Écriture** : appeler LUMINA-MEMORY-WRITE (episodic) via un appelant jetable.
2. **Lecture** : `POST /webhook/memory-search {question}` → doit renvoyer 200 + contenu.
3. **Agent** : exécuter le Maestro en chat → doit charger sa Chat Memory, faire une recherche mémoire et répondre.

## 9 · Durcissement (obligatoire après incident)

Construire un **health-check mémoire** planifié (`LUMINA-HEALTHCHECK-MEMOIRE`) : Schedule quotidien → HTTP `POST /webhook/memory-search` (neverError + fullResponse) → Code évalue `statusCode===200` → IF → **alerte Telegram** si KO, silence si OK. Empêche de redécouvrir une panne totale par hasard.

---

## Difficultés rencontrées
- Credential listée mais illisible (404) → fausse piste « re-sauver » impossible.
- Reboot Docker inopérant (le défaut est dans la donnée credential).
- Après repoint, la lecture par webhook restait cassée alors que l'écriture marchait.

## Solutions implémentées
- Recréation de la credential par l'utilisateur (host = conteneur Postgres, base `n8n`).
- Repoint API des 24 nodes + **republication `/activate`** des 17 actifs.
- Health-check mémoire planifié.

## Leçons apprises
- **Brouillon ≠ version publiée** : après tout changement de credential/param sur un workflow actif, republier via `/activate` — sinon les triggers gardent l'ancienne config.
- **executeWorkflow lit le brouillon frais ; webhook/trigger lit le publié** : un symptôme « écriture OK / lecture KO » pointe vers ce décalage.
- **Une credential partagée = point unique de défaillance** : la monitorer activement.
- L'erreur `configuration-node` « credential does not exist » = problème d'objet credential, **jamais** de la base ni des données.

---
*POS rédigée le 06.07.2026 après résolution de la panne. Générique par nature (infra LUMINA OS) — à pousser au canon.*
