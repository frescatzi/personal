---
type: raw
title: "HOWTO_GENERIQUE_creer-la-memoire-agents-et-humains"
source_url: "drive:1LH8Q_c5IF_4XQwd1qXE5YiiIQ6YZ6Csv"
captured: 2026-07-07
vault: ai-automation
brand: null
immutable: true
---

---
title: "How-To (générique) — Créer la mémoire IA (agents + humains) depuis un wiki Git"
type: howto
status: active
tags: [n8n, vector-db, notion, github, markdown, memory, idempotence, generic]
version: 1.0
last_review: 2026-06-25
---

# How-To (générique) — Créer la mémoire IA (agents + humains) depuis un wiki Git

> Version **générique et réutilisable** (aucune marque, aucun identifiant réel). Remplace les `<placeholders>` par tes valeurs.

## Objectif
À partir d'**un wiki en markdown versionné dans Git** (la source de vérité), alimenter **automatiquement** deux mémoires dérivées : la **mémoire des AGENTS** (base vectorielle, pour la recherche sémantique) et la **vue des HUMAINS** (un outil de doc partagé type Notion, pour lire/partager). Les deux sont **parallèles, idempotentes, régénérables** — jamais en chaîne, jamais éditées à la main.

## Vue d'ensemble (hub parallèle)
```
   Éditeur markdown local (coffre + graphe)
        │  synchro Git automatique : commit + push
        ▼
   Dépôt Git  wiki/  (SOURCE DE VÉRITÉ)
        │
   ┌────┴──────────────────────────────┐
   ▼ n8n + API d'embeddings             ▼ n8n + API de l'outil de doc
 Base vectorielle (mémoire AGENTS)     Outil de doc (vue HUMAINS)
 idempotent: TRUNCATE + reload         idempotent: upsert par fichier
```
> **Règle d'or :** Git = source unique. Les sorties (base vectorielle, outil de doc) se **régénèrent** depuis Git. On ne les édite jamais directement.

---

## Partie 1 — Le coffre & la synchro vers Git
**À quoi ça sert / pourquoi ici.** C'est le point de départ humain de la connaissance : on y écrit, organise et **visualise** le savoir avant qu'il n'irrigue le reste. On commence par là parce que rien ne peut être mémorisé ni publié tant que ça n'existe pas, proprement, dans le coffre. Le coffre est **physiquement le même dossier que le dépôt Git** → source de vérité unique et versionnée. Une **synchro Git automatique** (ex. plugin Git de l'éditeur) pousse chaque changement, et c'est ce push qui réveille toute la chaîne en aval (Parties 2 à 4). La vue graphe / backlinks aide à comprendre comment les idées se relient — d'autant plus utile que la base grossit.
**Accès (gouvernance).** Au départ, **accès restreint** (souvent une seule personne). À mesure que l'équipe grandit et monte en compréhension, on **élargit** l'accès par paliers (ex. responsables d'abord).

**Éléments et leur rôle :**
- **Coffre = dépôt Git** — mêmes fichiers (`raw/`, `wiki/`, fichier de schéma type `CLAUDE.md`, `index.md`, `log.md`).
- **Synchro Git automatique** — commit + push réguliers ; c'est elle qui **déclenche** les pipelines (Partie 4).
- **`raw/` vs `wiki/`** — `raw/` = sources brutes curées ; `wiki/` = pages propres (souvent maintenues par un LLM). On ingère/publie le `wiki/`, pas `raw/`.

---

## Partie 2 — Mémoire des AGENTS (wiki → base vectorielle)
**À quoi ça sert / pourquoi ici.** Transforme le wiki (texte lisible) en **mémoire interrogeable par les agents IA**. Vient après la Partie 1 car elle **consomme** le wiki validé. Son rôle : permettre aux agents de retrouver l'info **par le sens** (recherche sémantique), pas par mots-clés. On **reconstruit la table à chaque run** (TRUNCATE + reload) → miroir exact du wiki, sans doublons. C'est ce qui rend les agents utiles (ils répondent à partir de la connaissance maison) ; c'est donc le **socle opérationnel** des agents.

**Workflow : « Ingestion (Wiki→base vectorielle) »** — lit tout le `wiki/` et le charge ; idempotent.

**Chaîne et rôle de chaque node :**
1. **Trigger** — démarre (manuel pour test, webhook en prod).
2. **Truncate** (`TRUNCATE <table_mémoire>;`) — vide la table avant rechargement → idempotence.
3. **HTTP Request** (API Git « arbre récursif », ex. `/repos/<owner>/<repo>/git/trees/<branche>?recursive=1`) — récupère la **liste de tous les fichiers** en un appel.
4. **Split Out** (champ `tree`) — un **item par fichier**.
5. **Filter** (`type=blob` ET `path` commence par `wiki/` ET finit par `.md`) — ne garde que les pages du wiki.
6. **Get a file** (`File Path = {{ $json.path }}`, **As Binary OFF**) — récupère le **contenu réel**.
7. **Set Document** — met en forme + métadonnées (`document`, `title`, `source`, `source_ref`, `collection`, `type`).
8. **Chunk** (Code, `for … of $input.all()`) — découpe chaque doc en **morceaux** (boucle sur TOUS les docs).
9. **Embed** (API d'embeddings, **modèle figé**) — chaque morceau → **vecteur**.
10. **Build Insert** — prépare la requête d'insertion.
11. **Insert** (`{{ $json.query }}`) — écrit les vecteurs dans la base.

**Vérif :** `SELECT source_ref, count(*) FROM <table_mémoire> GROUP BY source_ref;` → uniquement des `wiki/…`.

---

## Partie 3 — Vue des HUMAINS (wiki → outil de doc)
**À quoi ça sert / pourquoi ici.** Crée la version **lisible et partageable** pour les humains. **Parallèle** à la Partie 2 : même source (wiki), mais les deux ne se parlent jamais. On sépare car agents et humains ont des **besoins différents** (sémantique vs lecture/partage/mobile). Publication **idempotente par fichier** (archive l'ancienne page + recrée la fraîche) → pas de doublons. L'outil de doc est **optionnel et non porteur** : le système tourne sans lui. Prend toute son importance quand **l'équipe grandit**.

**Workflow : « Publish (Wiki→outil de doc) »** — publie/met à jour les pages ; upsert par fichier.

**Chaîne et rôle de chaque node :**
1. **Build payload** (Code) — construit la page depuis le fichier wiki ; pose une **clé de correspondance** (ex. `properties.Source.url`).
2. **Query par source** — cherche si une page existe **déjà** (recherche par la clé). ⚠️ Corps du filtre en **Expression + `JSON.stringify`** (sinon `{{ }}` envoyé en texte). Header de version d'API requis.
3. **Split Out** (résultats) — éclate pour traiter l'ancienne page.
4. **Archive** (marquer l'ancienne page archivée) — **évite les doublons**.
5. **Create** — **recrée la page à jour**.

> ⚠️ Conserver une **fiche d'identifiants** séparée (DB id, data_source id, intégration…) hors de cette procédure générique.

---

## Partie 4 — Automatisation (un seul déclencheur)
**À quoi ça sert / pourquoi ici.** **Relie tout** pour une synchro sans intervention. Vient en dernier car elle **suppose** que les Parties 2 et 3 existent et sont **idempotentes** (on n'automatise que ce qui marche déjà). Déclencheur = **push Git** sur le `wiki/` : à chaque changement, les deux pipelines se relancent **en parallèle**. Idempotents → re-déclencher à chaque push est **sûr**. Résultat : on écrit dans le coffre, et secondes plus tard, **agents et humains ont la version à jour** sans toucher à n8n.

**Éléments et leur rôle :**
- **Webhook côté dépôt Git** (event = push, secret) — notifie n8n à chaque changement.
- **Webhook Trigger n8n** (« Orchestrateur — Sync ») — filtre branche `main` + chemins `wiki/**`.
- **2× Execute Workflow** — lance **en parallèle** l'Ingestion et le Publish.

---

## Pré-requis
- Dépôt Git avec `wiki/*.md` (voir How-To « Intake source → Git »).
- Base vectorielle + table mémoire (dimension = celle du modèle d'embedding ; index ANN type HNSW cosinus).
- Credentials n8n : Git, API d'embeddings, API de l'outil de doc. **Modèle d'embedding figé** (en changer = tout ré-encoder).

## Erreurs fréquentes
- **Un seul doc ingéré** → Chunk en `$input.first()`. Corriger en `for … of $input.all()`.
- **404 sur Get a file** → File Path en *Fixed* au lieu d'*Expression*.
- **Pas de texte / base64** → `As Binary` resté ON sur Get a file.
- **Outil de doc : 0 résultat au filtre** → corps non passé en *Expression + JSON.stringify*.
- **Doublons** → pas d'archivage de l'ancienne page, ou mauvais champ de matching.

## Points critiques
- **Idempotence** : base vectorielle = TRUNCATE+reload ; outil de doc = archive+recrée par fichier.
- **Modèle d'embedding figé** (dimension constante).
- Ne **jamais** éditer les sorties à la main (dérivés).

## Adaptation
Même pattern partout : changer owner/repo, la base de doc, la collection. Chaque entité (marque/projet) a **son propre coffre** et réutilise les mêmes pipelines.

## Version
v1.0 — 2026-06-25 (générique)
