---
type: raw
title: "POS-GENERIQUE_Suivi-Budget-Sheets-Base_20260705"
source_url: "drive:1Tj73CsA-Jy1ZhQfMhK-awzuJYFPl7MaT"
captured: 2026-07-07
vault: ai-automation
brand: null
immutable: true
---

# POS générique — Suivi budget piloté par un tableur cloud + une base de résumé, alertes de seuils, clôture auto

Pattern vérifié le 05.07.2026 (première implémentation : Budget-Tracker AFTRSN). Réutilisable pour toute marque gérée via LUMINA OS. Version spécifique : `POS-AFTRSN_Suivi-Budget-Sheets-Notion_20260705.md`.

## 1. Principe

Trois rôles séparés, à ne jamais mélanger :
1. **Le tableur (Google Sheets/Excel) = le détail.** C'est le document de travail humain, dérivé d'un **template** propre (structure fixe, formules de sous-totaux, zéro donnée d'exemple). Un fichier par objet suivi (édition, projet, chantier…), nommé par convention (`YYYYMMDD_<Type>_<Slug>`), dans un dossier cloud unique.
2. **La base structurée (Notion ou autre) = le résumé.** Quelques lignes par objet (catégories de dépenses, revenus, P&L), mises à jour par le workflow — jamais l'inverse.
3. **Un agent = la surveillance.** Il reçoit les chiffres à chaque passage et rend une lecture courte : dérives, moyennes utiles (ex. coût par tête vs capacité), recommandation actionnable.

## 2. Architecture n8n

- **Orchestrateur** (déclencheur périodique, timezone explicite) : lire la base des objets → Code de sélection (statuts ouverts, date requise ; date passée de N jours → mode `closing`, sinon `weekly` ; liste vide = arrêt silencieux) → **Execute Workflow mode « each »** → compteur → épisode mémoire → résumé.
- **Sous-workflow par objet** : localiser le fichier (recherche par préfixe de nom dans le dossier) → **le créer par copie du template s'il manque** (self-healing : aucun autre processus n'a besoin d'être modifié) → lire les plages utiles (`values:batchGet`, `UNFORMATTED_VALUE`, plages inline dans l'URL) → Code « chiffres » : sous-totaux par **lignes fixes du template**, totaux, indicateurs par tête, alertes de seuils (⚠️ ≥80 %, 🔴 ≥100 %, 🔴 dépense sans budget), et **construction ici même** du message de notification et de la query de l'agent → upsert des lignes de résumé (match par titre exact puis contains, créer si absent) → agent analyste (executeOnce) → si `closing` : marquer l'objet clos dans la base → notification (executeOnce) → sortie 1 item.
- **Idempotence** : hebdo = digest répétable (pas de dédup nécessaire) ; clôture = le statut clos sort l'objet de la sélection.

## 3. Points de vigilance techniques (n8n)

1. **Paramètres de requête dupliqués interdits** dans le node HTTP Request (sérialisation en objet, le dernier gagne — échec silencieux) : les répétitions type `ranges` vont dans l'URL.
2. Le tableur doit être en **format natif** (Google Sheets), pas xlsx : l'API values ne travaille bien que sur le natif.
3. Une credential **Drive** OAuth suffit pour l'API Sheets (scope accepté) — pas de credential Sheets dédiée. Mais l'**API Sheets doit être activée** sur le projet Google Cloud (403 sinon), action humaine (console, souvent protégée par passkey).
4. Les positions de lignes du template sont **câblées dans le code** : modifier le template = mettre à jour le mapping (documenter les deux ensemble).
5. Sous-workflows **publiés avant** l'appelant (l'appel échoue sur un callee inactif, même en test manuel) ; workflow dupliqué = **régénérer les webhookId** avant activation (409 sinon).
6. Tests scriptables sans UI : `POST /rest/workflows/<id>/run` avec le header `browser-id` depuis la console du navigateur connecté ; lire ensuite **les clés de `resultData.runData`** (jamais un `includes` sur le texte brut, qui contient aussi la définition du workflow).
7. Vérifier chaque branche en manipulant les données de pilotage (fichier absent/présent, lignes absentes/présentes, seuils franchis, mode clôture) + un run réel multi-objets pour valider le mode « each ».

## 4. Difficultés rencontrées / Solutions / Leçons

Voir §3–5 de la POS spécifique (identiques, formulées sur le cas concret) : ranges dupliqués silencieusement perdus → URL inline ; faux positifs des vérifications en texte brut → clés de runData ; 409 webhook → webhookId régénéré ; callee inactif → publier d'abord ; API non activée → activation humaine + vérification par l'effet ; données de test → neutraliser dans le statut ignoré par le process, suppression par l'humain.

**Leçon de fond :** séparer détail (tableur), résumé (base) et jugement (agent) rend chaque pièce remplaçable — on peut changer de tableur, de base ou d'agent sans toucher aux deux autres rôles.
