---
type: raw
title: "POS-GENERIQUE_Calendrier-Contenu-Agent-Base_20260705"
source_url: "drive:1YLct_JC1G94ADn888uje8v2Dl1PHSwLh"
captured: 2026-07-07
vault: ai-automation
brand: null
immutable: true
---

# POS-GÉNÉRIQUE — Générateur de calendrier de contenu (agent → base, draft-only)

**Date :** 2026-07-05 (snapshot d'un état vérifié)
**Portée :** générique (réutilisable pour toute marque gérée via LUMINA OS).
**Objet :** produire à la demande, pour une entité « campagne/événement », un calendrier de contenu ~3 semaines avec copies prêtes à publier + un draft de newsletter, entièrement en brouillon dans la base de contenu, sans publication automatique.

## 1 · Architecture (n8n)
`Trigger {cible}` → `HTTP Lire l'entité` (query base, titre = cible) → `Code Brief & calendrier` (créneaux relatifs à la date + assemblage du prompt) → `IF Entité trouvée ?` → `HTTP Vérifier existant` (idempotence par relation) → `Code Décider` → `IF Déjà généré ?` → `ExecuteWorkflow Agent-rédacteur (query)` → `Code Parser` (JSON strict → 1 item par contenu + 1 newsletter, chacun portant son payload d'écriture complet) → `HTTP Créer page` (une fois par item) → `Code Compter` → `Notification` → `ExecuteWorkflow Mémoire` → `Set Résumé`.

Branches terminales : `Erreur - introuvable` et `Déjà généré`.

## 2 · Points clés
- Déclenchement **humain, par entité** — le lancement de campagne reste une décision.
- **API brute de la base** (HTTP Request + credential prédéfini) en lecture/écriture pour maîtriser relations/dates/blocs et éviter les représentations simplifiées.
- Le **Code construit le payload complet** de chaque contenu ; le node HTTP se contente de le POSTer par item.
- **Copie dans le corps** de la page si la base n'a pas de champ texte long ; ajouter une propriété **date** pour porter le calendrier.
- **Idempotence par relation** vers l'entité ; si des contenus existent déjà, on ne recrée pas.
- **Draft-only** : statut brouillon partout ; validation/publication humaines.

## 3 · Difficultés → 4 · Solutions
1. Objet littéral dans une expression de jsonBody → erreur de parsing → **JSON littéral** avec `{{ champ }}` dans les valeurs.
2. Base cible sans champ date ni champ copie → **ajouter la propriété date**, écrire la copie dans le corps.
3. Réponse d'agent bruitée → **parseur robuste** (retrait des fences, isolation `{`…`}`, erreur explicite).
4. Entités homonymes en doublon → sélection **By ID** (l'ID canonique = celui des workflows en prod).

## 5 · Leçons apprises
- Pas d'**objet littéral** dans une expression de jsonBody ; JSON en clair + interpolation des valeurs.
- **Lire le schéma réel** avant d'écrire ; ajouter les champs manquants nécessaires au process.
- Faire **construire le payload d'écriture par un Code**, garder le node HTTP « bête » → lisible, testable, réutilisable.
- L'**idempotence par relation** est le garde-fou d'un générateur relancé à la demande.
- Pousser du **code non trivial par API** via base64 (décodage UTF-8) évite les pièges d'échappement.
