---
type: raw
title: "concept-prompt-caching"
source_url: "drive:1T2vqiKtishb2ocNhmsZK27Wug5ThHYQZ"
captured: 2026-07-07
vault: ai-automation
brand: null
immutable: true
---

---
type: wiki
title: Prompt caching — impact sur le débit effectif (API Claude)
status: active
publish: none
vault: ai-automation
brand:
sources:
  - raw/2026-06-22--claude-api-rate-limits.md
related:
  - wiki/concept-limites-api-claude.md
  - wiki/concept-gestion-erreurs-429.md
updated: 2026-06-29
---

# Prompt caching — impact sur le débit effectif

## Idée centrale

Le cache de prompt côté API Claude ne sert pas qu'à réduire les coûts (lecture cache facturée à 10 % du prix de base) : il **désengorge aussi la limite de débit ITPM**, puisque la quasi-totalité des tokens lus depuis le cache ne comptent pas dans le quota par minute.

## Points utiles

- Décomposition : `total_input_tokens = cache_read_input_tokens + cache_creation_input_tokens + input_tokens`.
- Seuls `input_tokens` (après le dernier point de rupture de cache) et `cache_creation_input_tokens` comptent dans l'**ITPM** — `cache_read_input_tokens` n'y compte **pas** (exception : Haiku 3.5).
- Effet multiplicateur concret : avec une limite ITPM de 2 M tokens et un taux de cache de 80 %, on peut traiter ~10 M tokens d'entrée/minute (2 M hors cache + 8 M servis depuis le cache).
- Cibles naturelles pour la mise en cache : instructions système, gros documents de contexte, définitions d'outils, historique de conversation — tout ce qui est répété d'appel en appel sans changer.
- Voir [[concept-limites-api-claude]] pour le reste du système de quotas (RPM/OTPM, niveaux, 429).

## Limites / conditions d'usage

- L'avantage ITPM ne s'applique pas à Haiku 3.5 : les `cache_read_input_tokens` y comptent dans le calcul.
- L'effet dépend directement du taux de cache réel obtenu (proportion de tokens servis depuis le cache vs recréés) — pas un gain garanti, à mesurer par usage.


<!-- test déclencheur croisé 2026-06-24 -->
