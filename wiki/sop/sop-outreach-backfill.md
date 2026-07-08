---
type: wiki
title: "SOP — Backfill : intégrer des contacts déjà démarchés manuellement dans un outreach automatisé"
status: active
publish: notion
vault: ai-automation
brand:
sources:
  - raw/2026-07-06--pos-generique-backfill-outreach-manuel-20260705.md
related:
  - wiki/sop/sop-calendrier-contenu-agent.md
  - wiki/synthese-lumina-ai-os.md
updated: 2026-07-06
---

# SOP — Backfill : intégrer des contacts déjà démarchés manuellement dans un outreach automatisé

## Principe

Un processus d'outreach piloté par base de données peut **absorber l'historique manuel sans rien envoyer à nouveau** : il suffit de reconstituer, pour chaque cible déjà contactée, les données que le processus aurait écrites lui-même — adresse réelle, **date du premier envoi**, **identifiant du fil** de conversation. Une fois posées dans la base avec le statut déclencheur, les relances reprennent automatiquement dans le fil d'origine, comme si le processus avait toujours été là.

S'applique à toute marque LUMINA dont l'outreach a commencé manuellement avant l'automatisation (lieux, artistes, partenaires, sponsors, prestataires).

## Procédure

1. **Collecter les noms** des cibles déjà contactées — attention : noms d'usage ≠ noms réels.
2. **Fouiller la boîte d'envoi** : recherche `in:sent` + mots-clés/variantes de graphie ; si introuvable, lister les N derniers envoyés et filtrer à l'œil. Extraire par cible : destinataire, date du premier envoi, identifiant de fil, sujet.
3. **Consigner dans la base** : adresse, date de premier contact, identifiant de fil, note d'historique (ex. « contacté manuellement le X, sans réponse »), puis le **statut déclencheur**.
4. **Laisser le processus reprendre** : il voit un premier contact ancien sans relance → prépare la relance 1 en brouillon dans le fil → puis relance 2 à intervalle de relance 1.

## Difficultés connues + fixes

| Difficulté | Solution |
|-----------|----------|
| Nom d'usage ≠ nom réel / passage par une agence tierce → introuvable par mots-clés | Repli : lister les derniers envoyés sans filtre, identifier à l'œil, faire confirmer l'adresse par l'expéditeur humain |
| Graphies collées non matchées par la recherche (« Studio45 » ≠ « studio 45 ») | Requêtes multi-variantes (exact, collé, séparé, domaine du site) avant le repli |
| Données anciennes + conditions temporelles neuves → deux relances envoyées à un jour d'intervalle | **Cadence relative** : chaque relance se calcule depuis la **relance précédente** (relance 2 = relance 1 + 4 jours), jamais depuis l'origine de la séquence |

## Leçons apprises

1. **La boîte d'envoi est la source de vérité** de l'historique — pas la mémoire humaine ni les noms d'usage.
2. Un backfill est un test de bord gratuit : il exerce des états que le chemin nominal ne produit jamais et révèle les défauts de conditions temporelles. **Relire les conditions de cadence avant d'injecter des données anciennes.**
3. Les intervalles d'une séquence se calculent toujours sur l'**événement précédent de la séquence** — règle générale.
4. Toujours récupérer l'**identifiant de fil** : relancer dans la conversation d'origine préserve le contexte côté destinataire et augmente les taux de réponse.

## Voir aussi

- [[sop/sop-calendrier-contenu-agent]] — génération automatisée d'un calendrier de contenu (draft-only).
- [[synthese-lumina-ai-os]] — architecture LUMINA OS, contexte multi-marques.
