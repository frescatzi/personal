---
type: raw
title: "POS-GENERIQUE_Clonage-roster-agents-multimarques"
source_url: "drive:1TNnkDdySG-G94puj3SRMAoahrl82FR4d"
captured: 2026-07-07
vault: ai-automation
brand: null
immutable: true
---

# LUMINA — POS GÉNÉRIQUE — Clonage d'un roster d'agents vers une nouvelle marque

Patron réutilisable pour dupliquer une équipe d'agents (1 hub + N sous-agents) vers une nouvelle marque/tenant, sans ré-écrire les workflows.

## 1. Distinguer spécifique-marque vs partagé

Avant de cloner, classer les workflows du roster :

- **Spécifiques marque** (à cloner) : les agents dont la mémoire/voix dépend de la marque (Knowledge scellé sur une banque, prompt avec identité de marque).
- **Partagés / infra** (à NE PAS cloner) : routeur LLM, écrivains mémoire, exécuteurs — ils dérivent la marque de leur input et servent toutes les marques. Les cloner = doublons + routage cassé.

## 2. Trois transforms par clone

1. **Nouveaux identifiants de nœuds** (copie profonde). Les connexions étant par **nom** de nœud, l'id peut changer sans casser le câblage.
2. **Rebind de la source de données** : réécrire le paramètre marque du nœud d'accès mémoire (`"brand": "<ancien>"` → `"<code>"`). C'est ce qui isole la mémoire du clone.
3. **Adaptation du prompt** : préfixer un bloc *contexte marque* (nom, voix/ton, mots interdits, tirés d'un `brand_profiles`) + remplacer les mentions de l'ancienne marque.

## 3. Remap du hub

Cloner le hub, puis pour chaque nœud « appel de sous-workflow » pointant un sous-agent cloné → remplacer l'id cible par l'id du clone (et le libellé caché). Laisser intacts les appels vers les workflows partagés.

## 4. Sécurité & idempotence

- **Dry-run d'abord** : construire les graphes clonés en mémoire et vérifier (rebind marque OK, remaps attendus) **sans** créer. Ne passer en création réelle qu'après.
- **Activation** = credentials valides sur tous les nœuds (hérités de la source).
- **Suppression d'un clone de test = action destructive** → laissée à l'humain, pas à l'assistant.
- Skills **génériques** hérités d'emblée ; skills **spécifiques** s'accumulent dans la banque de la marque et remontent via une recherche `générique + marque active`.

## 5. Tester

Ouvrir le hub cloné → requête métier → vérifier : délégation vers les sous-agents clonés, réponse à la bonne voix, mémoire scellée sur la banque de la nouvelle marque.

---
*POS GÉNÉRIQUE — 2026-07-02. Version exacte : POS-AFTRSN clonage roster (script `cloneRoster`). Brique du LUMINA-PLAYBOOK v2.*
