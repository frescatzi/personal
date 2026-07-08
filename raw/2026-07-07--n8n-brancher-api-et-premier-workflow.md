---
type: raw
title: "n8n-Brancher-API-et-Premier-Workflow"
source_url: "drive:1ghdCnslkzKEvTg5ck4q08uq_OC8SvJUo"
captured: 2026-07-07
vault: ai-automation
brand: null
immutable: true
---

---
type: wiki
title: "n8n — Brancher les 3 API et créer le premier workflow multi-agents"
status: active
publish: notion
vault: ai-automation
brand: null
sources: ["raw/2026-06-24--n8n-brancher-api-et-premier-workflow.md"]
related: ["sop/Guide-Connexion-Agents-AI-n8n", "concept-limites-api-claude"]
updated: 2026-06-24
---

# n8n — Brancher les 3 API et créer le premier workflow multi-agents

Pour votre instance : **https://n8n.aftersunpeople.com**

---

## Partie 1 — Ranger les 3 clés API (Credentials)

À faire une seule fois. Les clés sont stockées chiffrées dans n8n ; ensuite vous les sélectionnez d'un clic.

1. Connectez-vous à n8n → menu de gauche **Credentials** → bouton **Add credential** (ou **+ Create credential**).
2. Ajoutez les trois, une par une :

| Chercher dans la liste | Champ à remplir | Votre clé |
|---|---|---|
| **OpenAI** | API Key | `sk-…` (ChatGPT) |
| **Anthropic** | API Key | `sk-ant-…` (Claude) |
| **Google Gemini (PaLM) API** | API Key | `AIza…` (Gemini) |

3. Pour chacune : collez la clé → **Save**. Si n8n propose un bouton « Test », il confirme que la clé fonctionne.

> Si une clé n'est pas encore créée, voyez le guide précédent (platform.claude.com, platform.openai.com, aistudio.google.com).

---

## Partie 2 — Le premier workflow : les agents se passent le relais

Objectif simple et démonstratif : **Claude rédige → ChatGPT critique → Gemini synthétise**. C'est ça, « les agents qui discutent entre eux ».

On utilise le nœud **Basic LLM Chain** (un par modèle), chacun avec son « cerveau » branché dessous.

### Étape par étape

1. **New workflow** (bouton en haut à droite).

2. Ajoutez le déclencheur : **+** → cherchez **« Trigger manually »** (*When clicking 'Test workflow'*). C'est le bouton de test ; on l'automatisera après.

3. **Bloc 1 — Claude rédige**
   - **+** après le trigger → cherchez **Basic LLM Chain** → ajoutez-le.
   - Sous le nœud, cliquez sur le **+** du connecteur **Chat Model** → choisissez **Anthropic Chat Model**.
     - Credential : votre clé Anthropic.
     - Model : **Claude Sonnet 4.6** (bon équilibre pour démarrer).
   - Revenez au Basic LLM Chain → champ **Prompt (Text)** :
     ```
     Rédige un court paragraphe (4-5 phrases) sur le sujet : les bienfaits du soleil.
     ```

4. **Bloc 2 — ChatGPT critique**
   - **+** après le Bloc 1 → **Basic LLM Chain**.
   - Chat Model → **OpenAI Chat Model** → credential OpenAI → un modèle GPT récent.
   - Prompt :
     ```
     Voici un texte : {{ $json.text }}

     Critique-le en 3 points concrets (style, exactitude, clarté).
     ```
   - `{{ $json.text }}` = la sortie du bloc précédent (Claude). C'est le « passage de relais ».

5. **Bloc 3 — Gemini synthétise**
   - **+** après le Bloc 2 → **Basic LLM Chain**.
   - Chat Model → **Google Gemini Chat Model** → credential Gemini → un modèle Gemini récent.
   - Prompt :
     ```
     Texte d'origine et critique reçus. Réécris une version finale améliorée du paragraphe en tenant compte de la critique : {{ $json.text }}
     ```

6. Cliquez **Test workflow** (en bas). Chaque nœud s'allume vert l'un après l'autre. Ouvrez le dernier nœud → onglet **Output** pour lire le résultat final.

```
[Test workflow] → [Claude rédige] → [ChatGPT critique] → [Gemini synthétise] → résultat
```

### Comprendre le passage de relais
`{{ $json.text }}` veut dire « prends le champ *text* de la sortie du nœud juste avant ». Pour récupérer un nœud précis plus loin dans la chaîne (pas seulement le précédent), on écrit `{{ $('Nom du nœud').item.json.text }}`.

---

## Partie 3 — Rendre le résultat utile, puis autonome

Une fois la chaîne validée, deux ajouts fréquents :

**Envoyer le résultat quelque part** : ajoutez un nœud final — Email, Gmail, Slack, ou « écrire dans Notion / Google Sheets » — qui prend `{{ $json.text }}` du dernier bloc.

**Le faire tourner seul** : remplacez le déclencheur manuel par un nœud **Schedule Trigger** (ex. tous les jours à 8h), puis **activez** le workflow (interrupteur *Active* en haut à droite). Votre serveur tournant 24/7, ça s'exécute sans vous.

---

## Conseils de démarrage

- Commencez avec **2 blocs** seulement, validez le passage de relais, puis ajoutez le 3ᵉ.
- Gardez des prompts **courts et précis** : moins de tokens = moins cher et plus rapide.
- Fixez un **plafond de dépense** sur les portails OpenAI et Anthropic pour éviter toute surprise.
- Pour aller plus loin (un agent « chef » qui appelle les autres comme outils), c'est le nœud **AI Agent** — à explorer en phase 2.
