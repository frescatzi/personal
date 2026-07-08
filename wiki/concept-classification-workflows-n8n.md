---
type: wiki
title: Standard de classification des workflows n8n (process-flow)
status: active
publish: notion
vault: ai-automation
brand:
sources:
  - raw/2026-07-02--pos-generique-classification-workflows-n8n-process-flow-2026-07-02.md
  - raw/2026-07-06--pos-generique-classification-workflows-n8n-process-flow-2026-07-02.md
related:
  - wiki/sop/sop-clonage-roster-agents.md
  - wiki/sop/sop-audit-edition-n8n-api-interne.md
  - wiki/synthese-lumina-ai-os.md
  - wiki/sop/Guide-Connexion-Agents-AI-n8n.md
updated: 2026-07-06
---

# Standard de classification des workflows n8n (process-flow)

## Idée centrale

Convention **unique et réutilisable** : ranger, nommer et taguer tout workflow n8n de façon à **lire la chaîne du début à la fin** et savoir immédiatement où chercher en cas de problème. À réappliquer à l'identique pour chaque nouvelle marque.

## Structure de dossiers (numérotée dans l'ordre du flux)

| Dossier | Rôle | Où chercher si… |
|---------|------|-----------------|
| `01-INTAKE` 📥 | Capture des entrées brutes (Drive→GitHub, archivage RAW, ingestion fichiers) | …la donnée n'arrive pas / mauvaise source |
| `02-MEMORY` 🧠 | Connaissance : écriture, récupération, consolidation, dédup, vectoriel, provisioning | …une info manque en mémoire / doublons |
| `03-BRAIN` 🤖 | Raisonnement : Maestro + sous-agents (sub-dossier `Sub-Agents`) + routeur LLM 🧭 + exécuteur ⚙️ | …mauvaise réponse, mauvais agent/modèle |
| `04-AUTOMATION` 🔄 | Orchestrateurs & tâches planifiées | …quelque chose ne s'est pas déclenché |
| `05-CONNECT` 🔌 | Interfaces externes (MCP, webhooks publics) | …un outil externe ne se branche pas |
| `08-UTIL` 🛠️ | Outils & one-off (console SQL, DDL runner) | Maintenance manuelle |
| `Z_ARCHIVES` ⚫ | Obsolètes (préfixe `ZZZ_<date>_`) | Historique |
| `SandBox` 🔵 | Tests | Essais |

Le hub (**Maestro**) vit à la racine de `03-BRAIN` ; les sous-agents dans `03-BRAIN/Sub-Agents`.

## Tags — 4 axes obligatoires

Chaque workflow reçoit **1 tag par axe pertinent** dès sa création :

| Axe | Valeurs |
|-----|---------|
| **Domaine** | 📥 INTAKE · 🧠 MEMORY · 🤖 AI-AGENTS · 🧭 ROUTER · ⚙️ EXEC · 🔄 automation · 🔌 MCP · 🛠️ UTIL |
| **Statut** | 🟢 PROD · 🔵 TEST · ⚫ DEPRECATED |
| **Déclencheur** | 🪝 webhook · ⏰ scheduled · ⚡ EVENT · 👆 MANUAL *(les sous-workflows executeWorkflowTrigger peuvent n'avoir aucun tag déclencheur)* |
| **Marque** | 🌐 SHARED (infra Lumina, mutualisé) · 🌞 AFTRSN (spécifique marque) · *1 tag par nouvelle marque* |

Règle : tout ce qui vit sous `01-LUMINA` = 🌐 SHARED ; tout ce qui est propre à une marque = son tag.

## Nommage des workflows

- **Étape :** `<PORTÉE>-<DOMAINE>-<objet>/<précision>` — ex. `LUMINA-MEMORY-CONSOLIDATION/NIGHTLY`
- **Agent :** `<MARQUE>-<Rôle>` — ex. `AFTRSN-Maestro`, `AFTRSN-Secretary`
- **Obsolète :** `ZZZ_<AAAAMMJJ>_<ancien-nom>` + tag ⚫ DEPRECATED + dossier `Z_ARCHIVES`

## Onboarding d'une nouvelle marque

1. Créer le dossier-projet racine `NN-<MARQUE>`.
2. Y recréer les mêmes dossiers-étapes : `01-INTAKE`, `02-MEMORY`, `03-BRAIN` (+ `Sub-Agents`), `04-AUTOMATION`, `05-CONNECT`, `08-UTIL`.
3. Créer le tag Marque (emoji + nom) → appliquer à tous ses workflows.
4. En général une marque ne remplit que `02-MEMORY` (ingestions) + `03-BRAIN` (roster cloné) ; INTAKE/AUTOMATION/CONNECT/UTIL restent 🌐 SHARED au niveau LUMINA.

## API interne n8n (ranger + taguer par programme)

```js
const hdr = { credentials:'include', headers: { 'browser-id': localStorage.getItem('n8n-browserId'), 'content-type':'application/json', 'accept':'application/json' } };
// Déplacer + taguer un workflow en un seul PATCH (sans versionId nécessaire)
fetch('/rest/workflows/<wfId>', { method:'PATCH', ...hdr,
  body: JSON.stringify({ parentFolderId:'<folderId>', tags:['<tagId1>','<tagId2>'] }) });
```

**Pièges :** `PUT /rest/workflows/:id/tags` n'existe pas → passer les tags dans le PATCH (`{tags:[ids]}` remplace l'ensemble). Header `browser-id` obligatoire. Voir [[sop/sop-audit-edition-n8n-api-interne]] pour le détail complet.

## Voir aussi

- [[sop/sop-clonage-roster-agents]] — cloner le roster et le ranger dans `03-BRAIN/Sub-Agents`.
- [[sop/sop-audit-edition-n8n-api-interne]] — auditer/éditer les workflows par API interne.
- [[synthese-lumina-ai-os]] — la topologie globale du système Lumina.
- [[sop/Guide-Connexion-Agents-AI-n8n]] — créer les credentials dans n8n.
