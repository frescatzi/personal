---
type: raw
title: "concept-n8n-credentials"
source_url: "drive:14aSseDOof4Hw9BAyYmmjy8eRNfiWwFoz"
captured: 2026-07-07
vault: ai-automation
brand: null
immutable: true
---

---
type: wiki
title: "Credentials n8n — gestion et bonnes pratiques"
status: draft
publish: none
vault: ai-automation
brand: null
sources: []
related:
  - wiki/synthese-oauth2-n8n-google.md
  - wiki/concept-oauth2-automation.md
  - wiki/sop/Guide-Connexion-Agents-AI-n8n.md
  - wiki/sop/n8n-Brancher-API-et-Premier-Workflow.md
updated: 2026-06-29
---

# Credentials n8n — gestion et bonnes pratiques

> Page stub — à enrichir à partir d'une source raw/ dédiée.

## Idée centrale

n8n centralise tous les accès aux services externes dans des **credentials** réutilisables : clés API (Anthropic, OpenAI, Gemini) et tokens OAuth2 (Google Drive, Gmail…). Un credential bien configuré peut être partagé entre plusieurs workflows sans être ré-saisi.

## Types de credentials courants

- **API Key** : clé statique (ex. `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`). Simple, risquée si exposée.
- **OAuth2** : flux Authorization Code. n8n agit comme client OAuth — voir [[concept-oauth2-automation]] et [[synthese-oauth2-n8n-google]] pour la procédure Google.
- **HTTP Header Auth / Basic Auth** : pour APIs sans SDK dédié.

## Bonnes pratiques

- Ne jamais coller une clé API dans un nœud en dur — toujours utiliser un credential nommé.
- En équipe, préférer les **variables d'environnement n8n** (`N8N_ENCRYPTION_KEY`, etc.) pour éviter que les secrets soient stockés en clair dans la DB SQLite.
- Révoquer et régénérer les clés compromises immédiatement depuis la console du fournisseur.
- Pour les credentials OAuth2 Google en mode Test : anticiper l'**expiration des refresh tokens à 7 jours** (voir [[concept-oauth2-automation]]).

## À documenter (trou actuel)

- Procédure de rotation d'une clé API Anthropic dans n8n sans interrompre les workflows actifs.
- Export/import de credentials entre instances n8n.

## Voir aussi

- [[sop/Guide-Connexion-Agents-AI-n8n]] — créer les credentials Anthropic / OpenAI / Gemini pas à pas.
- [[sop/n8n-Brancher-API-et-Premier-Workflow]] — brancher et tester le premier workflow avec ces credentials.
- [[concept-oauth2-automation]] — patron OAuth2 générique utilisé par les credentials de type Google.
- [[synthese-oauth2-n8n-google]] — guide spécifique Google Drive / Gmail dans n8n.
