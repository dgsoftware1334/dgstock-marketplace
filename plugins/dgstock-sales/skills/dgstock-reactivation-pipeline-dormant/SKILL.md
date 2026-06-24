---
name: dgstock-reactivation-pipeline-dormant
description: Construit une campagne de récupération ciblée sur les prospections DGStock à forte valeur jamais relancées (gros tickets et dossiers post-démo dormants depuis des semaines/mois). Utilise OBLIGATOIREMENT ce skill quand l'utilisateur demande "réactiver les anciens prospects", "le pipeline dormant", "récupérer les ventes perdues", "les dossiers oubliés", "relancer les vieux devis", ou toute variante de win-back / récupération de pipeline. Déclenche aussi sur "/dgstock-reactivation-pipeline-dormant" ou "/dgstock-reactivation".
---

# Réactivation du pipeline dormant (win-back)

## L'enjeu chiffré

Le pipeline DGStock contient souvent un gisement important de valeur abandonnée : des prospects assez
intéressés pour avoir reçu un devis (gros ticket) ou avoir vu une démonstration (`SAD-Rappel`), mais
**jamais rappelés** — parfois depuis des mois. Commence par **mesurer ce gisement en direct via le MCP**
(nombre de dossiers et valeur cumulée, voir Déroulé) : c'est souvent bien plus gros qu'on ne le croit.
Récupérer ne serait-ce que quelques pourcents de ce stock pèse plus que beaucoup de leads neufs. Ce
skill transforme ce stock en **campagne priorisée et exécutable**.

C'est complémentaire de `dgstock-suivi-prospections` (rappels du jour) : ici on traite spécifiquement
l'**ancien** (>14 jours sans contact), par lots, comme une opération de reconquête.

## Lecture seule

Le MCP est en lecture seule : ce skill produit la cible et l'angle de réapproche, il ne relance pas
lui-même et ne modifie aucun statut. (Pour rédiger les messages, enchaîner avec `dgstock-messages-relance-prets`.)

## Déroulé

1. **Définir le seuil de dormance.** Par défaut : `created_at` ≤ aujourd'hui − 14 jours, états
   `PC-Rappel` ou `SAD-Rappel` (jamais conclus). L'utilisateur peut demander un autre seuil (30/60/90 j).

2. **Quantifier d'abord, lister ensuite.** Le volume se compte en centaines : commence par des
   `aggregate {fn:count}` et `{fn:sum, field:"restant"}` par segment pour chiffrer le gisement, AVANT
   de lister (limit max 200). Une requête `=` par état (l'opérateur `in` ne marche pas).

3. **Segmenter par valeur / produit.** Déduis le produit de chaque dossier à partir de son montant
   `totalTTC`, en rapprochant ce montant des **tarifs réels du compte lus via le MCP** (ne code aucun
   prix en dur — voir le playbook). Puis segmente :
   - **Gros tickets (produit le plus cher)** — priorité n°1 de réactivation (un seul deal = beaucoup de CA).
   - **SAD-Rappel toutes valeurs** — ont vu une démo, donc les plus avancés : forte probabilité résiduelle.
   - **Tickets de valeur intermédiaire** — à traiter ensuite.
   - **Petits tickets / produit d'entrée** — volume, cycle court : traiter en lot.
   - Écarter le bruit : montants à 0, `Faux numero`, noms manifestement non commerciaux.

4. **Prioriser dans chaque segment** par valeur puis par « réactivabilité » : un dossier avec une
   conversation WhatsApp qui s'était bien passée (intérêt, démo vue) est plus chaud qu'un `PC-NRP`
   jamais joint. Si le temps le permet, ouvre la conversation des meilleurs dossiers pour retrouver le
   contexte (objection laissée, raison du silence) et proposer un angle de réapproche personnalisé.

5. **Proposer un angle de réapproche par segment** (rôle de directeur commercial) — ex. :
   - Gros tickets dormants : « nouveauté produit / offre de relance limitée » pour créer une raison d'agir maintenant.
   - SAD dormants : « on n'a pas pu finaliser la démo, on vous refait une proposition adaptée ».
   - Lot petits tickets : message de réveil court + remise de réactivation.

## Format de sortie

```markdown
# Campagne de réactivation — pipeline dormant (> [seuil] jours) — [date]

## Le gisement en chiffres
| Segment | Nb dossiers | Valeur dormante | Plus ancien |
|---|---|---|---|
| Gros tickets | X | X DZD | [date] |
| SAD-Rappel (tous) | X | X DZD | [date] |
| ... | | | |
| **Total** | **X** | **X DZD** | |

## Cibles prioritaires (vague 1)
| Référence | Client | Téléphone | Produit | Montant | Jours dormant | Angle de réapproche |
|---|---|---|---|---|---|---|

## Plan de campagne suggéré
- Vague 1 (cette semaine) : [les gros tickets les plus récents parmi les dormants]
- Vague 2 : [SAD-Rappel]
- Vague 3 : [lot de petits tickets en relance groupée]
- Angle/offre recommandé par segment + mesure de succès (nb de rappels aboutis, réactivations).
```

Termine en proposant d'enchaîner avec `dgstock-messages-relance-prets` pour rédiger les messages de la vague 1,
ou avec `dgstock-fiche-preparation-appel` pour préparer les plus gros dossiers.

## Rappels techniques (MCP DGStock)

- Borne toujours les agrégations par dates (sinon timeout sur l'historique complet).
- `limit` max 200, pas de pagination → traiter par lots (filtre `created_at`/`numero`).
- Téléphone parfois absent de la relation `client` → `dgstock_fetch_clients`, sinon « → à vérifier ».
- `dgstock_generate_pdf` en panne : produire HTML/Markdown si export demandé.
