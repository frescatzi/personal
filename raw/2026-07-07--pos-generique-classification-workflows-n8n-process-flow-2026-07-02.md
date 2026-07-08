---
type: raw
title: "POS-GENERIQUE_Classification-workflows-n8n_process-flow_2026-07-02"
source_url: "drive:1E2wtSln4h0NnSdcee6gp6JiW38IVfohX"
captured: 2026-07-07
vault: ai-automation
brand: null
immutable: true
---

# LUMINA — POS — Standard de classification des workflows n8n (thème « process-flow ») — 2026-07-02

Convention **unique et réutilisable** pour ranger, nommer et taguer tout workflow n8n, de façon à **lire la chaîne du début à la fin** et savoir immédiatement où chercher en cas de problème. À réappliquer à l'identique pour chaque nouvelle marque.

## 1. Principe : la chaîne comme fil conducteur

Un workflow appartient à **une étape** de la chaîne de traitement. Les dossiers sont **numérotés dans l'ordre du flux** : la donnée entre en `01`, se transforme en connaissance en `02`, est raisonnée en `03`, est orchestrée en `04`, ressort en `05`. Quand un incident survient, on remonte la chaîne : problème en amont → regarder `01/02` ; problème en aval (réponse, sortie) → regarder `03/04/05`.

## 2. Étapes (dossiers) — mêmes dans CHAQUE projet/marque

| Dossier | Rôle dans la chaîne | Où chercher si… |
|---|---|---|
| `01-INTAKE` 📥 | Capture des entrées brutes (Drive→GitHub, archivage RAW→Drive, ingestion de fichiers) | …la donnée n'arrive pas / mauvaise source |
| `02-MEMORY` 🧠 | Couche connaissance : écriture, récupération, consolidation, dédup, ingestion vectorielle, provisioning de banque | …une info manque en mémoire / mauvais souvenir / doublons |
| `03-BRAIN` 🤖 | Couche raisonnement : Maestro + sous-agents (sous-dossier **Sub-Agents**), routeur LLM 🧭, exécuteur ⚙️ | …mauvaise réponse, mauvais agent, mauvais modèle |
| `04-AUTOMATION` 🔄 | Orchestrateurs & tâches planifiées | …quelque chose ne s'est pas déclenché à l'heure |
| `05-CONNECT` 🔌 | Interfaces externes (serveur MCP, webhooks publics) | …un outil externe ne se branche pas |
| `08-UTIL` 🛠️ | Outils & one-off (console SQL, DDL runner) | …opérations manuelles / maintenance |
| `Z_ARCHIVES` ⚫ | Obsolètes (préfixe `ZZZ_<date>_`) | (historique) |
| `SandBox` 🔵 | Bacs à sable / tests | (essais) |

Le sous-dossier **`Sub-Agents`** vit dans `03-BRAIN` et contient tous les sous-agents ; le **hub (Maestro)** reste à la racine de `03-BRAIN`.

## 3. Tags — 4 axes obligatoires (croisables pour filtrer)

Chaque workflow porte **1 tag par axe pertinent** :

- **Domaine** (= l'étape) : 📥 INTAKE · 🧠 MEMORY · 🤖 AI-AGENTS · 🧭 ROUTER · ⚙️ EXEC · 🔄 automation · 🔌 MCP · 🛠️ UTIL
- **Statut** : 🟢 PROD · 🔵 TEST · ⚫ DEPRECATED
- **Déclencheur** : 🪝 webhook · ⏰ scheduled · ⚡ EVENT · 👆 MANUAL *(les sous-workflows appelés — executeWorkflowTrigger — peuvent n'avoir aucun tag déclencheur)*
- **Marque** : 🌞 AFTRSN (spécifique marque) · 🌐 SHARED (plateforme LUMINA, mutualisé)

Règle marque : tout ce qui vit sous `01-LUMINA` = 🌐 SHARED ; tout ce qui est propre à une marque = son tag (🌞 AFTRSN, puis un tag par nouvelle marque).

## 4. Nommage

- Workflow d'étape : `<PORTÉE>-<DOMAINE>-<objet>/<précision>` — ex. `LUMINA-MEMORY-CONSOLIDATION/NIGHTLY`, `AFTRSN-MEMORY-INGESTION/DRIVE-RECURSIVE`.
- Agent : `<MARQUE>-<Rôle>` — ex. `AFTRSN-Maestro`, `AFTRSN-Comptable-Finance`.
- Obsolète : préfixe `ZZZ_<AAAAMMJJ>_…` + tag ⚫ DEPRECATED + dossier `Z_ARCHIVES`.

## 5. Onboarding d'une nouvelle marque (répliquer le standard)

1. Créer le dossier-projet racine `NN-<MARQUE>`.
2. Y recréer **les mêmes dossiers-étapes** : `01-INTAKE`, `02-MEMORY`, `03-BRAIN` (+ `Sub-Agents`), `04-AUTOMATION`, `05-CONNECT`, `08-UTIL`.
3. Créer le **tag Marque** (emoji + nom) et l'appliquer à tous ses workflows.
4. En général, une marque ne remplit que `02-MEMORY` (ses ingestions) + `03-BRAIN` (son roster via `cloneRoster`) ; INTAKE/AUTOMATION/CONNECT/UTIL restent surtout 🌐 SHARED au niveau LUMINA. Les dossiers vides servent de gabarit cohérent.
5. Chaque workflow reçoit ses 4 axes de tags dès sa création.

## 6. Registre concret (instance Karter — projet personnel `Tn1aNTcuxmqqHnKU`)

**Tags (nom → id)** :
`📥 INTAKE`=Bq1V7roUahSIFAqN · `🧠 MEMORY`=NrnNhX2FwO6DWYq2 · `🤖 AI-AGENTS`=PLsFqlaD1O3kVGZb · `🧭 ROUTER`=FyHnrNjLWWZSiYft · `⚙️ EXEC`=aUMh5GyVbBWoHKyJ · `🔄 automation`=Ob5ltXxuxwXTa4R7 · `🔌 MCP`=1X4KqJlfvyzLTfq5 · `🛠️ UTIL`=kweycDAtJ9KW26i3 · `🟢 PROD`=sMHN7rMS8y2Tuqc8 · `🔵 TEST`=OlrEdgJCVqSDkAca · `⚫ DEPRECATED`=rR3DMMpHtsW3GRXs · `🪝 webhook`=oCUaeWnaG6xZEgAc · `⏰ scheduled`=AUQSPxQbVXbDH1O5 · `⚡ EVENT`=6kMTaoT3rmNeTNOJ · `👆 MANUAL`=oau7nsmLueuUgZ9C · `🌞 AFTRSN`=CMOhObFy9TbwnlQw · `🌐 SHARED`=IRVRBhTdu1LHRSOq

**Dossiers (nom → id)** :
LUMINA-01-INTAKE=ntwo2MwpM7Rw4dhQ · LUMINA-02-MEMORY=VxHTlmxUXhgJa08E · LUMINA-03-BRAIN=koTTfFCTNnUbC9ap · LUMINA-04-AUTOMATION=VlyZEg5Fb5yuYcyp · LUMINA-05-CONNECT=KtGTP2XPa7QS0OlB · LUMINA-08-UTIL=0FWS0obsDKkax8Ie · (racine 01-LUMINA=ZEVnJsT8dEMqnEHX) · AFTRSN-01-INTAKE=kU2rSsztidL9NP6x · AFTRSN-02-MEMORY=u3uD67U3cyO7XRak · AFTRSN-03-BRAIN=Ioxkbv2hecviwAD6 · Sub-Agents=7XfoUsGBE3TZoWVp · AFTRSN-04-AUTOMATION=hknO6wSdn3KSNb0V · AFTRSN-05-CONNECT=D77E82uT5HrBvfIV · AFTRSN-08-UTIL=rnyO0wXQvOU7pDNo · (racine 02-AFTRSN=FgvLceDAe5kVygAc) · Z_ARCHIVES=Om3ebvS5VQsd8Ant · SandBox=LitVEyvZbLzzQsOF

## 7. Appliquer par API (n8n interne, via fetch dans la session Chrome de Karter)

```js
const hdr={credentials:'include',headers:{'browser-id':localStorage.getItem('n8n-browserId'),'content-type':'application/json','accept':'application/json'}};
const PID='Tn1aNTcuxmqqHnKU';
// créer un tag
fetch('/rest/tags',{method:'POST',...hdr,body:JSON.stringify({name:'🌞 AFTRSN'})});
// créer un dossier d'étape
fetch(`/rest/projects/${PID}/folders`,{method:'POST',...hdr,body:JSON.stringify({name:'AFTRSN-02-MEMORY',parentFolderId:'<racineMarque>'})});
// renommer un dossier
fetch(`/rest/projects/${PID}/folders/<fid>`,{method:'PATCH',...hdr,body:JSON.stringify({name:'…'})});
// RANGER + TAGUER un workflow en un seul PATCH (partiel, sans versionId)
fetch('/rest/workflows/<wfId>',{method:'PATCH',...hdr,body:JSON.stringify({parentFolderId:'<folderId>', tags:['<tagId1>','<tagId2>', /*…*/]})});
```

Pièges : l'endpoint `PUT /rest/workflows/:id/tags` n'existe pas ici → passer les tags **dans le PATCH du workflow** (`{tags:[ids]}`, remplace l'ensemble). Le PATCH `{parentFolderId}` déplace sans toucher au contenu. Header `browser-id` obligatoire.

---
*POS — 2026-07-02, Claude (Cowork). Standard de classification process-flow, appliqué à LUMINA + AFTRSN (32 workflows, 0 en racine, 0 sans tag). À réappliquer pour chaque nouvelle marque. Lié au PLAYBOOK v2 (cloneRoster range ses agents dans `03-BRAIN/Sub-Agents`).*
