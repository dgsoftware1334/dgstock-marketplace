---
name: dgstock-reactivation-pipeline-dormant
description: Construit une campagne de récupération ciblée sur les prospections DGStock à forte valeur jamais relancées (gros tickets DGProduction 60K et dossiers post-démo SAD-Rappel dormants depuis des semaines/mois). Utilise OBLIGATOIREMENT ce skill quand l'utilisateur demande "réactiver les anciens prospects", "le pipeline dormant", "récupérer les ventes perdues", "les dossiers oubliés", "relancer les vieux devis", ou toute variante de win-back / récupération de pipeline. Déclenche aussi sur "/dgstock-reactivation-pipeline-dormant" ou "/dgstock-reactivation".
---

# Réactivation du pipeline dormant (win-back)

## L'enjeu chiffré

Le pipeline DGStock contient un gisement énorme de valeur abandonnée : des prospects assez intéressés
pour avoir reçu un devis (souvent DGProduction à 60 000 DA) ou avoir vu une démonstration (`SAD-Rappel`),
mais **jamais rappelés** — certains depuis des mois. À titre de repère mesuré sur ce compte : des
centaines de dossiers `PC-Rappel` à 60 000 DA antérieurs à mi-juin (~15 M DZD) et ~130 `SAD-Rappel`
(~7 M DZD). Récupérer ne serait-ce que quelques pourcents de ce stock pèse plus que beaucoup de leads
neufs. Ce skill transforme ce stock en **campagne priorisée et exécutable**.

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

3. **Segmenter par valeur / produit** (déduction par montant) :
   - **DGProduction 60 000 DA** — gros ticket, priorité n°1 de réactivation (un seul deal = beaucoup de CA).
   - **SAD-Rappel toutes valeurs** — ont vu une démo, donc les plus avancés : forte probabilité résiduelle.
   - **Entreprise / Distributeur (45-53K / 15-17K)** — valeur élevée intermédiaire.
   - **DGStock 10 500 DA** — volume, cycle court : traiter en lot.
   - Écarter le bruit : montants à 0, `Faux numero`, noms manifestement non commerciaux.

4. **Prioriser dans chaque segment** par valeur puis par « réactivabilité » : un dossier avec une
   conversation WhatsApp qui s'était bien passée (intérêt, démo vue) est plus chaud qu'un `PC-NRP`
   jamais joint. Si le temps le permet, ouvre la conversation des meilleurs dossiers pour retrouver le
   contexte (objection laissée, raison du silence) et proposer un angle de réapproche personnalisé.

5. **Proposer un angle de réapproche par segment** (rôle de directeur commercial) — ex. :
   - 60K dormants : « nouveauté produit / offre de relance limitée » pour créer une raison d'agir maintenant.
   - SAD dormants : « on n'a pas pu finaliser la démo, on vous refait une proposition adaptée ».
   - Lot 10 500 : message de réveil court + remise de réactivation.

## Format de sortie

```markdown
# Campagne de réactivation — pipeline dormant (> [seuil] jours) — [date]

## Le gisement en chiffres
| Segment | Nb dossiers | Valeur dormante | Plus ancien |
|---|---|---|---|
| DGProduction 60K | X | X DZD | [date] |
| SAD-Rappel (tous) | X | X DZD | [date] |
| ... | | | |
| **Total** | **X** | **X DZD** | |

## Cibles prioritaires (vague 1)
| Référence | Client | Téléphone | Produit | Montant | Jours dormant | Angle de réapproche |
|---|---|---|---|---|---|---|

## Plan de campagne suggéré
- Vague 1 (cette semaine) : [X dossiers 60K les plus récents parmi les dormants]
- Vague 2 : [SAD-Rappel]
- Vague 3 : [lot 10 500 en relance groupée]
- Angle/offre recommandé par segment + mesure de succès (nb de rappels aboutis, réactivations).
```

Termine en proposant d'enchaîner avec `dgstock-messages-relance-prets` pour rédiger les messages de la vague 1,
ou avec `dgstock-fiche-preparation-appel` pour préparer les plus gros dossiers.

## Rappels techniques (MCP DGStock)

- Borne toujours les agrégations par dates (sinon timeout sur l'historique complet).
- `limit` max 200, pas de pagination → traiter par lots (filtre `created_at`/`numero`).
- Téléphone parfois absent de la relation `client` → `dgstock_fetch_clients`, sinon « → à vérifier ».
- `dgstock_generate_pdf` en panne : produire HTML/Markdown si export demandé.
