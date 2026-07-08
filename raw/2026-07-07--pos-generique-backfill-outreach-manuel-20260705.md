---
type: raw
title: "POS-GENERIQUE_Backfill-Outreach-Manuel_20260705"
source_url: "drive:1xV0_sFH99lH02FaLF0hWZa0QsSNTMMhs"
captured: 2026-07-07
vault: ai-automation
brand: null
immutable: true
---

# POS GÉNÉRIQUE — Backfill : intégrer des contacts déjà démarchés manuellement dans un processus d'outreach automatisé

Pattern vérifié le 05.07.2026 (première application : lieux événementiels). Réutilisable pour toute marque gérée via LUMINA OS dont l'outreach a commencé manuellement avant l'automatisation (lieux, artistes, partenaires, sponsors, prestataires). Version spécifique : `POS-AFTRSN_Backfill-Outreach-Manuel_20260705.md`.

## 1. Principe

Un processus d'outreach piloté par une base de données (voir `POS-GENERIQUE_Outreach-Relances-Base-Drafts_20260704.md`) peut **absorber l'historique manuel** sans rien envoyer à nouveau : il suffit de reconstituer, pour chaque cible déjà contactée, les trois données que le processus aurait écrites lui-même — l'adresse réelle, la **date du premier envoi** et l'**identifiant du fil** de conversation. Une fois posées dans la base avec le statut déclencheur, les relances reprennent automatiquement dans le fil d'origine, comme si le processus avait toujours été là.

## 2. Procédure

1. **Collecter les noms** des cibles déjà contactées (attention : noms d'usage ≠ noms réels).
2. **Fouiller la boîte d'envoi** du compte utilisé : recherche `in:sent` + mots-clés/variantes de graphie ; en dernier recours, lister les N derniers envoyés et filtrer à l'œil. Extraire par cible : destinataire, date du premier envoi, identifiant du fil, sujet.
3. **Consigner dans la base de pilotage** : adresse, date de premier contact, identifiant de fil, note d'historique (« contacté manuellement le X, sans réponse »), puis le **statut déclencheur**.
4. **Laisser le processus reprendre** : il voit un premier contact ancien sans relance → prépare la relance 1 en brouillon dans le fil ; puis la relance 2 à intervalle de la relance 1.

## 3. Difficultés rencontrées

1. Recherche par nom peu fiable : 2 cibles sur 3 introuvables par mots-clés (nom réel différent du nom d'usage ; contact passé par une agence tierce dont le domaine ne ressemble pas au nom du lieu).
2. Graphies collées non matchées par la recherche de la messagerie (« Studio45 » ≠ « studio 45 »).
3. Données anciennes × conditions temporelles neuves : les intervalles calculés sur la date d'origine (J+3/J+7 depuis le premier email) déclenchaient deux relances à un jour d'intervalle sur les fiches backfillées.

## 4. Solutions implémentées

1. Repli systématique : lister les derniers envoyés sans filtre et identifier à l'œil ; faire confirmer les adresses par l'humain qui a envoyé.
2. Requêtes multi-variantes avant le repli (exact, collé, séparé, domaine du site).
3. Correctif de cadence : chaque relance se déclenche par rapport à **la relance précédente** (ex. relance 2 = relance 1 + 4 jours), jamais par rapport à l'origine de la séquence.

## 5. Leçons apprises

1. La **boîte d'envoi est la source de vérité** de l'historique d'outreach — pas la mémoire humaine ni les noms d'usage.
2. Un backfill est un test de bord gratuit : il exerce des états que le chemin nominal ne produit jamais et révèle les défauts de conditions temporelles. Relire les conditions de cadence avant d'injecter des données anciennes.
3. Les intervalles d'une séquence se calculent sur l'événement précédent de la séquence — règle générale.
4. Toujours récupérer l'identifiant de fil : relancer dans la conversation d'origine préserve le contexte côté destinataire et augmente les chances de réponse.
