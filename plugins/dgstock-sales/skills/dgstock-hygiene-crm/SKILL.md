---
name: dgstock-hygiene-crm
description: Détecte les doublons clients (par téléphone), les fiches incomplètes (email/adresse manquants) et les prospections orphelines (faux numéro, etc.) dans le CRM DGStock. Utilise OBLIGATOIREMENT ce skill quand l'utilisateur demande "trouve les doublons", "nettoie la base clients", "quelles fiches sont incomplètes", "audit qualité du CRM", ou toute variante d'hygiène/nettoyage de données DGStock. Déclenche aussi sur "/dgstock-hygiene-crm".
---

# Hygiène du CRM DGStock

## Limite structurelle importante à connaître avant de commencer

**Ce MCP n'a pas de pagination (pas de paramètre `offset`/page) et le `limit` maximum est 200.** Sur une base qui compte en général plusieurs milliers de clients et de prospections, il est **impossible de scanner l'intégralité de la base en une seule passe**. Toute analyse d'hygiène est donc soit un échantillon ciblé (rapide), soit un balayage par lots successifs (lent mais complet) — annonce toujours à l'utilisateur laquelle des deux tu fais.

**`group_by` sur un champ à forte cardinalité (ex. `telephone` sur des milliers de clients) renvoie une erreur "trop de tokens"** — ne l'utilise jamais pour détecter des doublons sur l'ensemble de la base, ça ne fonctionne pas et gaspille un appel.

## Méthode qui fonctionne : tri par téléphone

La technique validée pour repérer des doublons est de **trier les clients par `telephone` (ordre croissant)** avec `dgstock_query_data` (resource=`clients`, sort=`{field: "telephone", dir: "asc"}`, limit=200) et de scanner le résultat : deux fiches avec le même numéro apparaissent côte à côte. Exemple type : un même numéro associé à deux fiches clients distinctes créées à des dates différentes — c'est exactement ce que tu cherches.

- **Scan rapide (par défaut)** : un seul lot de 200 (les premiers selon le tri). Donne une estimation du taux de doublons, pas une liste exhaustive — dis-le clairement à l'utilisateur.
- **Scan complet (si l'utilisateur demande un nettoyage exhaustif)** : répète l'appel en avançant dans l'alphabet/les chiffres du téléphone (ex. filtre `telephone >= dernier numéro vu` à chaque lot) jusqu'à couvrir toute la base. Préviens l'utilisateur que cela peut représenter un grand nombre d'appels selon la taille de la base et demande confirmation avant de lancer un scan complet (ce n'est pas instantané).

Pour vérifier un doublon précis suspecté (ex. un commercial pense avoir saisi deux fois le même client), utilise plutôt `dgstock_fetch_clients` avec `search` sur le nom ou le numéro — bien plus rapide qu'un scan complet.

## Fiches incomplètes

Sur ce type de compte, **une grande partie des clients n'ont pas d'email renseigné**, et le prénom/l'adresse sont souvent vides. Ne traite pas ça comme une anomalie à corriger en masse (le canal réel de la relation client est le téléphone/WhatsApp, pas l'email) — utilise plutôt cette info pour scoper : ne cible la complétion de fiche que pour des sous-ensembles à valeur ajoutée claire, par exemple :
- Les clients les plus récents (`sort: created_at desc`) sans email — plus facile à enrichir tant que le contact est encore chaud.
- Les clients ayant une prestation confirmée de valeur élevée — ces fiches méritent d'être complètes pour la facturation/suivi.

Requête type : `dgstock_query_data` (resource=`clients`, filters=`[{field:"email", op:"null"}]`, sort=`{field:"created_at", dir:"desc"}`, limit=20).

## Prospections orphelines / suspectes

Recherche les états annexes qui indiquent une fiche à nettoyer plutôt qu'à travailler commercialement : `Faux numero`, `deuxieme entree` (doublon probable de saisie), `Reclamation version d'essai` (mal classée comme prospection). Utilise une requête `=` par état (l'opérateur `in` ne fonctionne pas sur `etat`, voir skill `dgstock-suivi-prospections`).

## Produire le rapport

```markdown
# Hygiène CRM — [date]

## Doublons détectés (scan [rapide/complet])
| Téléphone | Fiche 1 | Fiche 2 | Créées le |
|---|---|---|---|

## Fiches incomplètes prioritaires
| Client | Téléphone | Champ manquant | Créé le |
|---|---|---|---|

## Prospections à nettoyer
| Référence | État | Client |
|---|---|---|

## Résumé
- X doublons trouvés sur l'échantillon scanné (200 clients sur l'ensemble de la base — scan partiel)
- X fiches récentes incomplètes à enrichir en priorité
- X prospections à classer comme non commerciales
```

Sois honnête sur la couverture du scan dans le résumé — un "X doublons trouvés" sans préciser "sur un échantillon de 200" donnerait une fausse impression d'exhaustivité.
