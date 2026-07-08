---
type: raw
title: "concept-gestion-erreurs-429"
source_url: "drive:1INGfM-IZeHRf8exIEyXuRssBS3u-hhFe"
captured: 2026-07-07
vault: ai-automation
brand: null
immutable: true
---

---
type: wiki
title: "Gestion des erreurs 429 — API Claude (et APIs REST)"
status: draft
publish: none
vault: ai-automation
brand: null
sources: []
related:
  - wiki/concept-limites-api-claude.md
  - wiki/concept-prompt-caching.md
updated: 2026-06-29
---

# Gestion des erreurs 429 — API Claude (et APIs REST)

> Page stub — à enrichir à partir d'une source raw/ dédiée.

## Idée centrale

Une erreur **HTTP 429 (Too Many Requests)** signale que la limite de débit a été atteinte. L'API Claude retourne un en-tête `retry-after` indiquant le nombre de secondes à attendre avant de réessayer. La stratégie correcte est un **backoff exponentiel avec jitter** — ne jamais marteler en boucle.

## Points essentiels

- Lire l'en-tête **`retry-after`** (secondes) et attendre ce délai avant tout retry.
- Les en-têtes **`anthropic-ratelimit-*`** exposent limite / restant / reset pour RPM, ITPM et OTPM — utiles pour un throttling proactif.
- Le **seau à jetons** (token bucket) se réapprovisionne en continu : une rafale courte peut déclencher un 429 même si le débit moyen reste sous la limite. Espacer les requêtes plutôt que les envoyer toutes d'un coup.
- **Levier principal pour éviter les 429 sur l'ITPM** : activer le [[concept-prompt-caching]] (les tokens lus depuis le cache ne comptent pas dans l'ITPM).

## À documenter (trou actuel)

- Schéma de code backoff exponentiel + jitter (Python / JavaScript).
- Gestion dans un contexte n8n (nœud "Wait" ou gestion d'erreur sur le nœud HTTP).
- Seuils de retry raisonnables (max tentatives, délai max).

## Voir aussi

- [[concept-limites-api-claude]] — système complet de quotas RPM/ITPM/OTPM, niveaux, modes.
- [[concept-prompt-caching]] — réduire la pression ITPM via le cache.
