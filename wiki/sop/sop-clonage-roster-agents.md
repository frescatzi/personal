---
type: wiki
title: SOP — Clonage d'un roster d'agents vers une nouvelle marque
status: active
publish: notion
vault: ai-automation
brand:
sources:
  - raw/2026-07-02--pos-generique-clonage-roster-agents-multimarques.md
  - raw/2026-07-06--pos-generique-clonage-roster-agents-multimarques.md
  - raw/2026-07-02--lumina-playbook-v1-2026-07-02.md
related:
  - wiki/synthese-lumina-ai-os.md
  - wiki/concept-memoire-vectorielle-multi-marques.md
  - wiki/sop/sop-audit-edition-n8n-api-interne.md
  - wiki/concept-classification-workflows-n8n.md
  - wiki/sop/SOP_systeme-multi-agents-memoire-centrale-mcp-n8n.md
updated: 2026-07-06
---

# SOP — Clonage d'un roster d'agents vers une nouvelle marque

## Principe

Dupliquer une équipe d'agents (1 hub Maestro + N sous-agents) vers une nouvelle marque **sans réécrire les workflows**. Les agents partagés (routeur LLM, mémoire, exécuteurs) ne sont **jamais** clonés — ils dérivent la marque de leur input.

## 1. Distinguer spécifique-marque vs partagé

**À cloner :** agents dont la mémoire/voix dépend de la marque (Knowledge scellé sur une banque, prompt avec identité de marque).

**À NE PAS cloner :** routeur LLM, writers mémoire, exécuteurs Hermes — ils servent toutes les marques. Les cloner = doublons + routage cassé.

## 2. Les 3 transforms par clone

1. **Nouveaux identifiants de nœuds** (copie profonde). Les connexions étant par **nom** de nœud, l'id peut changer sans casser le câblage.
2. **Rebind de la source de données** : réécrire `"brand": "<ancien>"` → `"<code>"` dans le nœud d'accès mémoire. C'est ce qui isole la mémoire du clone.
3. **Adaptation du prompt** : préfixer un bloc *contexte marque* (nom, voix/ton, mots interdits depuis `brand_profiles`) + remplacer les mentions de l'ancienne marque.

## 3. Remap du hub (Maestro)

Cloner le hub → pour chaque nœud « appel de sous-workflow » pointant un sous-agent cloné → remplacer l'ID cible par l'ID du clone. Laisser intacts les appels vers les workflows partagés (routeur, mémoire).

## 4. Procédure complète

1. **Provisionner** la banque mémoire : `LUMINA-BRAND-PROVISION {code, brand_name, tone_voice, forbidden_words}` → crée `<code>_memory` + index HNSW + ligne `memory_registry` + `brand_profiles`.
2. **Ingérer le canon** : bible de marque (PDF/MD) → pipeline d'ingestion avec `brand=<code>`, `collection='canon'`.
3. **Dry-run du clone** : construire les graphes clonés en mémoire, vérifier (rebind OK, remaps attendus) **sans** créer.
4. **Créer les clones** via l'API interne (voir [[sop/sop-audit-edition-n8n-api-interne]]).
5. **Remap du hub** : mettre à jour les `workflowId` des tools de délégation.
6. **Ranger** dans `<MARQUE>-03-BRAIN/Sub-Agents` (voir [[concept-classification-workflows-n8n]]).
7. **Activer** (credentials valides sur tous les nœuds, héritées de la source).
8. **Vérifier** : `memory-search {brand:<code>}` renvoie le canon ; test de délégation Maestro → sous-agent cloné fonctionne.

## 5. Sécurité & limites

- **Suppression d'un clone de test** = action destructive → laissée à l'humain, pas à l'assistant.
- **Skills génériques** hérités d'emblée via la banque partagée ; skills spécifiques s'accumulent progressivement dans `<code>_memory` via auto-promotion nocturne.
- **Activation** refusée si une credential est manquante → copier depuis le workflow source.

## 6. Tester

Ouvrir le hub cloné → requête métier → vérifier : délégation vers les sous-agents clonés, réponse à la bonne voix de marque, mémoire scellée sur la banque `<code>_memory`.

## Voir aussi

- [[synthese-lumina-ai-os]] — playbook v1 et onboarding multi-marques.
- [[concept-memoire-vectorielle-multi-marques]] — banque par marque, registre, schéma.
- [[sop/sop-audit-edition-n8n-api-interne]] — API interne pour créer/modifier les workflows.
- [[concept-classification-workflows-n8n]] — ranger le roster dans `03-BRAIN/Sub-Agents`.
- [[sop/SOP_systeme-multi-agents-memoire-centrale-mcp-n8n]] — SOP mémoire centrale MCP.
