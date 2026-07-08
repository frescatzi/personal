---
type: raw
title: "concept-pipeline-memoire-wiki-git"
source_url: "drive:1SQxqruqfRkw63Yq6Idty2Naz0vncT108"
captured: 2026-07-07
vault: ai-automation
brand: null
immutable: true
---

---
type: wiki
title: "Concept — Pipeline mémoire depuis wiki Git (hub parallèle agents + humains)"
status: active
publish: none
vault: ai-automation
brand: null
sources:
  - "raw/2026-06-29--howto-generique-creer-la-memoire-agents-et-humains.md"
related:
  - "sop/sop-creer-memoire-agents-humains"
  - "sop/SOP_systeme-multi-agents-memoire-centrale-mcp-n8n"
  - "concept-archivage-n8n-idempotent"
  - "concept-intake-source-git"
  - "synthese-lumina-systeme-reference"
updated: 2026-06-29
---

# Concept — Pipeline mémoire depuis wiki Git (hub parallèle agents + humains)

> **Règle d'or :** Git = source unique. Les sorties (base vectorielle, outil de doc) se **régénèrent** depuis Git. On ne les édite jamais directement.

Voir [[sop/sop-creer-memoire-agents-humains]] pour l'implémentation Lumina (pgvector + Notion + Obsidian Git).

---

## Objectif

À partir d'un wiki en markdown versionné dans Git, alimenter **automatiquement** deux mémoires dérivées — **en parallèle** :
- **Mémoire des AGENTS** : base vectorielle (recherche sémantique, ex. pgvector).
- **Vue des HUMAINS** : outil de doc partagé (ex. Notion) pour lire/partager.

Les deux sont idempotentes et régénérables. Jamais en chaîne, jamais éditées à la main.

---

## Architecture (hub parallèle)

```
Éditeur markdown local (coffre + graphe)
     │  synchro Git automatique : commit + push
     ▼
Dépôt Git  wiki/  ← SOURCE DE VÉRITÉ
     │
┌────┴──────────────────────────────────┐
▼  n8n + API d'embeddings               ▼  n8n + API outil de doc
Base vectorielle (mémoire AGENTS)      Outil de doc (vue HUMAINS)
idempotent: TRUNCATE + reload          idempotent: upsert par fichier
```

---

## Partie 1 — Coffre & synchro vers Git

**Pourquoi c'est le point de départ :** rien ne peut être mémorisé ni publié tant que ça n'existe pas, proprement, dans le coffre. Le coffre est **physiquement le même dossier que le dépôt Git** → source unique versionnée. La **synchro Git automatique** pousse chaque changement, et c'est ce push qui réveille toute la chaîne aval.

- **Coffre = dépôt Git** — mêmes fichiers (`raw/`, `wiki/`, `CLAUDE.md`, `index.md`, `log.md`).
- **Synchro Git auto** (plugin Obsidian Git ou équivalent) — déclenche les pipelines.
- **`raw/` vs `wiki/`** — `raw/` = sources brutes curées ; `wiki/` = pages propres (souvent maintenues par un LLM). On ingère/publie le `wiki/`, pas `raw/`.

**Gouvernance :** accès restreint au départ (fondateur), élargi progressivement (chefs de département).

---

## Partie 2 — Mémoire des AGENTS (wiki → base vectorielle)

**Rôle :** transformer le wiki (texte lisible) en mémoire interrogeable **par le sens** (recherche sémantique), pas par mots-clés.

**Workflow : Ingestion (Wiki → base vectorielle)** — idempotent (TRUNCATE + reload à chaque run).

Chaîne type :
1. **Trigger** (manuel pour test, webhook en prod).
2. **Truncate** — vide la table avant rechargement → miroir exact du wiki, sans résidus.
3. **HTTP Request** (API Git « arbre récursif ») — liste tous les fichiers du repo en un appel.
4. **Split Out** — 1 item par fichier.
5. **Filter** (`type=blob` + `path` commence par `wiki/` + finit par `.md`) — seules les pages wiki.
6. **Get a file** (`File Path = {{ $json.path }}`, As Binary OFF) — contenu réel.
7. **Set Document** — mise en forme + métadonnées (`document`, `title`, `source`, `source_ref`, `collection`, `type`).
8. **Chunk** (Code, `for … of $input.all()`) — découpe chaque doc en morceaux. ⚠️ Boucler sur **TOUS** les items, pas `$input.first()`.
9. **Embed** (API embeddings, **modèle figé**) — chaque morceau → vecteur.
10. **Build Insert** → **Insert** — écrit les vecteurs dans la base.

**Vérif :** `SELECT source_ref, count(*) FROM <table> GROUP BY source_ref;` → uniquement `wiki/…`.

---

## Partie 3 — Vue des HUMAINS (wiki → outil de doc)

**Rôle :** version lisible et partageable pour les humains. Parallèle à la Partie 2, jamais en chaîne. Publication **idempotente par fichier** (archive ancienne + recrée la fraîche) → pas de doublons.

**Workflow : Publish (Wiki → outil de doc)** — upsert par fichier.

Chaîne type :
1. **Build payload** — construit la page ; pose une **clé de correspondance** (ex. URL source).
2. **Query par source** — cherche si une page existe déjà. ⚠️ Corps du filtre en **Expression + `JSON.stringify`** (sinon `{{ }}` envoyé en texte).
3. **Split Out** — éclate les résultats pour traiter l'ancienne page.
4. **Archive** — marque l'ancienne version archivée → évite les doublons.
5. **Create** — recrée la page à jour.

---

## Partie 4 — Automatisation (un seul déclencheur)

**Rôle :** relier tout pour une synchro sans intervention. N'automatiser que ce qui marche déjà à la main.

- **Webhook Git** (event = push, secret) → notifie n8n.
- **Webhook Trigger n8n** — filtre branche `main` + chemins `wiki/**`.
- **2× Execute Workflow** — lance en **parallèle** Ingestion (Partie 2) et Publish (Partie 3).

Les deux cibles étant idempotentes, re-déclencher à chaque push est totalement sûr.

---

## Erreurs fréquentes

| Symptôme | Cause | Solution |
|---|---|---|
| Un seul doc ingéré | Chunk en `$input.first()` | Corriger en `for … of $input.all()` |
| 404 sur Get a file | `File Path` en *Fixed* | Passer en *Expression* |
| Pas de texte / base64 | `As Binary` resté ON | Binary OFF sur Get a file |
| Outil de doc : 0 résultat au filtre | Corps non passé en Expression | `JSON.stringify` dans le body du filtre |
| Doublons | Pas d'archivage de l'ancienne page | Ajouter le step Archive avant Create |

---

## Points critiques

- **Idempotence** : base vectorielle = TRUNCATE+reload ; outil de doc = archive+recrée par fichier.
- **Modèle d'embedding figé** (changer = tout ré-encoder).
- Ne **jamais** éditer les sorties (base vectorielle, outil de doc) à la main — ce sont des dérivés.
- Adapter le pattern à chaque entité : changer owner/repo, la base, la collection. Mêmes pipelines réutilisables.
