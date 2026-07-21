---
type: raw
title: "POS-GENERIQUE_accueil-et-glossaire-espace-connaissances_2026-07-17"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-generique_accueil-et-glossaire-espace-connaissances_2026-07-17.md"
captured: 2026-07-21
vault: personal
brand: null
immutable: true
---

# POS-GÉNÉRIQUE : doter un espace de connaissances d'une page d'accueil et d'un glossaire bilingues

Date : 17.07.2026 · Portée : générique, tout espace (Notion ou autre) qui héberge une base de connaissances et manque d'une porte d'entrée expliquant le système et sa nomenclature. Aucune référence à des IDs spécifiques.

## Principe

Un espace de savoir gagne une porte d'entrée : une page d'accueil qui dit ce qu'est le système, comment la connaissance est rangée, et ce que veut dire chaque sigle de la nomenclature documentaire, plus un glossaire des termes techniques. On construit de façon additive (rien n'est supprimé), on relie par des liens stables (par ID), et on garde la sidebar propre.

## Marche à suivre

1. Lire la vérité terrain d'abord : l'emplacement réel de la base (chemin d'ancêtres), son schéma, ses vues. Ne pas se fier à la doc seule.
2. Rédiger l'accueil en langage simple : ce qu'est le système (sans le réduire à un seul de ses usages ou clients), comment la connaissance est organisée (lien vers la base), la nomenclature des documents avec le nom complet à côté de chaque sigle, et un lien vers le glossaire.
3. Bilingue si demandé : deux moitiés claires (une par langue) plutôt qu'un mélange, pour la lisibilité.
4. Placement : si l'API ne peut pas créer à la racine de l'espace, créer sous une page adressable puis laisser le propriétaire glisser la page à la racine.
5. Glossaire : une entrée par terme, nom complet plus définition d'une ligne, en langage simple. Lier depuis l'accueil.
6. Portes filtrées : créer les vues (par catégorie, par type) en ONGLETS sur la base, pas incrustées sur l'accueil, pour ne pas encombrer la sidebar.
7. Nettoyer les noms : retirer les caractères parasites d'un titre (par exemple un tiret long) par API ; un renommage est stable par ID, sans impact sur les automations.
8. Montrer, faire valider, documenter.

## Garde-fous

1. Additif avant destructif ; toute suppression = geste du propriétaire, en dernier, prouvé sûr.
2. Renommer une option de liste : préférer l'UI (préserve l'identité et les valeurs) à l'API (réécrit la liste).
3. Ne pas réduire la définition d'une plateforme à un seul de ses clients ou marques.
4. Loi sidebar : une vue visible sur une page apparaît dans la sidebar sous cette page ; préférer les onglets sur la base.

## Difficultés rencontrées

1. Racine d'espace non adressable par l'API (création et déplacement).
2. Automation conditionnelle native souvent payante.
3. Doc antérieure en décalage avec la réalité (objet déclaré supprimé alors qu'il contenait encore du vivant).

## Solutions implémentées

1. Créer sous une page adressable puis drag propriétaire à la racine.
2. Valeur par défaut de template comme alternative gratuite au remplissage automatique.
3. Re-fetch systématique de la vérité terrain avant d'agir.

## Lessons learned

1. Une page d'accueil et un glossaire transforment un dépôt de fichiers en un espace navigable.
2. Les liens par ID survivent aux renommages et aux déplacements.
3. Onglets sur la base plutôt que vues incrustées, pour une sidebar propre.
4. Toujours donner le nom complet à côté d'un sigle : la nomenclature ne se devine pas.
