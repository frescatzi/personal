---
type: raw
title: "POS-GENERIQUE_Repondeur-Email-Drafts-Agent_20260704"
source_url: "drive:1kk9S1zTY-strO-Km5tVVR0X9nQEyWs7V"
captured: 2026-07-07
vault: ai-automation
brand: null
immutable: true
---

# POS GÉNÉRIQUE — Répondeur email à brouillons via agent (pattern draft-only)

Patron réutilisable pour toute marque/projet géré via LUMINA OS : une boîte email surveillée, un agent qui rédige des brouillons, un humain qui garde la main sur l'envoi. Vérifié le 04.07.2026. Version spécifique : `POS-AFTRSN_Repondeur-Email-Drafts-Agent_20260704.md`.

## 1. Principe et gouvernance

L'envoi d'un email est une **action critique** : elle reste toujours gatée par l'humain. Le workflow ne fait que : lire → rédiger un brouillon → déposer le brouillon dans le fil → notifier. L'humain relit et envoie depuis son client email. Aucune exception, quel que soit le niveau de confiance dans l'agent.

## 2. Chaîne type (10 nodes)

1. **Schedule** (fréquence adaptée au volume, l'horaire est un bon défaut).
2. **Lecture inbox** : non-lus récents (`is:unread newer_than:2d`), exclusions dans la requête (`-from:no-reply -category:promotions/social/updates`), limite (ex. 20).
3. **Filtre** (Code) : exclusions par expéditeur (regex `no-?reply|newsletter|notification|mailer-daemon`), extraction expéditeur/objet/corps, **rejet des corps vides** (on ne répond pas à rien).
4. **Agent rédacteur** (Execute Workflow vers l'agent de la marque) avec **sortie balisée** en un seul appel : `RESUME_EMAIL:` (1-2 phrases), `RESUME_DRAFT:` (1-2 phrases), `---DRAFT---` (corps complet). Règles du prompt : langue de l'email reçu, ton de la marque, contexte auto-détecté, questions plutôt qu'inventions, **jamais d'engagement ferme/dépense/contrat**, signature de la marque.
5. **Découpage** (Code) : parse les 3 parties, repli sur extraits tronqués si les balises manquent.
6. **Création du brouillon** en réponse dans le fil (threadId + destinataire = expéditeur).
7. **Idempotence** : markAsRead (simple) ou label dédié (plus robuste si l'humain lit sa boîte en parallèle).
8. **Notification par email traité** (Telegram/Slack…), format humain sans jargon : titre court, expéditeur, objet, résumé du reçu, résumé du draft.
9. **Journalisation mémoire** (épisode par run utile).
10. **Résumé** `{status, drafts}`.

Comportement à vide : 0 email → arrêt silencieux après la lecture (pas de notification vide, pas d'épisode) — c'est le comportement souhaitable pour un run fréquent.

## 3. Difficultés rencontrées (session de référence)

1. `403 Forbidden` sur l'API email malgré une credential valide.
2. Parseur MIME écrit contre un format supposé — 100 % des emails filtrés à tort.
3. Email de test sans corps → rejeté, diagnostic initialement orienté vers le mauvais suspect (le filtre).
4. Correctif API écrasé par l'autosave d'un éditeur resté ouvert.
5. Identifiant de chat de notification introuvable (les API de bots n'acceptent pas les alias en privé).

## 4. Solutions implémentées

1. Activation de l'API dans la console cloud du fournisseur (prérequis d'infrastructure, pas un bug de workflow).
2. Inspection de la sortie réelle d'une exécution, puis réécriture du parseur contre le format observé.
3. Diagnostic par les données (longueur du corps) avant de toucher au code ; test refait avec un contenu réaliste.
4. Rechargement systématique de l'éditeur après toute écriture API.
5. Identifiant obtenu en demandant à l'utilisateur d'écrire à un bot d'introspection (ex. @rawdatabot sur Telegram) qui renvoie son propre id.

## 5. Leçons apprises

1. **Prérequis d'infrastructure d'abord** : une API désactivée côté fournisseur mime un bug de workflow — vérifier l'activation avant de déboguer.
2. **Coder contre le format observé, jamais supposé** : une exécution d'inspection coûte 30 secondes, un parseur faux coûte une session.
3. **Un seul appel d'agent, sortie balisée** : résumés + brouillon en une passe, découpés ensuite — moins cher, plus cohérent, parsable avec repli.
4. **Le format des notifications est une exigence produit** : nom métier du processus, pas de codes techniques, contenu utile à la décision (qui, quoi, résumé reçu, résumé proposé).
5. **Tester le chemin vide et le chemin plein** : boîte vide, email vide, email réel — trois comportements distincts à valider.
