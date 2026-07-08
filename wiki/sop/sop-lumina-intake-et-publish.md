---
type: wiki
title: "SOP Lumina — Intake (Drive→GitHub) & Publication Notion idempotente"
status: active
publish: notion
vault: ai-automation
brand: null
sources:
  - "raw/2026-06-24--sop-lumina-intake-et-publish.md"
  - "raw/2026-06-29--sop-intake-github-idempotent-blinde.md"
related:
  - "synthese-lumina-systeme-reference"
  - "sop/sop-generique-pipeline-source-vers-vues"
  - "sop/SOP_systeme-multi-agents-memoire-centrale-mcp-n8n"
  - "concept-intake-source-git"
  - "sop/sop-lumina-archive-raw-vers-drive"
updated: 2026-06-29
---

# SOP Lumina — Intake (Drive→GitHub) & Publication Notion idempotente

> Procédure **propre à l'infrastructure Lumina (AFTRSN)**. Pour la version réutilisable/abstraite, voir [[sop/sop-generique-pipeline-source-vers-vues]].
> Couvre 2 workflows n8n : le **robot d'intake v2** et le **publish Notion idempotent**.
> Vue d'ensemble du système : [[synthese-lumina-systeme-reference]].

---

## 1. Objet & périmètre

- **But A (intake)** : vider en continu le dossier Drive **Lumina Inbox**, classer chaque fichier (LLM), le ranger dans `<coffre>/raw/` sur GitHub, puis le supprimer de Drive.
- **But B (publish)** : générer/synchroniser le wiki GitHub vers la base **Notion « AI Automation — Knowledge »**, en **upsert par fichier** (zéro doublon, idempotent).
- **Principe directeur** : Git = source de vérité ; Notion/pgvector = dérivés régénérables, jamais édités à la main.

## 2. Prérequis (credentials & identifiants)

- n8n self-hosted : `n8n.aftersunpeople.com`.
- Credentials n8n : **OpenAI**, **GitHub** (PAT, Contents R/W sur `ai-automation`/`brands`/`personal`), **Google Drive**, **Notion**.
- Identifiants :
  - Drive **Lumina Inbox** : `1gjuB438pW7NKo4A-wbjaud_PH7ijnNUW`
  - GitHub owner : `frescatzi` · repos = nom du coffre (`ai-automation`, `brands`, `personal`)
  - Notion DB : `6fac8cb891e44c749e37a86419826905` · data_source : `657827e3-1e2a-48d5-8f52-5954e3618687`
  - ⚠️ L'intégration **AFTRN-n8n** doit être ajoutée en **Connection** sur la base Notion (sinon « database not found »).

---

## 3. SOP A — Robot d'intake (Drive → GitHub `raw/`)

Chaîne : `Schedule Trigger → Search files and folders → Download file → Extract from File → OPEN-AI-CALL → Code_Parse → If Rejected → Create a file (GitHub) → Delete a file (Drive)`

1. **Schedule Trigger** — ex. `Every Hour`. (Active le workflow pour qu'il tourne.)
2. **Google Drive → Search files and folders** — dossier = Lumina Inbox, **Return All = ON**, `trashed = false`. → liste TOUS les fichiers (backlog + nouveaux).
3. **Google Drive → Download file** — File ID `{{ $json.id }}` (mode Expression).
4. **Extract from File** — `Extract From Text File`, Input Binary Field `data`.
5. **HTTP → OPEN-AI-CALL** :
   - `POST https://api.openai.com/v1/chat/completions`, auth OpenAI.
   - Body JSON : `model: gpt-4o-mini`, `temperature: 0`, `response_format: {type: json_object}`, `messages` = system (classeur : brands / ai-automation / personal / reject, répond en JSON `{vault, brand, reason}`) + user `{{ JSON.stringify($json.data) }}`.
   - Settings : **Retry On Fail** ON (4-5 essais, 5 s).
6. **Code → Code_Parse** — **Mode = Run Once for Each Item** :
   ```js
   const gem = JSON.parse($json.choices[0].message.content);
   const trig = $('Search files and folders').item.json;
   const content = $('Extract from File').item.json.data;
   const date = new Date().toISOString().slice(0,10);
   const base = (trig.name || 'sans-titre').replace(/\.[^.]+$/,'');
   const slug = base.toLowerCase().normalize('NFD').replace(/[̀-ͯ]/g,'').replace(/[^a-z0-9]+/g,'-').replace(/(^-|-$)/g,'');
   const vault = gem.vault, brand = gem.brand;
   const path = vault === 'brands' ? `${brand}/raw/${date}--${slug}.md` : `raw/${date}--${slug}.md`;
   const fm = `---\ntype: raw\ntitle: "${base}"\nsource_url: "drive:${trig.id}"\ncaptured: ${date}\nvault: ${vault}\nbrand: ${brand ?? 'null'}\nimmutable: true\n---\n\n`;
   return { vault, brand, reject: vault === 'reject', repo: vault, path, fileContent: fm + content, fileId: trig.id, filename: trig.name };
   ```
7. **IF → If Rejected** — `{{ $json.reject }}` : true = ne pas ranger.
8. **GitHub → Create a file** — repo `By URL` = `https://github.com/frescatzi/{{ $json.repo }}`, path `{{ $json.path }}`, content `{{ $json.fileContent }}`. **On Error = Stop Workflow**.
9. **Google Drive → Delete a file** — File ID `{{ $('Code_Parse').item.json.fileId }}`.
10. **Tester** en `Execute workflow`, vérifier que l'inbox se vide, puis **Activer**.

## 4. SOP B — Publication Notion idempotente (wiki → Notion)

Chaîne : `Trigger → Query Notion DB → HTTP GitHub Trees → Split Out1 (tree) → Filter (wiki/*.md, exclut README) → Get a file → Set Document → Code (Parsing)` puis **fork** :
- branche **créer** : `Build Notion payload → POST Notion`
- branche **nettoyer** : `Query BY SOURCE → Split Out (results) → ARCHIVE PAGE`

1. **Filter** : garder `wiki/*.md` ET `path does not contain README`.
2. **Code (Parsing)** : extrait frontmatter (notionTitle/type/tags/vault/sourceUrl/updated/body). ⚠️ **une seule entrée** (Set Document).
3. **Build Notion payload** : markdown → blocs Notion + propriétés ; produit l'URL Source complète dans `payload.properties.Source.url`.
4. **POST Notion** : `POST /v1/pages`, header `Notion-Version: 2022-06-28`, body `{{ $json.payload }}`.
5. **Query BY SOURCE** : `POST /v1/databases/6fac8cb891e44c749e37a86419826905/query`, body en **mode Expression** :
   `{{ JSON.stringify({ filter: { property: "Source", url: { equals: $json.payload.properties.Source.url } } }) }}`. Branché sur **Build Notion payload**.
6. **Split Out** sur `results` → **ARCHIVE PAGE** : `PATCH /v1/pages/{{ $json.id }}` body `{"archived":true}`, On Error = Continue.
7. **Tester** : 2 runs doivent donner le même nombre de pages (idempotent).

---

## 5. Challenges rencontrés & solutions

| # | Challenge | Cause | Solution |
|---|---|---|---|
| 1 | Doublons Notion (~40→74 entrées) | Publish non idempotent | **Upsert par fichier** : archive ancienne + recrée |
| 2 | Filtre Notion renvoie 0 résultat | `{{ }}` dans body JSON en texte brut | Body en **mode Expression** + `JSON.stringify` |
| 3 | Pas de match sur l'URL | `sourceUrl` tronqué `…/blob/main/` | Matcher sur `payload.properties.Source.url` |
| 4 | Page Notion fantôme (titre vide) | Fil parasite `Query Notion DB → Code (Parsing)` | Code (Parsing) reçoit **uniquement** Set Document |
| 5 | README publié | Filter trop large | Condition `path does not contain README` |
| 6 | Perte de 74 entrées en test | Branche archive lancée seule | Ne jamais single-step l'archive ; re-remplir = 1 run publish |
| 7 | Classifieur en panne (503) | Gemini free tier surchargé | Bascule sur **OpenAI** (`gpt-4o-mini`) |
| 8 | 1 seul fichier traité | « Fetch Test Event » = 1 item | Node **Search files and folders** (liste tout l'inbox) |
| 9 | Code_Parse ne sort qu'1 item | `Run Once for All Items` | **Run Once for Each Item** + `$json` + `$('…').item` |
| 10 | Delete « Referenced node doesn't exist » | Mauvais nom de node | `{{ $('Code_Parse').item.json.fileId }}` |
| 11 | GitHub 422 « sha wasn't supplied » | Fichier déjà existant (backlog) | `On Error = Continue` le temps d'un drain, puis remettre Stop |
| 12 | Fichiers non supprimés de Drive | Delete jamais atteint | Réparer l'amont ; Delete après Create réussi |

## 6. Exploitation & sécurité

- **Garde-fou** : Delete (Drive) toujours **après** un Create (GitHub) réussi → `On Error = Stop Workflow` en régime normal.
- **Idempotence** : intake = traité→supprimé ; publish = upsert par fichier ; pgvector = TRUNCATE+reload.
- **Jamais** de PAT/token dans le chat ; jamais d'édition manuelle de pgvector/Notion.

*SOP Lumina v1.0 — 2026-06-24.*
