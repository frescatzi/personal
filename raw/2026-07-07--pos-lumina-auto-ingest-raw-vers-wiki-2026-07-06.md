---
type: raw
title: "POS-LUMINA_Auto-Ingest-Raw-vers-Wiki_2026-07-06"
source_url: "drive:1Pck2yCuYNDFIAv-HnPU5Ei9K_B6IY8bj"
captured: 2026-07-07
vault: ai-automation
brand: null
immutable: true
---

# POS-LUMINA — Auto-Ingest `raw/ → wiki/` (full automatique)

> **Objet :** automatiser l'unique étape manuelle du pipeline Lumina — la compilation `raw/ → wiki/` — avec le **moteur Claude** (qualité + contexte du coffre) orchestré par **n8n**, en **full automatique**, sur un **runner Claude headless** hébergé sur Coolify/Hetzner.
> **Déclencheur :** push GitHub touchant `raw/**`.
> **Mode :** écriture directe sur `main`, sans revue humaine. Filet conservé : pages auto-créées en `status: draft` / `publish: none` → indexées mais **n'atteignent les agents qu'après promotion manuelle** en `active`+`notion`.
> **Environnement :** n8n `n8n.aftersunpeople.com` · repo `frescatzi/ai-automation` · runner conteneur sur Coolify.
> **Date :** 2026-07-06. **Gate CEO :** clé API Anthropic (coût), PAT/deploy key git (écriture), secret webhook.

---

## 1. Place dans le pipeline

```
Drive Inbox → (intake n8n) → GitHub raw/  ──push raw/**──▶  n8n Webhook
                                                              │  HTTP POST /ingest (secret)
                                                              ▼
                                              Runner Claude headless (Coolify)
                                              1. git pull --rebase           ← remet le coffre à jour (règle la synchro)
                                              2. Claude /ingest              ← lit les nouveaux raw, rédige wiki (draft)
                                              3. maj index.md + log.md
                                              4. git commit + push  →  main
                                                              │
                              push wiki/** ─────┴──▶ pgvector + Notion (workflows existants)
                              archive cron ───────▶ balaie raw/ vers Drive
```

L'auto-ingest est le **chaînon manquant automatisé** entre l'intake (qui remplit `raw/`) et l'archivage (qui vide `raw/`).

---

## 2. Prérequis (secrets — gate CEO, jamais dans le chat)

| Secret | Où | Rôle |
|---|---|---|
| `ANTHROPIC_API_KEY` | env Coolify du runner | fait tourner Claude Code |
| `GIT_PAT` (ou deploy key SSH, scope `contents:write` sur `ai-automation`) | env Coolify du runner | pull + push sur `main` |
| `INGEST_SECRET` (chaîne aléatoire) | env runner **et** credential n8n | authentifie l'appel n8n → runner |
| Webhook GitHub `secret` | GitHub repo settings **et** n8n | signe les événements push |

---

## 3. Le runner Claude headless (Coolify)

Petit service HTTP qui expose `POST /ingest`. À chaque appel authentifié : sync → compile → commit → push.

### 3.1 Image / build (Dockerfile)

```dockerfile
FROM node:20-slim
RUN apt-get update && apt-get install -y git ca-certificates && rm -rf /var/lib/apt/lists/*
RUN npm install -g @anthropic-ai/claude-code
WORKDIR /app
COPY package.json server.js ingest-prompt.md ./
RUN npm install
# Clone au démarrage (ou volume persistant monté sur /vault)
CMD ["node", "server.js"]
```

### 3.2 Variables d'environnement (Coolify)

```
ANTHROPIC_API_KEY=...          # gate CEO
GIT_PAT=...                    # gate CEO
INGEST_SECRET=...              # partagé avec n8n
REPO_URL=https://x-access-token:${GIT_PAT}@github.com/frescatzi/ai-automation.git
GIT_AUTHOR_NAME=lumina-auto-ingest
GIT_AUTHOR_EMAIL=bot@aftersunpeople.com
VAULT_DIR=/vault
MAX_FILES_PER_RUN=15           # plafond de coût
```

### 3.3 Endpoint `POST /ingest` — étapes exactes

1. **Auth** : rejeter si header `X-Ingest-Secret` ≠ `INGEST_SECRET` (403).
2. **Verrou** : si un ingest tourne déjà → répondre `{status:"busy"}` (202) et sortir. (Évite deux runs concurrents.)
3. **Sync** : `cd $VAULT_DIR` → `git fetch origin` → `git checkout main` → `git reset --hard origin/main`.
4. **Compiler** : lancer Claude Code non-interactif dans le coffre :
   ```bash
   claude -p "$(cat /app/ingest-prompt.md)" \
     --permission-mode acceptEdits \
     --allowedTools Read Write Edit Bash Grep Glob \
     --max-turns 40
   ```
5. **Rien à faire ?** `git status --porcelain` vide → répondre `{status:"noop"}`.
6. **Commit + push** :
   ```bash
   git add -A
   git commit -m "auto-ingest: raw→wiki (draft) [skip ci]"
   git pull --rebase origin main    # absorbe un éventuel commit concurrent (Obsidian Git)
   git push origin main
   ```
7. **Réponse** : renvoyer le résumé (dernière ligne `log.md` + liste des pages créées/màj) à n8n.

> ⚠️ Le runner est le **seul writer de `wiki/`**. Les humains n'écrivent que dans `raw/` (via l'intake). Ça évite les conflits.

---

## 4. Le prompt d'ingest (`ingest-prompt.md`) — la qualité éditoriale

C'est le cœur : il donne à Claude la règle exacte de compilation du coffre. Contenu :

```
Tu es le bibliothécaire du coffre ai-automation. Applique le workflow INGEST du CLAUDE.md.

POUR CHAQUE fichier raw/ NON encore reflété dans wiki/ :
1. DÉDUP D'ABORD (anti-doublon obligatoire) :
   - Si un autre raw/ a un CORPS identique (même contenu, date différente) → NE crée PAS de page ; c'est une re-capture. Passe.
   - Si une page wiki/ couvre déjà le sujet → METS-LA À JOUR (ajoute/nuance/date), n'en crée pas une seconde.
2. SINON crée wiki/concept-<slug>.md ou wiki/sop/sop-<slug>.md (kebab-case, SANS date), en SYNTHÉTISANT (pas un copier-coller du brut).
3. Frontmatter OBLIGATOIRE : type: wiki · title · status: draft · publish: none · vault: ai-automation · brand · sources: [le(s) raw] · related: [backlinks] · updated: <aujourd'hui>.
4. Ajoute les backlinks [[...]] internes.
5. NE TOUCHE JAMAIS aux fichiers raw/ (immuables). Ne mets JAMAIS publish: notion ni status: active (promotion = décision humaine).
6. Exclus : vault: personal (ex. marketing), et les passations/milestones/sessions/spec (historique, pas du wiki) — laisse-les pour l'archivage.
7. Mets à jour index.md (ajoute les nouvelles pages) et ajoute une ligne dans log.md (## [date] ingest | ...).
Borne-toi à MAX_FILES_PER_RUN fichiers par exécution.
```

---

## 5. Workflow n8n `LUMINA — Auto-Ingest — Raw→Wiki`

Chaîne : `Webhook (GitHub push) → Code (filtre) → IF → HTTP Request (runner) → (notif)`

1. **Webhook Trigger** — méthode POST, authentification par `secret` GitHub. URL à coller dans le webhook GitHub.
2. **Code — filtre anti-boucle** (Run Once for All Items) :
   - Garder l'événement seulement si `ref = refs/heads/main` **ET** au moins un fichier **ajouté ou modifié** sous `raw/` (`commits[].added` / `commits[].modified`).
   - **Ignorer les suppressions** (`commits[].removed`) → c'est le workflow d'**archivage** qui supprime raw/ ; ne pas re-déclencher dessus.
   - Ignorer les pushs de l'auteur `lumina-auto-ingest` (le runner touche `wiki/`, pas `raw/`, donc normalement déjà filtré — ceinture + bretelles).
3. **IF** — s'il reste des raw ajoutés/modifiés → continuer ; sinon stop (no-op).
4. **HTTP Request → runner** : `POST https://runner.aftersunpeople.com/ingest`, header `X-Ingest-Secret = {{$credential}}`, body `{ reason: "push {{ $json.after }}" }`. **Timeout élevé** (ex. 300 s — Claude prend des minutes). **Retry Off** (pas de double run).
5. **(Option) Notification** — envoyer le résumé renvoyé par le runner (e-mail/Telegram) : « X pages créées/màj en draft ».

### GitHub webhook
Repo `frescatzi/ai-automation` → Settings → Webhooks → Add : Payload URL = l'URL du Webhook n8n · Content type `application/json` · Secret = le même que n8n · Événement = **Just the push event**.

---

## 6. Garde-fous automatiques (remplacent la revue humaine)

1. **Dédup par empreinte** (dans le prompt) → tue les re-captures « même contenu, date différente » (la cause des doublons constatés).
2. **Anti-boucle** → n8n ne se déclenche que sur `raw/` **ajouté/modifié** ; les suppressions d'archivage n'entraînent rien.
3. **Verrou de concurrence** → un seul ingest à la fois.
4. **Plafond de coût** → `MAX_FILES_PER_RUN`.
5. **`draft` / `publish: none`** → rien n'atteint les agents/Notion sans promotion manuelle.
6. **Runner = seul writer de `wiki/`** + `git pull --rebase` avant push → pas de conflit avec Obsidian Git.
7. **Timeout** côté n8n et `--max-turns` côté Claude.

---

## 7. Tests de recette

1. Déposer un fichier de test dans l'Inbox Drive → l'intake le range dans `raw/` (push) → le webhook déclenche le runner.
2. Vérifier qu'une page `wiki/…` en `status: draft` apparaît sur `main`, avec frontmatter + `sources:` + backlinks, et que `index.md`/`log.md` sont à jour.
3. **Idempotence** : re-pousser le même raw (même contenu) → **aucune** nouvelle page (dédup).
4. Vérifier que pgvector s'est mis à jour (le `source_ref` de la nouvelle page apparaît), Notion aussi si la page est ensuite promue.
5. Vérifier qu'une **suppression** d'archivage sur `raw/` **ne déclenche pas** un ingest (anti-boucle).

---

## 8. Incidents probables

| Symptôme | Cause | Action |
|---|---|---|
| Boucle d'ingests | n8n déclenche sur suppression raw/ | Vérifier le filtre `added/modified` uniquement |
| Doublons de pages | dédup non appliquée | Renforcer la règle 1 du prompt ; comparer les corps (md5) |
| Push rejeté (non-fast-forward) | commit Obsidian concurrent | `git pull --rebase` avant push (déjà prévu) ; retry |
| Runs qui se chevauchent | pas de verrou | Vérifier le lock ; n8n Retry Off |
| Coût qui grimpe | trop de fichiers/run | Baisser `MAX_FILES_PER_RUN` |
| Page part vers les agents sans validation | `status: active`/`publish: notion` mis par erreur | Le prompt l'interdit ; garder la promotion manuelle |

---

## 9. Évolutions

- **Promotion assistée** : un second workflow qui, sur ta demande, passe une page `draft → active` + `publish: notion` (la seule étape qui la rend visible aux agents).
- **Réactivité** : déjà sur push (maximale). Ajouter un Schedule de secours (ex. horaire) au cas où un webhook est manqué.
- **Revue optionnelle** : si un jour tu veux un filet humain, basculer l'écriture sur une branche `auto-ingest` + PR sans rien changer d'autre.

---

*POS-LUMINA — Auto-Ingest raw→wiki (full automatique) — v1 — 2026-07-06. Double sauvegarde : projet Claude « Claude Lessons » + Google Drive (règle CLAUDE.md).*
