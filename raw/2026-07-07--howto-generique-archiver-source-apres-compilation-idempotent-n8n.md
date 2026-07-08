---
type: raw
title: "HOWTO_GENERIQUE_archiver-source-apres-compilation-idempotent-n8n"
source_url: "drive:17Om7R4jqudG15xeMAOMiS8nlRNj7uW4B"
captured: 2026-07-07
vault: ai-automation
brand: null
immutable: true
---

# Marche à suivre générique — Archiver une source après compilation (workflow n8n idempotent)

> **But** : pattern réutilisable pour, après qu'un agent a compilé des sources brutes en livrables, **déplacer le brut vers un stockage d'archive et le supprimer de la source** — sans perte de données et sans doublons. Indépendant du cas LUMINA précis.
> **Principe directeur** : *l'IA (ou le process amont) étiquette, n8n exécute.* L'amont écrit un **manifeste** listant ce qui est prêt à archiver ; n8n fait le déplacement physique.

---

## 1. Pré-requis

1. **Un manifeste** produit par l'amont (fichier JSON dans le repo/stockage), p. ex. :
   ```json
   [ { "path": "raw/xxx.md", "category": "…", "processed_at": "YYYY-MM-DD" } ]
   ```
   Il liste **uniquement** les fichiers réellement compilés (pas ceux encore en attente).
2. **Un dossier d'archive** dans le stockage cible (Drive, S3…), **hors de tout dossier surveillé** par un workflow d'ingestion → sinon **boucle infinie** (archive ré-aspirée → re-compilée → ré-archivée).
3. **Credentials** du repo source (GitHub…) et du stockage cible, configurés dans n8n.

---

## 2. Le pattern, node par node

```
Trigger
  → Lire le manifeste (texte)
  → Décoder + parser → 1 item par fichier
  → Récupérer le fichier source (binaire)  ──existe──→ Uploader vers l'archive ──succès──→ Supprimer de la source
                                            └─absent (404) = skip                └─échec = skip
```

### a. Trigger
- **Schedule (cron)** recommandé pour démarrer : simple, prévisible, pas de boucle de re-déclenchement.
- Pour caler une minute précise et **désynchroniser** d'autres workflows, utiliser une **expression cron** (et non un « toutes les X heures » qui compte depuis l'activation).
- Alternative réactive (plus tard) : trigger sur push, **filtré** sur le manifeste.

### b. Lire le manifeste — **en texte**
- Récupérer le fichier manifeste **sans mode binaire** (on veut du JSON lisible).
- ⚠️ Beaucoup d'API (ex. GitHub) renvoient le contenu **encodé en base64** → à décoder à l'étape suivante.

### c. Décoder + parser (node Code)
- `Run Once for All Items`. Décoder si besoin, parser, **éclater en N items** :
  ```javascript
  const file = $input.first().json;
  const decoded = Buffer.from(file.content, 'base64').toString('utf-8'); // si base64
  return JSON.parse(decoded).map(e => ({ json: e }));
  ```
- Sortie : 1 item par fichier, avec au minimum `path` (et `category` si rangement voulu).

### d. Récupérer le fichier source — **idempotence + contenu** ⭐
- Récupérer chaque fichier (`path` dynamique), **en mode binaire** (on va le transporter).
- **On Error = `Continue (using error output)`** :
  - **Success** = le fichier existe → contenu prêt à uploader.
  - **Error (404)** = déjà archivé/supprimé → **skip**.
- C'est **le mécanisme d'idempotence** : récupérer le fichier sert à la fois de preuve d'existence et de source de contenu. Relancer 10× ne crée jamais de doublon.

### e. Uploader vers l'archive
- Envoyer la **pièce jointe binaire** (champ `data`) vers le dossier d'archive (par ID).
- Nom de fichier : lire depuis le binaire (`$binary.data.fileName`) pour éviter un champ recréé.
- **On Error = `Continue (using error output)`** → un upload raté part sur Error et ne déclenchera **pas** la suppression.

### f. Supprimer de la source — **le garde-fou** 🔒
- Brancher **uniquement sur la sortie Success de l'upload**.
- `path` à supprimer : via **pairing** vers le node de parse (`{{ $('<NodeParse>').item.json.path }}`), car le binaire/upload a remplacé `$json`.
- **On Error = `Continue`** → une suppression ratée laisse le fichier (retenté au prochain run).

---

## 3. Les 4 garde-fous (à reproduire systématiquement)

1. **Supprimer seulement après confirmation d'upload** → suppression branchée sur la sortie Success de l'upload.
2. **N'archiver que ce qui est dans le manifeste** → les fichiers non listés restent intacts.
3. **Archive hors du dossier surveillé** → pas de boucle.
4. **La suppression source ne doit pas re-déclencher les workflows aval** → vérifier leurs filtres.

---

## 4. Pièges classiques (et parades)

| Piège | Parade |
|---|---|
| Contenu illisible après lecture | L'API renvoie du **base64** → décoder avant `JSON.parse`. |
| « no binary file 'data' found » à l'upload | Un node intercalé (ex. Set/Edit Fields) **perd le binaire** → garder l'upload **directement** en aval du fetch binaire ; lire le nom via `$binary.data.fileName`. |
| Champs d'origine disparus après fetch/upload | `$json` a changé → **pairing** `{{ $('<Node>').item.json.x }}`. |
| Minute du cron non garantie | Utiliser une **expression cron** explicite, pas « toutes les X heures ». |
| Doublons à l'upload sur re-run | L'idempotence est côté **source** (le fichier disparaît après archivage). Le stockage cible peut accepter des doublons de nom : ne pas compter dessus pour dédupliquer. |
| Rangement par sous-dossier complexe | Un « search dossier » au milieu du flux **perd le binaire** → soit pré-créer les dossiers + table de correspondance, soit brancher sur une **branche séparée + merge**. Commencer **à plat** pour valider le pipeline. |
| Catégories qui dérivent | Le classeur amont (IA/LLM) **invente des noms libres** à chaque run → ne correspondent plus aux dossiers cibles. **Figer un vocabulaire contrôlé** (liste fermée de catégories) partagé entre le classeur et la table de dossiers ; repli sur un dossier racine pour tout hors-liste. |
| Outils locaux qui agissent sur des données périmées | L'automate (n8n) écrit **directement sur le remote** (`main`) sans toucher le clone local → le local prend du retard silencieusement. **Toujours synchroniser (`git pull`/`fetch`) avant toute opération locale** ; protéger toute purge/écriture par un **garde-fou de sync** (n'agir que si local = remote, sinon alerter). |

---

## 5. Ordre de validation recommandé

1. Tester chaque node isolément (`Execute step`).
2. Tester l'**upload** seul (la suppression non branchée) → vérifier dans le stockage.
3. Brancher la **suppression** et tester → vérifier la récupérabilité (archive + historique de version).
4. Tester l'**idempotence** : re-run complet → tout en 404 → 0 archivé, 0 supprimé.
5. **Activer** le workflow (cron).
