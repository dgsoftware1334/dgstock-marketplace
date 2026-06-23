---
name: dgstock-suivi-prospections
description: Produit la liste priorisée des prospections (pipeline commercial DGStock/DG-SALE) à recontacter aujourd'hui — qui rappeler, dans quel ordre, et pourquoi. Utilise OBLIGATOIREMENT ce skill dès que l'utilisateur demande "qui dois-je rappeler", "fais le suivi des prospections", "où en est le pipeline", "quels prospects sont en attente", ou toute variante autour du suivi commercial / des rappels / du pipeline de vente DGStock. Déclenche aussi sur "/dgstock-suivi-prospections".
---

# Suivi des prospections (pipeline commercial DGStock)

## Contexte métier

Le pipeline commercial s'appelle "prospections" dans le CRM DGStock (pas "prospects" ni "opportunités"). Chaque prospection traverse des étapes encodées dans le champ `etat` :

| Préfixe | Signification | Exemples |
|---|---|---|
| *(aucun)* | Lead pas encore travaillé | `Premiere Entree`, `deuxieme entree` |
| `PC-` | **P**remier **C**ontact (premier appel/message) | `PC-Rappel`, `PC-pas interssé`, `PC-NRP` (pas de réponse), `PC-INJ` (injoignable), `PC-ligne ocupée` |
| `SAD-` | **S**uivi **A**près **D**émonstration (après une démo faite par appel ou message) | `SAD-Rappel`, `SAD-Pas Intéressé` |
| *(positif)* | Proche de la conversion | `Intéressé`, `Encours de livraison` |
| *(annexe)* | États particuliers | `Faux numero`, `Reclamation version d'essai` |

Un client qui a eu une démo et qui est en `SAD-Rappel` est plus avancé et plus prioritaire qu'un lead en `PC-Rappel` qui n'a eu qu'un premier contact.

## Important : ce skill est en LECTURE SEULE

Le MCP DGStock ne permet aucune écriture (pas de mise à jour de statut, pas d'envoi de message). Ce skill produit une **liste d'action que l'humain exécute lui-même** (appel, WhatsApp). Ne jamais formuler une phrase qui laisse croire qu'un statut a été changé ou qu'un message a été envoyé.

## Comment exécuter ce skill

1. **Localiser les outils DGStock disponibles.** Cherche parmi les outils MCP ceux dont le nom contient `dgstock_query_data`, `dgstock_fetch_prospections` ou `get_prospections` (le préfixe exact peut varier selon la connexion). Si ces outils ne sont pas chargés (liste d'outils différés), utilise ToolSearch avec la requête `"dgstock prospections"` avant de continuer.

2. **Interroger les prospections actionnables**, en priorité avec `dgstock_query_data` (resource=`prospections`, relations=`["client"]`) car il permet de filtrer/trier et d'inclure le client en une seule requête.

   **Piège testé en réel : l'opérateur `in` ne fonctionne pas sur le champ `etat`** (il renvoie 0 résultat même quand les valeurs existent). Fais une requête séparée par valeur d'état avec l'opérateur `=`, puis fusionne les résultats côté analyse. Pour connaître le volume total avant de lister (le `limit` par défaut est 20), lance d'abord un `aggregate: {fn: "count"}` par état pour savoir combien de dossiers existent, puis ajuste `limit` (max 200) en conséquence — sur ce compte, `SAD-Rappel` et `PC-Rappel` contiennent typiquement plusieurs centaines de dossiers cumulés, largement au-delà de la limite par défaut.

   Récupère, par ordre de priorité :
   - **Priorité 1 — Rendez-vous de rappel déjà pris** : `etat` = `PC-Rappel` ou `SAD-Rappel` (le client a déjà accepté d'être recontacté ; les `SAD-Rappel` passent avant les `PC-Rappel` car ils ont déjà vu une démo).
   - **Priorité 2 — Nouveaux leads non travaillés** : `etat` = `Premiere Entree` ou `deuxieme entree`, créés depuis plus de 48h (filtre sur `created_at`).
   - **Priorité 3 — À retenter** : `etat` = `PC-NRP`, `PC-INJ`, `PC-ligne ocupée` (le contact a échoué une fois, mérite une nouvelle tentative — mais signale ceux qui sont très anciens comme candidats à l'abandon plutôt qu'à la relance).
   - **Ne pas inclure par défaut** les états terminaux négatifs (`PC-pas interssé`, `SAD-Pas Intéressé`, `Faux numero`) ni `Intéressé`/`Encours de livraison` (déjà gagnés, suivis par le skill conversion-prospection-prestation) — sauf si l'utilisateur demande explicitement une "réactivation" ou une vue complète.
   - Si l'utilisateur précise une fenêtre de dates ou un état particulier, respecte sa demande plutôt que ces filtres par défaut.

3. **Trier** chaque groupe de priorité par `created_at` décroissant (le plus ancien d'abord = le plus urgent à traiter), et calcule le nombre de jours écoulés depuis `created_at`.

4. **Mettre en avant la valeur** : utilise `restant` ou `totalTTC` pour signaler les dossiers à forte valeur en tête de chaque groupe de priorité (un `SAD-Rappel` à 200 000 DZD passe avant un `SAD-Rappel` à 5 000 DZD).

5. **Produire le rapport** avec cette structure exacte :

```markdown
# Suivi des prospections — [date du jour]

## Priorité 1 — Rappels déjà programmés (PC/SAD)
| Référence | Client | Téléphone | Étape | Montant | Jours d'attente | Action |
|---|---|---|---|---|---|---|

## Priorité 2 — Nouveaux leads sans suite (>48h)
| Référence | Client | Téléphone | Créé le | Montant | Jours d'attente | Action |
|---|---|---|---|---|---|---|

## Priorité 3 — À retenter (NRP / injoignable / ligne occupée)
| Référence | Client | Téléphone | Échec précédent | Montant | Jours d'attente | Action |
|---|---|---|---|---|---|---|

## Résumé
- Nombre total de dossiers actionnables : X
- Valeur totale en jeu (somme des montants ci-dessus) : X DZD
- Dossiers à plus de 30 jours sans contact (à reconsidérer) : X
```

Le téléphone n'est pas toujours présent sur la relation `client` incluse dans `prospections` — si absent, va le chercher via `dgstock_fetch_clients` (recherche par nom) ou indique "à vérifier" plutôt que d'inventer une valeur.

6. **Proposer la suite** : termine en demandant si l'utilisateur veut un export PDF (`dgstock_generate_pdf`) ou si tu dois croiser cette liste avec les conversations WhatsApp (skill `dgstock-priorite-contact-clients`) pour enrichir le contexte avant les appels.

## Pièges à éviter

- Ne pas confondre `total`/`totalTTC` (montant du contrat) avec `restant` (ce qui reste à encaisser) — pour les prospections pas encore facturées, `restant` = `totalTTC` en général, c'est normal.
- L'agrégation par `group_by` sur `prospections` peut être lente (timeout) sans filtre de dates sur un compte avec beaucoup d'historique — toujours borner avec `date_from`/`date_to` si tu fais des agrégations, pas pour les listes simples.
- Le champ `etat` est sensible à l'orthographe exacte du compte (ex. "interssé" sans accent, "ocupée" avec une seule c) — recherche avec l'opérateur `like` plutôt qu'une égalité stricte si tu n'es pas sûr de l'orthographe exacte utilisée par ce compte.
- **Ne sous-estime jamais le volume du backlog `SAD-Rappel`/`PC-Rappel`.** Sur ce compte, ce sont des centaines de dossiers, dont certains datent de plusieurs mois (vérifié : un `SAD-Rappel` peut dater de janvier alors qu'on est en juin). Toujours afficher dans le résumé le nombre total par état ET l'ancienneté du dossier le plus vieux — c'est souvent l'info la plus actionnable du rapport, plus que la liste elle-même qui ne peut montrer que les premiers dossiers.
