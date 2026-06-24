---
name: dgstock-prevision-ventes
description: Produit une prévision de ventes DGStock à partir du pipeline pondéré par état et par produit, plus le réalisé (prestations confirmées). Utilise OBLIGATOIREMENT ce skill quand l'utilisateur demande "la prévision de ventes", "le forecast", "combien on va faire ce mois/trimestre", "le CA prévisionnel", "valeur pondérée du pipeline", ou toute variante de prévision/projection commerciale. Déclenche aussi sur "/dgstock-prevision-ventes" ou "/dgstock-forecast".
---

# Prévision de ventes (forecast pondéré)

## Principe et honnêteté de la méthode

Une prévision n'est pas une certitude : c'est le pipeline **pondéré par la probabilité de conclure à
chaque étape**, plus ce qui est déjà réalisé. Ce skill donne une fourchette raisonnée, pas un chiffre
magique — et il affiche toujours ses hypothèses pour que la direction puisse les ajuster.

⚠️ Deux limites à annoncer dans le rapport :
1. **Pas de date de clôture fiable** dans les prospections (`date_fin` quasi toujours vide) → la
   prévision est « à pipeline constant », pas calée sur un calendrier précis. On raisonne en potentiel
   du pipeline actuel, à répartir sur les prochaines semaines.
2. Les coefficients de pondération ci-dessous sont des **hypothèses de départ** : si l'historique réel
   de conversion par état devient disponible, les recaler. Toujours présenter le forecast avec ses
   coefficients visibles.

## Lecture seule

Le MCP est en lecture seule : ce skill calcule et projette, il ne modifie rien.

## Modèle de pondération (hypothèses de départ, ajustables)

Probabilité de conclusion par état du pipeline :

| État | Prob. de conclure | Justification |
|---|---|---|
| `Encours de livraison` | 90% | quasi gagné |
| `Intéressé` | 60% | intérêt explicite |
| `SAD-Rappel` | 30% | a vu une démo, à relancer |
| `PC-Rappel` | 12% | premier contact, rappel convenu |
| `Premiere Entree` | 5% | lead brut |
| `PC-NRP` / `PC-INJ` / `PC-ligne ocupée` | 2% | contact non établi |
| `*pas interssé` / `Faux numero` | 0% | exclus du forecast |

Produit déduit en rapprochant le montant (`totalTTC`) des tarifs réels du compte, lus via le MCP (ne
code aucun prix en dur) ; un montant à 0 = non qualifié (exclu du chiffrage).

## Déroulé

1. **Réalisé** : somme des `prestations` `Vente confirmee` sur la période demandée
   (`aggregate {fn:sum, field:"totalTTC"}`, borné par dates). C'est le socle certain.
2. **Pipeline pondéré** : pour chaque état retenu, `aggregate {fn:count}` et
   `{fn:sum, field:"totalTTC"}` (une requête `=` par état — l'opérateur `in` ne marche pas, et borner
   par dates pour éviter les timeouts). Multiplie la valeur de chaque état par sa probabilité →
   contribution pondérée. Somme = **forecast du pipeline**.
3. **Ventiler par produit** (DGStock vs DGProduction surtout) pour montrer d'où vient le CA prévu et
   sur quel produit se concentre le potentiel.
4. **Donner une fourchette** : basse (réalisé + pipeline très avancé seulement : Encours + Intéressé),
   médiane (modèle complet ci-dessus), haute (si les taux de conversion s'amélioraient). Cela vaut
   mieux qu'un point unique faussement précis.
5. **Mettre en perspective** : signale les gros contributeurs (ex. le stock du produit le plus cher) et
   le fait qu'une partie du forecast dort (voir `dgstock-reactivation-pipeline-dormant`) — donc
   « activable » mais pas automatique.

## Format de sortie

```markdown
# Prévision de ventes — [période] (à pipeline constant)

## Hypothèses
[Tableau des coefficients de pondération utilisés + limites annoncées.]

## Réalisé
- Ventes confirmées sur la période : X DZD (X prestations)

## Pipeline pondéré
| État | Nb | Valeur brute | Prob. | Contribution pondérée |
|---|---|---|---|---|
| ... | | | | |
| **Forecast pipeline** | | | | **X DZD** |

## Forecast par produit
| Produit | Valeur brute pipeline | Contribution pondérée |
|---|---|---|

## Fourchette
- Basse : X DZD · Médiane : X DZD · Haute : X DZD

## Lecture commerciale
[2-3 points : d'où vient le CA prévu, ce qui est activable (dormant), où pousser pour sécuriser la médiane.]
```

## Garde-fous

- Toujours afficher les coefficients : un forecast sans hypothèses visibles n'est pas pilotable.
- Si une somme `totalTTC` renvoie 0 de façon suspecte (champ stocké en texte sur certains
  enregistrements), signale l'anomalie plutôt que de fausser le total.
- Distingue **réalisé** (certain) et **pondéré** (probable) — ne les additionne pas sans le dire.
- `dgstock_generate_pdf` en panne : produire HTML/Markdown si export demandé.
