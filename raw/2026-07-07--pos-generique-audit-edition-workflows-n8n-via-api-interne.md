---
type: raw
title: "POS-GENERIQUE_Audit-edition-workflows-n8n-via-API-interne"
source_url: "drive:10XjDHzlb3Jk2wxyCum5PKDbFhAOlzlSI"
captured: 2026-07-07
vault: ai-automation
brand: null
immutable: true
---

# LUMINA — POS GÉNÉRIQUE — Auditer/éditer un n8n self-hosté par son API interne (sans clé API)

Patron réutilisable pour toute instance n8n : lire, modifier et publier des workflows par programme en s'appuyant sur une session navigateur authentifiée. Utile pour audits de parc d'agents, corrections en masse, documentation générée depuis la réalité.

## 1. Principe

n8n expose une API interne `/rest/*` (celle de sa propre UI). Deux conditions : cookie de session valide (`credentials:'include'` en same-origin) + header **`browser-id`** (clé localStorage `n8n-browserId`). Un `401` sur `/rest/login` = session expirée.

## 2. Endpoints utiles

| Action | Appel |
|---|---|
| Lister | `GET /rest/workflows?limit=N` |
| Détail (nodes, connexions, versionId) | `GET /rest/workflows/:id` |
| Modifier (crée un **draft**) | `PATCH /rest/workflows/:id` `{name, nodes, connections, settings, versionId}` |
| **Publier** le draft | `POST /rest/workflows/:id/activate` `{versionId}` |
| Historique d'exécutions | `GET /rest/executions?filter={"workflowId":"…"}` puis `GET /rest/executions/:execId` |

## 3. Règles d'or

1. **PATCH ≠ live** : sans `activate`, la PROD exécute l'ancienne version.
2. `versionId` frais obligatoire (GET → PATCH → activate avec le versionId renvoyé).
3. Publication refusée si credential manquante sur un node → copier la référence `{id, name}` d'un node équivalent sain (jamais de secret en clair).
4. Modifier une **copie profonde** des nodes ; ne toucher qu'au strict nécessaire ; GET de contrôle après.
5. Les connexions d'outils IA : `connections["<nom node>"] = {ai_tool: [[{node:'<agent>', type:'ai_tool', index:0}]]}` — le **nom** du node est la clé.

## 4. Audit type d'un parc d'agents

Pour chaque workflow : `type` des nodes, `options.systemMessage` (prompt réel), cibles des `toolWorkflow` (`parameters.workflowId`), présence de `credentials`, `active`. Croiser prompts ↔ tools câblés (les écarts sont fréquents : prompt à jour, tools en retard, ou l'inverse). Documenter les anomalies avant de corriger.

## 5. Lire une exécution (debug des entrées réelles)

`execution.data` est un tableau compressé : les valeurs scalaires `"N"` pointent vers l'index N du même tableau. Chercher ses marqueurs (noms de champs) puis résoudre les pointeurs. C'est le moyen le plus fiable de savoir **ce qu'un sub-workflow a réellement reçu** (vs ce qu'on croit lui envoyer).

## 6. Pièges

- Trigger sub-workflow en mode « JSON example » : champs non déclarés droppés silencieusement (l'exécution montre des défauts → voir §5).
- UI et API peuvent diverger temporairement (badge Publish) → l'API fait foi, `activate` est idempotent.
- Automatiser l'UI (chat de test) : cibler les champs par référence d'élément et vérifier le contenu avant Enter — les frappes perdues déclenchent les raccourcis du canvas.

---
*POS GÉNÉRIQUE — 2026-07-02. Version exacte : POS-LUMINA audit/édition API interne. S'applique à tout n8n self-hosté récent (schéma draft/publish).*
