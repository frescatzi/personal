# Session — Agents AFTRSN, Orchestration & Serveur MCP

**Date :** 2026-06-20
**Projet :** AFTER SUN PEOPLE (AFTRSN) — AI Operating System
**Objectif de la session :** Construire les 3 agents spécialistes, brancher l'orchestration sur le Superviseur, et exposer la mémoire centrale à Claude via un serveur MCP.
**Statut :** ✅ Atteint et vérifié de bout en bout

---

## Ce qui a été fait

**1. Les 3 agents spécialistes (montés par duplication du Superviseur).**
Noms finaux des workflows n8n :
- **Brand-Guardian** — cohérence & voix de marque.
- **Channel Content** — contenu & canaux (Instagram, newsletter, site, calendrier éditorial).
- **Acquisition Performance** — campagnes, KPIs, acquisition, recaps de performance.

Chaque agent reprend le même montage que le Superviseur (Chat Model gpt-4o-mini + outil **Knowledge** + **Chat memory**), avec un **System Message propre, en anglais**, structuré ainsi : identité + règle de nommage AFTRSN + « always query the Knowledge tool before answering » + périmètre/limites (rester dans son domaine, déférer aux autres) + validation finale par Katel sur le critique + « Respond in the user's language ».

**2. Orchestration réelle (le vrai hub-and-spoke).**
Le **Superviseur** appelle désormais chaque spécialiste comme un **outil** :
- Sur chaque spécialiste : ajout d'un trigger **« When Executed by Another Workflow »** avec un champ d'entrée `query` (String), et passage du **Source for Prompt** de l'AI Agent à `{{ $json.query || $json.chatInput }}` (fonctionne en test chat *et* en appel comme outil).
- Sur le Superviseur : 3 nodes **« Call n8n Workflow Tool »** (un par spécialiste), chacun avec `query` rempli via le bouton **✨** (`$fromAI(...)`).
- System Message du Superviseur enrichi d'un bloc « You can delegate to three specialist tools… ».

**3. Serveur MCP (mémoire partagée Claude ↔ n8n).**
- Nouveau workflow **AFTRSN-MCP-Server** : node **MCP Server Trigger** + **HTTP Request Tool** `search_brand_memory` (POST vers le webhook `memory-search`, body `{"question": "{{ $fromAI('question', '...') }}"}`).
- Workflow activé → récupération de la **Production URL (SSE)**.
- Ajout dans Claude via **Customize → Connectors → Add custom connector** (URL HTTPS).
- **Test validé :** depuis Claude/Cowork, l'outil `search_brand_memory` renvoie bien les entrées de la bible de marque (boucle complète Claude → MCP → n8n → Postgres/pgvector).

---

## Décisions

- **Noms d'agents figés** : Brand-Guardian, Channel Content, Acquisition Performance (+ AFTRSN-Supervisor).
- **Spécialistes = « ouvriers »** : c'est le **Superviseur** qui porte la mémoire de conversation. (Variante retenue pour corriger le bug Chat memory, voir plus bas.)
- **MCP sans authentification pour le MVP** (URL HTTPS à identifiant aléatoire = obscurité). Durcissement OAuth2 prévu plus tard.
- **Contenu des champs en anglais** ; explications en français (préférence projet).

---

## Challenges & solutions

1. **`query` introuvable dans le Call Workflow Tool.** La section *Workflow Inputs* (donc le champ `query`) n'apparaît qu'**après** avoir sélectionné le workflow cible dans le dropdown. Solution : choisir d'abord le spécialiste, puis le champ apparaît → bouton **✨** pour le `$fromAI`.
2. **« Workflow is not active and cannot be executed ».** Un workflow appelé comme outil doit être **actif**. Les spécialistes, créés par duplication, étaient inactifs. Solution : **Publish/Active** chaque spécialiste.
3. **« Error in sub-node Chat Memory ».** Quand un spécialiste est lancé via *Executed by Another Workflow* (et non via son Chat Trigger), le node Chat memory ne trouve pas de session de chat. Solution appliquée : **Session ID → Define below → `{{ $json.sessionId || 'subagent' }}`** (clé valide dans les deux cas). Alternative : retirer la Chat memory des spécialistes.
4. **Auth MCP côté Claude.** L'interface connecteur de Claude (web/Cowork/Desktop) gère OAuth ou sans-auth, pas un simple Bearer collé. Solution MVP : démarrer **sans auth** ; Bearer possible côté **Claude Code** (`--header`).

---

## Lessons learned

- Un **agent n8n peut être l'outil d'un autre agent** via *Call n8n Workflow Tool* + *Execute Workflow Trigger* : c'est le cœur du hub-and-spoke.
- Pour qu'un agent fonctionne **et** en chat direct **et** comme sous-outil, prompt source = `{{ $json.query || $json.chatInput }}`.
- Les **sous-agents n'ont pas besoin de mémoire de conversation** : centraliser la mémoire de fil sur le Superviseur évite les bugs de session.
- Le node **MCP Server Trigger** transforme n'importe quel outil n8n en outil MCP → **une seule mémoire, accessible partout** (n8n + Claude web/Cowork/Code).

---

## Points critiques à arbitrer (CEO)

1. **Sécuriser le MCP** (OAuth2) avant d'y exposer des données sensibles. Aujourd'hui l'accès est ouvert via URL obscure.
2. **Enrichir la bible de marque** : une recherche « voix et ton » remonte surtout des règles de nommage, pas un passage dédié *Voice & Tone* (meilleur score ~0.67). Ajouter un bloc explicite dans Notion (ré-ingéré automatiquement) rendra Brand-Guardian beaucoup plus précis.

---

## Prochaines étapes

- Enrichir la bible (bloc *Voice & Tone*, *Operations*, etc.) puis vérifier la qualité des réponses via `search_brand_memory`.
- Passer le serveur MCP en **OAuth2**.
- Reprendre l'autre chantier : **documentation business/financière** (vision, mission, stratégie, KPIs, centres de coûts) → puis tableurs.
