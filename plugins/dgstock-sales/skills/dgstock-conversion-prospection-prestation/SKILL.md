---
name: dgstock-conversion-prospection-prestation
description: Mesure le taux et le délai de transformation d'une prospection (pipeline DGStock) en prestation confirmée (vente). Utilise OBLIGATOIREMENT ce skill quand l'utilisateur demande "quel est notre taux de conversion", "combien de prospects deviennent clients", "où perd-on le plus de prospects dans le pipeline", "le funnel de vente", ou toute variante sur la transformation pipeline→vente DGStock. Déclenche aussi sur "/dgstock-conversion-prospection-prestation".
---

# Conversion prospection → prestation (funnel DGStock)

## Limite structurelle à annoncer à l'utilisateur avant de commencer

**Il n'existe aujourd'hui aucun champ technique reliant directement une `prospection` à la `prestation` qu'elle a éventuellement générée** (pas de `prospection_id` sur `prestations`). Toute mesure de conversion ici est donc une **estimation par recoupement** (nom du client, montant, proximité de dates), pas une mesure exacte. Dis-le explicitement dans le rapport — ne présente jamais un taux de conversion comme une vérité absolue. Si l'utilisateur veut une mesure fiable, la vraie solution est de demander à DGStock d'ajouter ce lien en base (à mentionner une fois, pas besoin d'insister à chaque rapport).

## Pièges techniques déjà rencontrés sur ce MCP

- **Les filtres `=` sur des valeurs accentuées peuvent échouer silencieusement** (ex. `etat = "Intéressé"` renvoie 0 résultat alors que des lignes existent avec cette valeur — probablement un problème d'encodage/normalisation Unicode). Utilise l'opérateur `like` avec une sous-chaîne sans accent (ex. `"ress"`) pour les états sensibles, puis **filtre toi-même côté résultat** sur le texte exact, car `like` "ress" remonte aussi bien `Intéressé` que `PC-pas interssé`/`SAD-Pas Intéressé` (qui contiennent aussi cette sous-chaîne). Ne te fie jamais à un seul filtre serveur pour les états positifs vs négatifs — vérifie le champ `etat` réellement renvoyé pour chaque ligne avant de classer.
- L'opérateur `in` ne fonctionne pas sur `etat` — fais une requête par valeur.

## Comment exécuter ce skill

1. **Vue d'ensemble du pipeline** : compte les prospections par grande famille d'état (utilise `aggregate: {fn: "count", group_by: "etat"}` borné par une fenêtre de dates raisonnable pour éviter un timeout sur tout l'historique) :
   - États d'avancement positif : `Encours de livraison`, et les lignes dont l'`etat` exact (vérifié ligne à ligne) est `Intéressé` (pas `*pas interssé`/`*Pas Intéressé`).
   - États de premier contact (`PC-*`) et de suivi après démo (`SAD-*`) — voir skill `dgstock-suivi-prospections` pour le détail des codes.
   - États terminaux négatifs (`*pas interssé`, `Faux numero`).

2. **Volume de ventes confirmées** : compte les `prestations` avec `etat = "Vente confirmee"` (utilise `like` si besoin pour les variantes), et regarde aussi combien de prestations n'ont **aucun** état renseigné (`etat` null ou vide) — c'est une anomalie de saisie à signaler, pas un état métier.

3. **Estimer le lien pipeline → vente** : prends l'échantillon de prestations confirmées récentes (avec `relations: ["client"]`) et recherche, pour chaque client, s'il existe une prospection au nom similaire. C'est un travail d'appariement approximatif — fais-le sur un échantillon (10-20 prestations récentes) plutôt que sur l'intégralité, et présente-le comme illustratif ("sur cet échantillon, X/Y ventes confirmées correspondent à un client qu'on retrouve aussi dans le pipeline prospections").

4. **Mettre en garde sur le biais des ventes directes** : une partie des `prestations` correspond probablement à des clients jamais passés par le pipeline `prospections` (vente directe, client de passage — le nom générique `"passager"` vu dans les données en est un indice). Ne calcule jamais un "taux de conversion" en divisant simplement le nombre de prestations par le nombre de prospections : ce ratio mélangerait deux populations différentes et serait trompeur. Présente plutôt : (a) le volume à chaque étape du pipeline, (b) le volume de ventes confirmées, (c) une estimation prudente du lien entre les deux, avec la réserve du point 1.

5. **Produire le rapport** :

```markdown
# Funnel prospection → prestation — [date ou période]

## Répartition du pipeline (prospections)
| Étape | Nombre | Part du total |
|---|---|---|
| Premier contact (PC-*) | X | X% |
| Suivi après démo (SAD-*) | X | X% |
| Intéressé / Encours de livraison | X | X% |
| États négatifs/terminaux | X | X% |

## Ventes
- Prestations confirmées (toute période demandée) : X
- Prestations sans état renseigné (anomalie) : X

## Estimation du lien pipeline → vente (échantillon, approximatif)
- Sur un échantillon de X ventes confirmées récentes, Y semblent correspondre à un client déjà suivi dans le pipeline prospections.
- ⚠️ Cette estimation est approximative : il n'existe pas de lien technique direct entre les deux entités côté CRM. Pour un taux de conversion fiable, il faudrait demander l'ajout d'un identifiant de prospection sur la prestation.

## Recommandation
[ce qui ressort le plus clairement du goulot d'étranglement observé, ex. "le pipeline perd l'essentiel de son volume entre Premier Contact et Suivi Après Démonstration — c'est là qu'investiguer en priorité"]
```

6. Propose un export PDF si l'utilisateur le souhaite.
