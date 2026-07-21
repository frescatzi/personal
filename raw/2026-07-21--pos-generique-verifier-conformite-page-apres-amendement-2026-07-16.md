---
type: raw
title: "POS-GENERIQUE_verifier-conformite-page-apres-amendement_2026-07-16"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-generique_verifier-conformite-page-apres-amendement_2026-07-16.md"
captured: 2026-07-21
vault: personal
brand: null
immutable: true
---

# POS-GÉNÉRIQUE — Clore un milestone absorbé par un amendement (vérification de conformité plutôt que re-build)

Date : 16.07.2026 · Portée : tout projet piloté par spec à milestones (Notion ou autre)

## Contexte d'usage

En cours de build, le commanditaire amende la cible (ex. « finalement, le dashboard reste sur la page d'accueil »). L'exécution de l'amendement construit de fait l'état visé par un milestone ultérieur. Comment clore proprement ce milestone sans le re-faire ni le cocher « gratuitement » ?

## Marche à suivre

1. **Au moment de l'amendement** : exécuter la nouvelle cible immédiatement, la faire valider, puis amender les documents de référence (concept + spec) — le milestone ultérieur est requalifié « allégé » dans la spec, avec mention explicite de l'amendement.

2. **Au moment du milestone allégé** : ne rien reconstruire. Dérouler une **checklist de conformité** :
   - vérification structurelle par API (ordre et présence des blocs/sections attendus) ;
   - vérification visuelle (rendu réel, navigateur) ;
   - chaque item de la checklist coché explicitement.

3. **Proposer le polish résiduel** au commanditaire (cosmétique : textes, icônes, espacement) — c'est lui qui tranche « tel quel » ou « retouche ».

4. **Documenter quand même** (POS allégé) : objectif, checklist, décision, et le renvoi vers le POS de l'amendement qui a fait le vrai travail. Cocher le milestone uniquement après validation explicite.

## Difficultés rencontrées

- Tentation de cocher le milestone « gratuitement » (le travail semble déjà fait) sans vérification formelle.

## Solutions implémentées

- Checklist de conformité en double canal (structure par API + rendu par navigateur) + validation explicite du commanditaire, avec option polish.

## Lessons learned

- **Exécuter un amendement tout de suite** (plutôt que le noter pour plus tard) transforme un futur milestone de démolition en simple vérification.
- **Vérifier ≠ reconstruire** : un milestone absorbé se clôt par une checklist, pas par du travail redondant — mais il se clôt FORMELLEMENT (validation + doc), sinon la spec ment.
- La trace documentaire (« ce milestone a été absorbé par l'amendement X, voir POS Y ») garde l'historique lisible pour les sessions suivantes.
