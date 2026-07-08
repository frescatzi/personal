# Récapitulatif de session — Connecter Claude, ChatGPT et Gemini dans n8n

*Session du 18 juin 2026 · Projet : AFTER SUN PEOPLE*

---

## Objectif de départ

Connecter les plateformes IA (Claude, ChatGPT, Gemini) à n8n, les faire **dialoguer entre elles**, et faire tourner le tout **en autonomie** sur le serveur, sans avoir besoin d'être devant l'ordinateur.

---

## Décisions clés

- **Claude Chat / Cowork / Code ne sont pas 3 connexions** : ce sont des interfaces sur le même moteur. Dans n8n, on connecte **une seule clé API Anthropic**.
- **Abonnement ≠ API** : il faut une clé API dédiée par plateforme, facturée à l'usage, séparée de l'abonnement.
- **Compte API Anthropic** : choisi en mode **Individual**. Crédit chargé : $20.
- **Hébergement** : n8n auto-hébergé sur **Hetzner** via **Coolify** → autonomie 24/7 (serveur allumé en permanence).

---

## Infrastructure (déjà en place)

| Élément | Détail |
|---|---|
| Serveur | Hetzner (serveur nu) |
| Déploiement | Coolify (dashboard port 8000, HTTPS via Traefik + Let's Encrypt) |
| n8n | version 2.10.4 |
| URL | https://n8n.aftersunpeople.com |
| Domaine | géré chez GoDaddy (enregistrement DNS A → IP Hetzner) |

> Principe retenu : *« Hetzner fournit le serveur nu ; Coolify s'installe ensuite par SSH »* et *« le DNS dit où aller, le HTTPS est généré sur le serveur par Coolify/Traefik »*.

---

## Le problème rencontré (et résolu)

**Erreur dans n8n** sur la credential Anthropic :
> « Couldn't connect with these settings — The resource you are requesting could not be found »

**Cause** : bug connu de n8n. Son test de credential appelle en dur un vieux modèle retiré, `claude-3-haiku-20240307`, qui n'existe plus → erreur 404. **La clé API était valide** (crédit présent, authentification OK).

**Vérification faite** : appel manuel via HTTP Request avec `claude-3-haiku-20240307` → 404 « model not found » ; puis avec `claude-sonnet-4-6` → réponse texte normale. Preuve que seul le modèle était en cause.

**Contournement adopté** : utiliser un nœud **HTTP Request** pour Claude au lieu du nœud natif.

```
Method  : POST
URL     : https://api.anthropic.com/v1/messages
Headers : x-api-key = <clé sk-ant-…>
          anthropic-version = 2023-06-01
          content-type = application/json
Body (JSON) :
{
  "model": "claude-sonnet-4-6",
  "max_tokens": 1024,
  "messages": [{ "role": "user", "content": "…" }]
}
```
Réponse de Claude → `{{ $json.content[0].text }}`

> Les nœuds natifs **OpenAI** et **Google Gemini** ne sont pas touchés par ce bug.

---

## Le workflow construit (fonctionnel ✅)

```
[Execute workflow] → [HTTP Request : Claude rédige] → [ChatGPT : critique]
→ [Gemini : synthèse finale] → [Edit Fields : légende propre]
```

Cas d'usage : générer une **légende Instagram** pour un soin après-soleil — Claude rédige, ChatGPT critique, Gemini réécrit la version finale.

### Références entre nœuds (le « passage de relais »)

| Sortie de… | Expression pour récupérer le texte |
|---|---|
| Claude (HTTP Request) | `{{ $json.content[0].text }}` |
| ChatGPT (Message a Model) | `{{ $json.content[0].text }}` |
| Gemini | à récupérer via glisser-déposer depuis le panneau d'input |

**Réflexe anti-`[undefined]`** : un `[undefined]` = soit le nœud précédent n'a pas encore tourné, soit le nom du champ est faux. Solution : exécuter le nœud précédent, ouvrir sa sortie **JSON**, suivre le chemin jusqu'au texte, ou glisser-déposer le champ depuis le panneau de gauche (n8n écrit l'expression tout seul).

---

## Coûts (ordre de grandeur)

- n8n auto-hébergé : gratuit (logiciel)
- Serveur Hetzner : ~5-10 €/mois
- APIs IA : à l'usage (quelques € / mois en usage léger) — penser à fixer des plafonds de dépense côté OpenAI et Anthropic

---

## Prochaines étapes

1. **Sauvegarder** le workflow dans n8n (lui donner un nom + Save).
2. **Finaliser Edit Fields** : champ `legende_finale` = texte de Gemini, pour un résultat propre et réutilisable.
3. **Autonomie** : remplacer le déclencheur manuel par un **Schedule Trigger** (ex. lundi 9h) puis activer l'interrupteur **Active**.
4. **Diffusion** : ajouter un nœud final **Gmail** / **Notion** / **Google Sheets** pour recevoir ou archiver chaque légende automatiquement.
5. **Phase 2 (avancé)** : explorer le nœud **AI Agent** pour un agent « chef » qui appelle les autres comme outils.
