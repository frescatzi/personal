---
type: raw
title: "POS-GENERIQUE_Capture-Connaissance-Debrief_20260706"
source_url: "drive:1Hv2gPdeTO64h-mjFz3zJw2kVpF9NX4jE"
captured: 2026-07-07
vault: personal
brand: null
immutable: true
---

# POS-GÉNÉRIQUE — Capture de connaissance post-projet (débrief → mémoire)

**Portée :** réutilisable pour toute marque gérée via LUMINA OS. Débarrassée des IDs AFTRSN. Paire spécifique : `POS-AFTRSN_Capture-Connaissance-Debrief_20260706.md`.

---

## 1 · But

Transformer chaque unité de travail terminée (édition, projet, campagne, sprint) en connaissance réutilisable et traçable, stockée dans la banque mémoire de la marque, collection `insights`, avec validation humaine.

## 2 · Modèle deux temps (Schedule périodique)

**Phase A — Ouvrir (auto sur statut « terminé »).** Les entités passées au statut terminal sans fiche débrief → un agent rédacteur pré-remplit un résumé factuel depuis les données déjà connues → création d'une **fiche débrief dédiée** (base/collection dédiée), reliée à l'entité, statut `À remplir`, avec un **canevas qualitatif** (ce qui a marché / raté / chiffres réels / à refaire / à éviter / idées). Notification. Idempotence = existence de fiche.

**Phase B — Capturer (sur statut « prêt »).** L'humain complète la fiche et la passe en `Prêt` → lecture du corps → un agent extrait N insights structurés (JSON strict, autonomes, actionnables, tagués par thème) → écriture de chaque insight dans la mémoire → fiche `Capturé` + horodatage + notification + épisode. Idempotence = transition `Prêt`→`Capturé`.

## 3 · Traçabilité (invariant)

Chaque insight : collection dédiée, `knowledge_type`, `source`, `source_ref` (= identifiant de l'entité), `content_hash` pour dédup, embedding figé. Le lien entité↔mémoire passe par l'identifiant stable.

## 4 · Règles de robustesse

- **Gate humain via un statut** dans l'outil de gestion (pas de capture avant validation).
- **Pré-remplissage IA factuel + canevas fixe déterministe** (l'IA ne fabrique pas la structure).
- **Idempotence double garde** (existence de fiche en A, transition de statut en B) → le poll périodique ne duplique jamais.
- **Écriture mémoire via le sous-workflow d'écriture dédié** (jamais d'insertion SQL directe depuis le workflow métier).
- **Parser d'insights défensif** : retirer les fences, isoler le tableau JSON, tolérer 0 insight (marquer quand même `Capturé` pour ne pas retraiter).

## Difficultés rencontrées
- Ambiguïté « déclenchement auto » + « fiche dédiée » → un modèle deux temps la lève.
- Dépendance à une infra mémoire partagée (une panne de credential bloque l'écriture).

## Solutions implémentées
- Deux branches parallèles sur un déclencheur planifié ; statut de fiche comme pilote et comme gate.
- Traiter les pannes d'infra partagée dans un runbook séparé.

## Leçons apprises
- Le meilleur endroit pour un gate humain récurrent est un **champ statut** visible dans l'outil que l'humain utilise déjà.
- Séparer nettement **rédaction IA** (résumé, insights) et **structure déterministe** (canevas, blocs) donne fiabilité + intelligence.
- Avant de corriger un workflow qui échoue en écriture mémoire, **vérifier l'infra partagée** (credential, base).

---
*POS générique rédigée le 06.07.2026.*
