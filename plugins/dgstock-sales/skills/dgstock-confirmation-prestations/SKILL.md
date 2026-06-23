---
name: dgstock-confirmation-prestations
description: Produit la liste des prestations (ventes DGStock/DG-SALE) à confirmer et la liste des impayés à relancer, avec le montant total en jeu. Utilise OBLIGATOIREMENT ce skill quand l'utilisateur demande "quelles ventes sont à confirmer", "qui me doit de l'argent", "liste des impayés", "relance paiement", "chiffre d'affaires en attente", ou toute variante sur le suivi des prestations/factures/encaissements DGStock. Déclenche aussi sur "/dgstock-confirmation-prestations".
---

# Confirmation des prestations & relance impayés (DGStock)

## Contexte métier

Une "prestation" dans DGStock = une vente/facture (pas un service au sens littéral). Elle suit deux signaux indépendants :
- **`etat`** : `Premiere Entree` (vente pas encore confirmée) vs `Vente confirmee` (vente actée).
- **`restant`** : montant qui reste à encaisser (`restant` = `totalTTC` si rien n'a encore été payé ; `restant` = 0 si tout est payé).

Ces deux signaux sont indépendants : une vente peut être `Vente confirmee` et avoir un `restant` > 0 (vendu mais pas encore payé intégralement).

## Lecture seule

Le MCP DGStock ne permet aucune écriture. Ce skill produit des listes d'action (qui appeler pour confirmer, qui relancer pour le paiement) — il ne change jamais de statut et n'envoie jamais de message lui-même.

## Comment exécuter ce skill

1. Localise les outils MCP DGStock (`dgstock_query_data`, `dgstock_fetch_prestations`). Si absents de la liste d'outils chargés, utilise ToolSearch avec `"dgstock prestations"`.

2. **Ventes à confirmer** : interroge `dgstock_query_data` (resource=`prestations`, relations=`["client"]`, filters=`[{field: "etat", op: "=", value: "Premiere Entree"}]`, sort par `created_at` ascendant). Ce sont des ventes saisies mais pas encore validées — chaque jour qui passe sans confirmation est un risque de perte. Vérifie d'abord le volume total avec `aggregate: {fn: "count"}` avant de fixer le `limit` (la valeur par défaut de 20 peut être trop basse).

3. **Impayés à relancer** : même resource, `filters=[{field: "restant", op: ">", value: "0"}]`, trié par `restant` décroissant (les plus gros montants en tête) puis par ancienneté. Calcule aussi le total : `aggregate: {fn: "sum", field: "restant"}` avec les mêmes filtres pour donner le montant total impayé, et compare-le à `aggregate: {fn: "sum", field: "totalTTC"}` (sans filtre, ou filtré sur la période demandée) pour donner un taux d'impayé en %.

   **Piège connu** : l'opérateur `in` ne fonctionne pas de façon fiable sur les champs liés à un état dans ce MCP — préfère des requêtes séparées par valeur si tu as besoin de combiner plusieurs états, et fusionne les résultats toi-même.

4. **Si l'utilisateur précise une période** (ce mois, cette semaine), utilise `date_from`/`date_to` sur `created_at` pour borner. Sans précision, prends l'ensemble du portefeuille pour les impayés (l'argent dû ne disparaît pas avec le temps) mais limite les "ventes à confirmer" aux 30 derniers jours par défaut (au-delà, signale-les comme probablement obsolètes plutôt que comme priorité du jour).

5. **Produire le rapport** :

```markdown
# Confirmation des prestations & impayés — [date]

## Ventes à confirmer (etat = Premiere Entree)
| Référence | Client | Montant TTC | Créé le | Jours d'attente |
|---|---|---|---|---|

## Impayés à relancer (restant > 0), triés par montant
| Référence | Client | Montant restant | Montant total | Créé le | Jours de retard |
|---|---|---|---|---|---|

## Résumé financier
- Chiffre d'affaires facturé total : X DZD
- Total impayé : X DZD (X % du facturé)
- Nombre de ventes en attente de confirmation : X
- Plus gros impayé individuel : [client] — X DZD
```

6. Termine en proposant un export PDF (`dgstock_generate_pdf`) et, si pertinent, en signalant les clients qui apparaissent à la fois dans les impayés ici et dans le pipeline `prospections` (signal d'un client à risque pour de futures ventes).

## Pièges à éviter

- `total`/`totalTTC`/`restant` sont parfois stockés en texte plutôt qu'en nombre selon l'enregistrement — si une agrégation `sum` renvoie 0 de façon suspecte alors que les lignes individuelles ont des valeurs non nulles, c'est probablement un problème de typage de champ côté source : signale-le à l'utilisateur plutôt que de conclure que le montant réel est zéro.
- Ne pas confondre "vente à confirmer" (etat) et "vente impayée" (restant) — ce sont deux listes différentes, parfois avec des dossiers en commun, et il faut les présenter séparément pour que l'action à mener soit claire (confirmer ≠ relancer un paiement).
