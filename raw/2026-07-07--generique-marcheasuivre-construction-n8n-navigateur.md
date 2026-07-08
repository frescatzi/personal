---
type: raw
title: "Generique_MarcheASuivre_Construction-n8n-Navigateur"
source_url: "drive:1P1x1jdBM2dZH6qa36X6r-JGLGV07-okO"
captured: 2026-07-07
vault: ai-automation
brand: null
immutable: true
---

# Marche à suivre générique — Construire un workflow n8n via automatisation navigateur

**Usage :** Guide réutilisable pour n'importe quel projet nécessitant de construire ou modifier un workflow n8n en pilotant l'interface web via un agent (Claude in Chrome ou équivalent), plutôt que via l'API n8n ou un fichier JSON importé. Rédigé après une session de construction complète d'un workflow à 14 nodes.

---

## 1. Principes de base

1. **Toujours partir d'une spec écrite** (liste des nodes, types, contrat d'entrée) avant de toucher au canvas. Construire "à vue" dans l'interface sans plan multiplie les erreurs de connexion.
2. **La source de vérité, c'est le workflow live, pas le document.** Si la spec et l'état réel du workflow divergent (type de trigger, nom de node, credential manquante), vérifier l'interface avant d'agir et adapter la construction en conséquence plutôt que de suivre le doc à la lettre.
3. **Ne jamais saisir de secret** (mot de passe, clé API, client secret OAuth) à la place de l'utilisateur. Naviguer jusqu'au formulaire, expliquer ce qu'il faut coller, puis vérifier l'état après coup (rechargement de page, credential apparue dans la liste).
4. **Renommer chaque node dès sa configuration terminée.** Les noms par défaut ("HTTP Request", "Create a database page2", "Edit Fields1") rendent le canvas illisible dès le 5e node — renommer immédiatement évite d'avoir à tout retracer plus tard.

## 2. Pièges classiques de l'éditeur n8n (UI) et leurs parades

### a. Champs Expression / resourceLocator
- **Symptôme :** la valeur résolue contient un `=` en trop en début de chaîne (ex. `=abc123` au lieu de `abc123`), ce qui casse les lookups par ID.
- **Cause :** taper manuellement `={{ ... }}` alors que le mode "Expression" (ou le format `fx`) implique déjà le signe égal.
- **Parade :** ne jamais taper le `=` soi-même. Taper uniquement `{{ ... }}`.

### b. Auto-fermeture des accolades (CodeMirror)
- **Symptôme :** des `}}` en double apparaissent en fin d'expression.
- **Cause :** l'éditeur insère automatiquement les caractères fermants (`}}`, `)`, `"`) quand on tape l'ouvrant. Si on retape soi-même le fermant à la fin, ça duplique au lieu de "sauter par-dessus" — sauf si le curseur est resté juste avant le caractère auto-inséré (auquel cas taper le même caractère le "skip" sans dupliquer).
- **Parade :** taper uniquement `{{ ` puis le contenu, **sans** retaper le `}}` final ; laisser l'auto-fermeture gérer la fin. Si un doute existe sur l'état du champ, cliquer dedans, aller en fin de champ (`End`) et zoomer/lire le contenu réel avant de valider.

### c. Champs "name" / "value" dans les nodes Set / Edit Fields
- **Symptôme :** après avoir tapé le nom d'un champ, cliquer immédiatement sur son champ "value" et taper ne laisse rien dans la valeur.
- **Cause :** perte de focus lors de l'enchaînement rapide clic-tape-clic dans une liste de champs qui se redimensionne dynamiquement.
- **Parade :** séquencer clairement — clic sur "name" → taper → clic ailleurs (neutre) → **re-clic** explicite sur "value" → taper → vérifier par screenshot avant de passer au champ suivant.

### d. Panneaux de recherche de node tronqués
- **Symptôme :** à la première ouverture du panneau "What happens next?" (ou d'un sous-panneau de catégorie), le contenu apparaît coupé au bord droit de la fenêtre.
- **Parade :** le clic sur l'élément partiellement visible fonctionne quand même dans la plupart des cas. Sinon, fermer le panneau (clic sur le canvas) et le rouvrir corrige généralement le rendu.

### e. Dropdowns qui se referment avant la sélection
- **Symptôme :** cliquer sur une option d'un dropdown (ex. "By ID" dans un sélecteur de workflow) ne change rien, le dropdown semble se refermer tout seul.
- **Parade :** privilégier la navigation clavier (`Down`/`Up` puis `Return`) plutôt que le clic sur coordonnées quand un dropdown se comporte de façon instable.

## 3. Tester sans polluer les données réelles

- **Ne pas cliquer "Execute step" sur un node dont la chaîne en amont n'a pas de données en cache.** n8n relance alors l'exécution complète depuis le trigger avec des données de test vides/nulles, ce qui peut créer de vraies ressources parasites (fichiers, pages, événements) si le workflow a des effets de bord (API Drive, Notion, Calendar...).
- Utiliser **"set mock data"** pour tester un node isolément sans dépendre de l'exécution complète en amont.
- Si des doublons de test sont malgré tout créés, ne pas les supprimer automatiquement sans confirmation explicite de l'utilisateur — signaler clairement quoi a été créé (IDs, noms) et laisser la suppression à l'utilisateur.
- Pour un corps JSON long saisi en plusieurs morceaux, revérifier visuellement le **début** du champ (pas seulement la fin) avant de considérer la saisie comme fiable — un défilement automatique peut masquer une corruption en tête de chaîne.

## 4. Architecture : nodes de synchronisation et de sortie

- Quand deux branches parallèles doivent se rejoindre avant un node commun (ex. deux appels API indépendants suivis d'une agrégation), insérer un node **Merge** entre les deux branches et ce node commun. Sans lui, n8n exécute le node suivant une fois par branche entrante au lieu d'attendre les deux.
- Le node Merge peut servir de simple **point de synchronisation** : les nodes en aval peuvent continuer à lire les données originales via une référence explicite au node source (`$('NomDuNode').item.json...` en n8n) plutôt que de dépendre du contenu réellement produit par le Merge.
- Avant d'ajouter un node de type "réponse" (ex. "Respond to Webhook"), **vérifier le type du trigger du workflow**. Un node de réponse à un webhook échoue à l'exécution si le workflow n'a pas été déclenché par un vrai node Webhook (ex. déclenché via "Execute Workflow" à la place). Dans ce cas, un simple node Set en bout de chaîne suffit : n8n renvoie automatiquement l'output du dernier node exécuté comme valeur de retour du sous-workflow.

## 5. Checklist avant de considérer un workflow "prêt"

- [ ] Tous les nodes de la spec présents et connectés, aucun node orphelin.
- [ ] Chaque node renommé de façon descriptive.
- [ ] Aucun credential saisi par l'agent — uniquement par l'utilisateur.
- [ ] Testé au moins une fois de bout en bout avec des données réalistes (pas seulement une relecture visuelle).
- [ ] Toute donnée de test parasite créée pendant la construction est identifiée et signalée à l'utilisateur.
- [ ] Le workflow est publié/activé si c'est l'usage voulu.

## 6. Techniques avancées (ajout du 04.07.2026 — refactoring en sous-workflows)

1. **API REST interne pour les gros refactorings** : depuis la console de l'onglet n8n connecté, `fetch('/rest/workflows/<id>', {credentials:'include', headers:{'browser-id': localStorage.getItem('n8n-browserId')}})` permet GET/POST/PATCH de workflows entiers (nodes, connections, `versionId` requis pour PATCH). Incomparablement plus fiable que le canvas pour créer N sous-workflows ou réécrire des expressions en masse. Toujours vérifier ensuite en rechargeant l'éditeur. La mutation directe du store Pinia n'est jamais détectée par la sauvegarde — ne pas essayer.
2. **Sémantique des 0 items** : un node qui sort 0 item ne déclenche pas ses successeurs et le run est « Success » quand même. Sur tout node de recherche dont le résultat vide a un sens métier (upsert), activer **Always Output Data** — et le code de vérification en aval doit tester la présence d'un champ (`i.json.id`), jamais `items.length` (l'item vide compte pour 1).
3. **Contrat de sortie d'un sous-workflow** : sa sortie = la sortie de son dernier node. Terminer chaque sous-workflow par un node Set « Sortie » avec **executeOnce** pour garantir exactement 1 item ; dans l'orchestrateur, référencer les résultats en `$('Node').first().json.x` (immunité au nombre d'items).
4. **Audit des expressions** : une valeur d'expression doit commencer par `=` et fermer ses `}}`. Symptôme d'une expression malformée : le texte brut `{{ ... }}` apparaît tel quel dans l'output. Auditer systématiquement après copier-coller ou édition API.
5. **Session expirée** : symptômes jumeaux « Autosave failed: Unauthorized » (toast en boucle) et « Error fetching options » dans les sélecteurs. Se reconnecter avant de diagnostiquer autre chose — les mappings par nom survivent généralement.

---
*Rédigée le 03.07.2026 à partir d'une session de construction n8n via navigateur ; complétée le 04.07.2026 (refactoring en sous-workflows par API REST). Réutilisable pour tout futur projet n8n, indépendamment du contexte métier.*
