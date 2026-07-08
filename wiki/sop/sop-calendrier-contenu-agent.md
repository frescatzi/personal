---
type: wiki
title: SOP — Générateur de calendrier de contenu (agent → base, draft-only)
status: active
publish: notion
vault: ai-automation
brand:
sources:
  - raw/2026-07-06--pos-generique-calendrier-contenu-agent-base-20260705.md
related:
  - wiki/sop/sop-outreach-backfill.md
  - wiki/sop/sop-cablage-orchestrateur-subagents.md
  - wiki/concept-memoire-vivante-agents.md
  - wiki/synthese-lumina-ai-os.md
updated: 2026-07-06
---

# SOP — Générateur de calendrier de contenu (agent → base, draft-only)

## Principe

Produire à la demande, pour une entité « campagne/événement », un calendrier de contenu ~3 semaines avec copies prêtes à publier + un draft de newsletter, entièrement en brouillon dans la base de contenu — **sans publication automatique**. Le lancement de campagne reste une décision humaine.

## Architecture n8n

```
Trigger {cible}
→ HTTP Lire l'entité (query base, titre = cible)
→ Code Brief & calendrier (créneaux relatifs à la date + assemblage du prompt)
→ IF Entité trouvée ?
  → HTTP Vérifier existant (idempotence par relation)
  → Code Décider
    → IF Déjà généré ?
      → ExecuteWorkflow Agent-rédacteur (query)
      → Code Parser (JSON strict → 1 item/contenu + 1 newsletter, chacun avec payload complet)
      → HTTP Créer page (une fois par item)
      → Code Compter
      → Notification
      → ExecuteWorkflow Mémoire
      → Set Résumé
    → Branche : Déjà généré (idempotent)
  → Branche : Erreur — introuvable
```

## Points clés de l'implémentation

**API brute de la base** (HTTP Request + credential) en lecture/écriture — ne pas utiliser les représentations simplifiées des intégrations natives pour maîtriser les relations, les dates et les blocs.

**Le Code construit le payload complet** de chaque contenu ; le node HTTP se contente de le POSTer par item — découpage clair, testable indépendamment.

**Idempotence par relation** : vérifier l'existence d'un contenu lié à l'entité avant de créer. Si des contenus existent déjà, on ne recrée pas.

**Draft-only** : statut brouillon partout ; validation et publication sont des actions humaines.

**Copie dans le corps** de la page si la base n'a pas de champ texte long. Ajouter une propriété **date** pour porter le calendrier.

## Pièges & fixes

| Piège | Fix |
|-------|-----|
| Objet littéral `{ key: value }` dans une expression de jsonBody → erreur de parsing | JSON littéral dans le body + `{{ champ }}` dans les valeurs uniquement |
| Base sans champ date ni champ copie | Ajouter la propriété date ; écrire la copie dans le corps de la page |
| Réponse d'agent bruitée (fences markdown, texte autour du JSON) | Parseur robuste : retrait des fences, isolation `{`…`}`, erreur explicite si pas de JSON valide |
| Entités homonymes en doublon | Sélection **By ID** (l'ID canonique = celui utilisé par les workflows en prod) |
| Code non trivial à passer par API | Encoder en base64, décoder en UTF-8 côté réception → évite les pièges d'échappement |

## Leçons apprises

- Pas d'objet littéral dans une expression de `jsonBody` — JSON en clair + interpolation des valeurs.
- **Lire le schéma réel** avant d'écrire ; ajouter les propriétés manquantes nécessaires au process.
- Faire **construire le payload par un Code**, garder le node HTTP « bête » → lisible, testable, réutilisable.
- L'idempotence par relation est le garde-fou d'un générateur relancé à la demande.

## Voir aussi

- [[sop/sop-outreach-backfill]] — intégrer l'historique manuel dans un pipeline automatisé.
- [[sop/sop-cablage-orchestrateur-subagents]] — câbler l'agent rédacteur comme sous-workflow.
- [[concept-memoire-vivante-agents]] — écriture mémoire après génération (`ExecuteWorkflow Mémoire`).
- [[synthese-lumina-ai-os]] — contexte LUMINA OS, agents hub-and-spoke.
