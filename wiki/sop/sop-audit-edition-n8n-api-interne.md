---
type: wiki
title: SOP — Auditer/éditer un n8n self-hosté par son API interne (sans clé API)
status: active
publish: notion
vault: ai-automation
brand:
sources:
  - raw/2026-07-02--pos-generique-audit-edition-workflows-n8n-via-api-interne.md
  - raw/2026-07-06--pos-generique-audit-edition-workflows-n8n-via-api-interne.md
  - raw/2026-07-02--pos-lumina-audit-edition-workflows-n8n-api-interne-2026-07-02.md
  - raw/2026-07-06--pos-lumina-audit-edition-workflows-n8n-api-interne-2026-07-02.md
  - raw/2026-07-02--pos-generique-classification-workflows-n8n-process-flow-2026-07-02.md
related:
  - wiki/concept-classification-workflows-n8n.md
  - wiki/sop/sop-cablage-orchestrateur-subagents.md
  - wiki/sop/sop-clonage-roster-agents.md
  - wiki/sop/Guide-Connexion-Agents-AI-n8n.md
updated: 2026-07-06
---

# SOP — Auditer/éditer un n8n self-hosté par son API interne (sans clé API)

## Principe

n8n expose une API interne `/rest/*` (celle de sa propre UI). Conditions : cookie de session valide (`credentials:'include'`, même origine) + header **`browser-id`** (clé localStorage `n8n-browserId`). Un `401` sur `/rest/login` = session expirée.

## Endpoints utiles

| Action | Appel |
|--------|-------|
| Lister les workflows | `GET /rest/workflows?limit=N` |
| Détail (nodes, connexions, versionId) | `GET /rest/workflows/:id` |
| Modifier (crée un **draft**) | `PATCH /rest/workflows/:id` `{name, nodes, connections, settings, versionId}` |
| Publier le draft | `POST /rest/workflows/:id/activate` `{versionId}` |
| Déplacer + taguer (sans versionId) | `PATCH /rest/workflows/:id` `{parentFolderId, tags:[ids]}` |
| Créer un tag | `POST /rest/tags` `{name}` |
| Créer un dossier | `POST /rest/projects/:pid/folders` `{name, parentFolderId}` |
| Historique d'exécutions | `GET /rest/executions?filter={"workflowId":"…"}` |
| Détail d'une exécution | `GET /rest/executions/:execId` |

## Règles d'or

1. **PATCH ≠ live** : sans `POST activate`, la PROD exécute l'ancienne version.
2. `versionId` frais obligatoire pour les modifications structurelles (GET → PATCH → activate avec le versionId renvoyé par GET).
3. Publication refusée si **credential manquante** sur un node → copier la référence `{id, name}` d'un node sain.
4. Modifier une **copie profonde** des nodes ; ne toucher qu'au strict nécessaire ; GET de contrôle après.
5. Connexions outils IA : `connections["<nom node>"] = {ai_tool: [[{node:'<agent>', type:'ai_tool', index:0}]]}` — le **nom** du node est la clé.

## Template d'appel (session Chrome)

```js
const hdr = {
  credentials: 'include',
  headers: {
    'browser-id': localStorage.getItem('n8n-browserId'),
    'content-type': 'application/json',
    'accept': 'application/json'
  }
};
const PID = '<projectId>';

// Lister les workflows
const wfs = await fetch('/rest/workflows?limit=100', hdr).then(r => r.json());

// Déplacer + taguer en un seul PATCH (remplace l'ensemble des tags)
await fetch('/rest/workflows/<wfId>', { method: 'PATCH', ...hdr,
  body: JSON.stringify({ parentFolderId: '<folderId>', tags: ['<tagId1>', '<tagId2>'] }) });

// Modifier le contenu (draft) puis publier
const detail = await fetch('/rest/workflows/<wfId>', hdr).then(r => r.json());
const nodes = JSON.parse(JSON.stringify(detail.data.nodes)); // copie profonde
// ... modifier nodes ...
await fetch('/rest/workflows/<wfId>', { method: 'PATCH', ...hdr,
  body: JSON.stringify({ nodes, connections: detail.data.connections, settings: detail.data.settings, versionId: detail.data.versionId }) });
await fetch('/rest/workflows/<wfId>/activate', { method: 'POST', ...hdr,
  body: JSON.stringify({ versionId: '<versionId du PATCH>' }) });
```

## Audit type d'un parc d'agents

Pour chaque workflow récupérer : `type` des nodes · `options.systemMessage` (prompt réel) · cibles des `toolWorkflow` (`parameters.workflowId`) · présence de `credentials` · `active`. Croiser prompts ↔ tools câblés (les écarts sont fréquents : prompt à jour, tools en retard, ou l'inverse).

## Lire une exécution (debug des entrées réelles)

`execution.data` est un tableau compressé : les valeurs scalaires `"N"` pointent vers l'index N du même tableau. Chercher ses marqueurs (noms de champs) puis résoudre les pointeurs. Méthode la plus fiable pour savoir **ce qu'un sub-workflow a réellement reçu**.

## Pièges

- Trigger `executeWorkflowTrigger` en mode « JSON example » : champs non déclarés droppés silencieusement (diagnostic via exécution → entrées réelles).
- UI et API peuvent diverger temporairement → l'API fait foi, `activate` est idempotent.
- `PUT /rest/workflows/:id/tags` n'existe pas → passer les tags dans le `PATCH` du workflow.
- Header `browser-id` obligatoire (sinon 401).

## Voir aussi

- [[concept-classification-workflows-n8n]] — standard de rangement (dossiers + 4 axes de tags).
- [[sop/sop-cablage-orchestrateur-subagents]] — câbler les outils et vérifier les connexions.
- [[sop/sop-clonage-roster-agents]] — cloner un roster par API.
- [[sop/Guide-Connexion-Agents-AI-n8n]] — créer et gérer les credentials dans n8n.
