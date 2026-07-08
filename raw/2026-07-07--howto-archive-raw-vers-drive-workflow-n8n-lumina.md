---
type: raw
title: "HOWTO_archive-raw-vers-drive_workflow-n8n-LUMINA"
source_url: "drive:1nhZ_2WS3mxwVIfizitEmjw0aXPAMcJ9d"
captured: 2026-07-07
vault: ai-automation
brand: null
immutable: true
---

# Marche à suivre — Build du workflow n8n `LUMINA — Archive — Raw→Drive`

> **But du document** : avoir une clarté exacte et durable du build. Qui fait quoi, quand, où, comment et pourquoi — node par node, avec les challenges rencontrés, les solutions, et les leçons apprises.
> **Statut** : v1 construite, testée de bout en bout et idempotente (2026-06-29).
> **Environnement** : n8n self-hosté `n8n.aftersunpeople.com` (Hetzner/Coolify), v2.10.4. Dossier n8n : infra LUMINA. Projet : Personal.

---

## 1. Ce que fait le workflow (en une phrase)

Après que Claude Code (`/ingest`) a compilé `raw/ → wiki/`, ce workflow lit le manifeste `_archive_queue.json`, **archive chaque fichier brut original dans Google Drive**, puis **le supprime de `raw/`** — mais **seulement après confirmation de l'upload**. Résultat : `wiki/` reste la seule source qui fait foi, `raw/` redevient vide, et le brut est conservé dans Drive comme filet de sécurité (provenance).

**Principe directeur : l'IA étiquette, n8n exécute.** Claude Code sait dans quelle catégorie il a rangé chaque fichier et l'écrit dans le manifeste ; n8n se contente du déplacement physique.

---

## 2. Place dans le pipeline Lumina

```
Drive Inbox → (intake n8n) → GitHub raw/ → Claude Code /ingest compile → wiki/ + _archive_queue.json
   → push (Obsidian Git) → cross-trigger pgvector + Notion
                          → [CE WORKFLOW] vide raw/ vers Drive
```

Ce workflow est le **maillon manquant** : il consomme `_archive_queue.json` pour vider `raw/`.

---

## 3. Décisions de design (actées en début de build)

| Question | Décision | Pourquoi |
|---|---|---|
| Déclencheur | **Schedule Trigger** (cron `0 15 * * * *`) | Simple, robuste, zéro risque de boucle liée aux pushs. La minute `:15` désynchronise des autres workflows qui partent à `:00`. |
| Nettoyage du manifeste | **Non** (on s'appuie sur l'idempotence) | Si le fichier n'existe plus dans `raw/`, le Node 4 le skip (404). Les entrées périmées sont inoffensives. Moins de nodes. |
| Arborescence Drive | **v1 : à plat dans `Archive-Raw`** ; catégories en **v2** | Insérer un « search dossier » au milieu du flux fait perdre la pièce jointe binaire. On a d'abord validé le pipeline complet (surtout le garde-fou de suppression), les catégories viendront ensuite. |

---

## 4. Chaîne de nodes (v1 finale)

```
Schedule Trigger
   → Get a file (manifeste)
   → Code (décoder + parser le manifeste)
   → Get raw file (idempotence + contenu)   ──Success──→ Google Drive : Upload ──Success──→ GitHub : Delete a file
                                             └─Error (404 = skip)                └─Error (skip)
```

### Node 1 — Schedule Trigger
- **Quoi** : point de départ. Réveille le workflow tout seul.
- **Réglage** : Trigger Interval = **Custom (Cron)**, Expression = **`0 15 * * * *`**.
- **Comment lire le cron** : n8n utilise ici **6 champs** → `[seconde] [minute] [heure] [jour du mois] [mois] [jour de semaine]`. Donc `0 15 * * * *` = seconde 0, minute 15, toutes les heures, tous les jours.
- **Pourquoi `:15`** : décaler de `:00` pour ne pas charger le processeur en même temps que les autres workflows.

### Node 2 — Get a file (lire `_archive_queue.json`)
- **Quoi** : lit le **manifeste** = la liste de courses (quels fichiers archiver, dans quelle catégorie).
- **Réglages** : Credential GitHub ; Resource **File** / Operation **Get** ; Owner `frescatzi` ; Repo `ai-automation` ; File Path `_archive_queue.json` ; **As Binary = OFF** ; Reference `main`.
- **Pourquoi As Binary OFF** : on veut lire du **texte JSON**, pas un fichier binaire. (Au Node 4 ce sera l'inverse.)
- **Sortie** : l'API GitHub renvoie le contenu dans le champ `content`, **encodé en base64** (`encoding: "base64"`).

### Node 3 — Code (décoder + parser le manifeste)
- **Quoi** : transforme « 1 gros fichier texte » en **N items** (1 par fichier à archiver), pour traiter chacun individuellement.
- **Mode** : `Run Once for All Items`, JavaScript.
- **Code** :
  ```javascript
  const file = $input.first().json;
  const decoded = Buffer.from(file.content, 'base64').toString('utf-8');
  const entries = JSON.parse(decoded);
  return entries.map(e => ({ json: e }));
  ```
- **Pourquoi décoder** : GitHub emballe le contenu en base64 (alphabet « sûr » pour le transport JSON). `Buffer.from(..., 'base64')` le déballe, puis `JSON.parse` lit le tableau.
- **Sortie** : N items, chacun `{ path, category, processed_at }`.

### Node 4 — Get raw file (idempotence + contenu) ⭐
- **Quoi** : pour chaque entrée, récupère le fichier dans `raw/`. **Le récupérer prouve qu'il existe** → on fusionne « vérifier l'existence » et « récupérer le contenu » en un seul node.
- **Réglages** : GitHub Get a file ; File Path = `{{ $json.path }}` ; **As Binary = ON** (propriété `data`) ; **Settings → On Error = `Continue (using error output)`**.
- **Le cœur de l'idempotence** :
  - **Success** → le fichier existe → on a son contenu binaire prêt pour Drive.
  - **Error (404)** → déjà archivé lors d'un run précédent → **skip** (sortie error laissée dans le vide).
  - Relancer le workflow 10× : les fichiers déjà traités tombent en 404 et sont ignorés. Zéro doublon, zéro plantage.
- **Pourquoi As Binary ON ici** (vs OFF au Node 2) : on **transporte un fichier** vers Drive ; le binaire est le format adapté et marche pour tout type (md, image, pdf…).

### Node 5 — Google Drive : Upload
- **Quoi** : dépose chaque fichier binaire dans `Archive-Raw` (le filet de sécurité).
- **Réglages** : Credential Google Drive ; Resource **File** / Operation **Upload** ; **Input Data Field Name = `data`** ; **File Name = `{{ $binary.data.fileName }}`** ; **Parent Folder = By ID → `16FCAzO6mVsQm8ZyC2HS951OsWV4p9Zgk`** (= `Archive-Raw`) ; **On Error = `Continue (using error output)`**.
- **Pourquoi `$binary.data.fileName`** : on lit le nom directement dans les métadonnées de la pièce jointe (déjà `2026-06-…​.md`).

### Node 6 — GitHub : Delete a file (le garde-fou) 🔒
- **Quoi** : vide `raw/` — l'étape sensible.
- **Branchement** : connecté **uniquement à la sortie Success de l'Upload**. Donc un fichier n'est supprimé **que si son upload a réussi**. Un upload raté part sur Error → jamais supprimé → retenté au prochain run.
- **Réglages** : GitHub Delete ; Owner `frescatzi` ; Repo `ai-automation` ; File Path = `{{ $('Code (décoder + parser le manifeste)').item.json.path }}` ; Commit Message = `archive: suppression de {{ $('Code (décoder + parser le manifeste)').item.json.path }} (copié dans Drive)` ; **Branch `main`** ; On Error = `Continue (using error output)`.
- **Pourquoi le pairing** (`$('Code …').item.json.path`) : après l'Upload, `$json` contient les infos Drive (plus le `path`). On va donc rechercher le chemin `raw/` d'origine dans le node Code ; n8n garde le lien (pairing) entre chaque item et sa ligne de manifeste.

---

## 5. Les garde-fous (et où ils vivent)

1. **Supprimer seulement après confirmation** → le Delete est branché sur la sortie **Success** de l'Upload (pas avant). Zéro perte de données.
2. **N'archiver QUE les fichiers du manifeste** → le Node 4 ne traite que les `path` listés. Les fichiers `raw/` non listés (ex. `README`, le fichier `vault: personal`) sont laissés intacts. *Vérifié en test.*
3. **Archive ≠ Inbox** → `Archive-Raw` est hors du dossier « Lumina Inbox » surveillé par l'intake → pas de boucle infinie. (Ils sont frères sous `OBSIDIAN`, pas l'un dans l'autre.)
4. **La suppression dans `raw/` ne déclenche pas pgvector/Notion** → ces workflows filtrent sur `wiki/`. À reconfirmer si on les retouche.

---

## 6. Challenges rencontrés → solutions → leçons

| # | Challenge | Cause | Solution | Leçon |
|---|---|---|---|---|
| 1 | `/ingest` → **404** au Node 2 | `_archive_queue.json` pas encore créé/poussé sur `main` | Lancer `/ingest` puis push via Obsidian Git | Le manifeste est un **pré-requis** produit par Claude Code + push, pas par n8n. Amorcer le pipeline avant de tester. |
| 2 | `/ingest` → **Unknown command** | La commande n'était pas installée dans le repo cible | Créer `.claude/commands/ingest.md` (et `sync.md`) dans le repo, puis redémarrer Claude Code | Une slash-command Claude Code est **scoped-projet** : elle doit vivre dans `<repo>/.claude/commands/`. |
| 3 | `cd` échoue | `~` déjà = `/Users/karter` (chemin doublé) + espace « Lumina AI » non échappé | Un seul des deux + **guillemets** autour du chemin | `~` ≠ à recombiner avec `/Users/...` ; toujours guillemeter les chemins avec espaces. |
| 4 | La minute du trigger non garantie | `Hours Between Triggers` compte depuis l'activation | Passer en **Cron `0 15 * * * *`** | Pour caler une minute précise (et désync de `:00`), utiliser une expression cron. n8n = cron **6 champs** (secondes en tête). |
| 5 | Contenu illisible au Node 2 | GitHub renvoie le contenu en **base64** | `Buffer.from(content, 'base64').toString('utf-8')` au Node 3 | Toujours décoder le `content` de l'API GitHub avant `JSON.parse`. |
| 6 | Upload → **« no binary file 'data' found »** | Le node **Edit Fields (Set)** intercalé ne transmettait pas le binaire | Brancher l'Upload **directement** en aval du node qui produit le binaire ; lire le nom via `$binary.data.fileName` | Garder le chemin binaire **direct** ; le Set node casse le transport de pièce jointe dans cette version. |
| 7 | `category`/`path` absents après le fetch | Le binaire **remplace** `$json` | **Pairing** : `{{ $('Code …').item.json.x }}` | Après un node qui change l'item (binaire, upload), récupérer les champs d'origine par référence au node source. |
| 8 | Sous-dossiers par catégorie compliqués | Un « search dossier » au milieu du flux **perd le binaire** | **v1 à plat** dans `Archive-Raw`, catégories en v2 | « Make it work, then make it nice » : valider le pipeline + le garde-fou critique d'abord. |
| 9 | Catégories libres qui dérivent | `/ingest` inventait un nom de catégorie différent à chaque run (`OAuth2 / Automation`, `Lumina / SOP`…) → ne matchait aucun dossier | **Vocabulaire contrôlé** : liste fermée de 7 catégories imposée dans `ingest.md`, partagée avec la `FOLDER_MAP` | Figer la taxonomie garde le classeur et les dossiers synchronisés. Voir §10. |
| 10 | Purge du manifeste qui se trompe / état local périmé | **n8n écrit directement sur `main`** (crée/supprime via l'API) → le **clone local** prend du retard, silencieusement | **`git pull` avant tout travail local** ; purge dotée d'un **garde-fou de sync** (n'agit que si local = `origin/main`) | `main` = vérité, le clone local = une photo qui se périme. *Un seul clone confirmé (pas de double checkout).* Voir §11. |

---

## 7. Tests réalisés (preuves)

- **Upload** : 13/13 sur Success, tous avec `parents = [Archive-Raw]`. Fichiers visibles dans Drive.
- **Delete (garde-fou)** : 13 commits de suppression sur `main` (récupérables : Drive + historique Git). `raw/` vidé des 13 fichiers ; `README` et `marketing-psychology` (non listés) laissés intacts.
- **Idempotence** : re-run complet → **Node 4 : Success = 0 / Error = 13** → Upload et Delete reçoivent 0 item. Aucun doublon, aucun plantage.

---

## 8. Infos techniques

- **n8n** : v2.10.4 self-hosté. Node GitHub v1.1, node Google Drive v3.
- **Repo** : `frescatzi/ai-automation`, branche `main`. Credential `GitHub account`.
- **Drive** : credential `Google Drive account - Live`. Dossier archive `Archive-Raw` = **ID `16FCAzO6mVsQm8ZyC2HS951OsWV4p9Zgk`** (sous `LUMINA VPS/LUMINA AI/OBSIDIAN`, frère de `Lumina Inbox`). Inbox à NE PAS utiliser : `1gjuB438pW7NKo4A-wbjaud_PH7ijnNUW`.
- **Sécurité** : aucun token/PAT dans les conversations — les creds vivent dans n8n.

---

## 9. Reste à faire / horizon

- **v2 — rangement par catégorie — FAIT (2026-06-29)** : `Archive-Raw/<category>/` via vocabulaire contrôlé + `FOLDER_MAP`. Voir §10.
- **Croissance du manifeste — RÉSOLU (2026-06-29)** : purge automatique par `/sync`, durcie avec garde-fou de synchronisation. Voir §11.
- **Activation** : workflow à passer en **Active** pour que le cron tourne. *(Fait : actif/publié, renommé `LUMINA-INTAKE-ARCHIVE/RAW→DRIVE`, rangé dans `LUMINA-01-INTAKE`.)*

---

## 10. v2 — Rangement par catégorie (construit 2026-06-29)

**Décision** : **vocabulaire de catégories contrôlé** + dossiers **pré-créés** + **repli racine**. `/ingest` doit classer chaque fichier dans **une seule** des 7 catégories canoniques (liste fermée imposée dans `.claude/commands/ingest.md` step 8, référencée par `sync.md`). Les 7 dossiers existent sous `Archive-Raw` ; le node Code mappe catégorie → `folderId`, avec repli sur la racine si une catégorie hors-liste apparaît. On a écarté la création de dossiers à la volée (dédup → search → create → merge, ~4-5 nodes) pour garder l'archive nette, sans quasi-doublons.

> **Pourquoi le vocabulaire contrôlé ?** Sans liste fermée, `/ingest` **inventait des catégories libres à chaque run** (`OAuth2 / Automation`, `Lumina / SOP`…) qui ne correspondaient à aucun dossier → tout tombait dans le repli racine. La liste figée garde `/ingest` et les dossiers Drive **synchronisés**.

**Les 3 pièces du dispositif :**

1. **`ingest.md` (step 8)** — règle de vocabulaire contrôlé : `category` ∈ {7 valeurs canoniques}, jamais hors-liste.
2. **Node Code** — table `FOLDER_MAP` (catégorie → ID de dossier) ; chaque item reçoit `folderId` (`FOLDER_MAP[e.category] || ROOT`).
3. **Node Upload** — **Parent Folder** = `{{ $('Code (décoder + parser le manifeste)').item.json.folderId }}` (pairing ; binaire préservé).

**Les 7 catégories canoniques** (Sphère A — Capacités IA & Build), dossiers sous `Archive-Raw` (`16FCAzO6mVsQm8ZyC2HS951OsWV4p9Zgk`) :

| Catégorie (clé du manifeste) | ID du dossier Drive |
|---|---|
| `IA & LLM` | `1RZ0WmENcUPmXF4eekWASxX5AXyWUSGKU` |
| `Agents & MCP` | `1l8cyjWuW-HDv3DSMuvXkn9D7dwqQwoRB` |
| `Automatisation (n8n)` | `1KfqnFO18q67UcKEClvQihwoQlU_qG4To` |
| `Mémoire & Connaissance` | `1oc2Zd-W2HxNgqDrDwxMjSucXmdMexk63` |
| `Infrastructure` | `1r3kLYktqT5Vgi79syy2irslQKhCFmIEq` |
| `Architecture & Stratégie` | `1NtEvNjbhO6kjjytIPkNnJbzZLKAfxnmY` |
| `Méthodes & SOP` | `1KRoiAb8j6bcfEpRWYlzQ2mWj8OIqMGxY` |

*(Une « Sphère B » agence/business — Offres, Clients, Ventes, Marketing, Marque, Finance, Légal — est conçue mais **non activée** ; à déployer à la création de l'agence.)*

**Ajouter une catégorie** : 1) créer le dossier dans `Archive-Raw` ; 2) ajouter sa ligne dans `FOLDER_MAP` (node Code) ; 3) l'ajouter à la liste fermée de `ingest.md`.

**Test e2e validé (2026-06-29)** : 12 fichiers archivés, chacun dans son dossier de catégorie (Méthodes & SOP ×4 · Agents & MCP ×2 · Automatisation (n8n) ×2 · Mémoire & Connaissance ×2 · Architecture & Stratégie ×1 · Infrastructure ×1), `raw/` vidé. Pipeline complet Inbox → archive opérationnel.

---

## 11. Hygiène du manifeste & synchronisation Git (2026-06-29)

**Pourquoi le clone local se périme.** n8n écrit **directement sur `main`** : l'intake *crée* des fichiers `raw/`, l'archivage en *supprime* + *commit*. Ces changements n'atterrissent **jamais** dans ton clone local tant que tu ne fais pas `git pull`. → Le dossier local devient « en retard » dès que n8n tourne, silencieusement. **`main` = la vérité ; le clone local = une photo à recharger.**

**Règle d'or : `git pull` avant tout travail local.** `/sync` le fait (pull + compile). Éviter `/ingest` seul (il ne pull pas) sauf juste après un pull.

**Purge du manifeste (auto, durcie).** À chaque `/sync`, l'étape PURGE de `ingest.md` :
1. **Garde-fou de sync** — `git fetch` + vérifie que local = `origin/main`. Si non aligné → **ne purge pas** et alerte (évite de purger sur des données périmées : l'erreur survenue une fois où elle a cru 12 fichiers présents alors qu'ils étaient archivés).
2. **Vérification disque réelle** — pour chaque entrée, teste si le fichier `path` existe encore dans `raw/`. Absent → retiré (archivé) ; présent → conservé (en attente/échec) ; tableau vide → `[]`.

→ Le manifeste ne contient jamais que des fichiers réellement présents dans `raw/` : **il ne grossit pas** avec des entrées 404.

**Note checkout** : vérifié le 2026-06-29 — il n'existe **qu'un seul clone** du repo (`~/Documents/Lumina AI/GitHub/ai-automation`). Les décalages observés (Obsidian Git « no changes », purge qui se trompe) venaient du **retard local sur `main`**, pas d'un double clone.
