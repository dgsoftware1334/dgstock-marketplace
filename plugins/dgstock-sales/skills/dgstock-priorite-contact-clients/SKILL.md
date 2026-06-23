---
name: dgstock-priorite-contact-clients
description: Construit une liste priorisée des clients/prospects à contacter pendant la semaine de travail (dimanche à jeudi), en croisant les conversations WhatsApp, les avis clients et les réclamations non traitées. Utilise OBLIGATOIREMENT ce skill quand l'utilisateur demande "qui dois-je contacter cette semaine", "quels clients sont en attente de réponse", "analyse les conversations WhatsApp", "y a-t-il des clients mécontents à rappeler", ou toute variante mêlant priorisation de contact + communications client. Déclenche aussi sur "/dgstock-priorite-contact-clients".
---

# Priorité de contact clients (WhatsApp + avis + réclamations)

## Pourquoi ce skill existe

Le bot WhatsApp de DGSoftware qualifie déjà les clients automatiquement, mais il **promet parfois des rappels dans le texte du message** ("on vous appelle ce soir", "notre commercial va vous contacter") **sans qu'aucun rendez-vous structuré ne soit créé dans le CRM** (le module `rendezvous` est vide sur ce compte). Ces engagements n'existent donc que dans le texte brut des conversations — ce skill est ce qui permet de ne pas les perdre.

La semaine de travail ici va de **dimanche à jeudi** (vendredi/samedi = week-end). Le rapport doit donc couvrir les 5 prochains jours ouvrés, pas une semaine calendaire classique.

## Lecture seule

Aucune écriture n'est possible sur ce MCP. Ce skill ne répond jamais à un message et ne marque jamais une conversation comme lue — il produit uniquement la liste de qui contacter, avec le contexte nécessaire pour que l'appel/message soit pertinent.

## Comment exécuter ce skill

### 1. Récupérer les conversations WhatsApp actives
Utilise `dgstock_fetch_whatsapp_connections` pour connaître les connexions actives, puis `dgstock_fetch_whatsapp_conversations` (sans filtre `unread` au départ — ce champ est souvent `null` côté données et n'est pas fiable seul) en triant mentalement par `last_message_at` décroissant. Pour les conversations les plus récentes ou les plus volumineuses (`messages_count` élevé), récupère le détail avec `dgstock_fetch_whatsapp_messages`.

### 2. Détecter qui attend une réponse
Chaque message a un champ `direction` : `inbound` (le client a écrit) ou `outbound` (l'agent/bot a répondu). **Le signal le plus fiable est : si le dernier message d'une conversation est `inbound`, le client attend une réponse — c'est une priorité haute, quel que soit le flag `is_read`/`status` (peu fiables sur ce compte).**

### 3. Détecter les engagements pris dans le texte
Les messages sont souvent en darija (arabe dialectal, parfois en alphabet latin, parfois en lettres arabes) ou en français. Lis le contenu des derniers messages `outbound` du bot/agent et repère les promesses de rappel : mentions d'un moment ("ce soir", "العشية" = ce soir, "غدوة"/"demain", "الصباح" = le matin) ou d'un commercial qui doit appeler. Si un engagement a été pris et que la date/moment est dépassée sans nouveau message, signale-le comme **"engagement non tenu"** — c'est souvent le signal le plus important du rapport, parce qu'il révèle une promesse faite au client et non honorée.

### 4. Croiser avec les réclamations non traitées
Interroge `dgstock_query_data` (resource=`reclamations`, filters=`[{field:"traite", op:"=", value:"false"}]`, relations=`["client"]`). **Attention** : le champ `traite` a un typage incohérent sur ce compte (`true`/`false`/`0`/`1`/vide) — vérifie aussi les valeurs `0` et vide comme "non traité", ne te fie pas uniquement à `false`. Priorise par ancienneté : une réclamation vieille de plusieurs mois est plus urgente qu'une nouvelle. Si un même client a plusieurs réclamations non traitées, signale-le explicitement ("client récurrent insatisfait") — c'est un signal de risque de perte plus fort qu'une réclamation isolée.

### 5. Lire les avis clients avec discernement, pas juste la note
La `note` (1-5) sur ce compte ne reflète pas toujours une vraie insatisfaction — beaucoup de `note: 1` correspondent en réalité à des notes d'appel ("nrp" = pas de réponse, "appelle plus tard"), pas à un mécontentement. **Lis le champ `commentaire` pour juger du vrai contexte avant de classer un avis comme "client mécontent".** Un commentaire qui dit juste "rappeler plus tard" est une tâche de relance, pas une alerte satisfaction.

### 6. Pondérer par valeur business (optionnel mais utile)
Si tu as le temps, recherche rapidement (par nom de client) si la personne a une prospection en cours (`PC-Rappel`/`SAD-Rappel`, voir skill `dgstock-suivi-prospections`) ou un impayé (`prestations.restant > 0`, voir skill `dgstock-confirmation-prestations`). Un client qui a à la fois un message WhatsApp en attente ET un impayé en cours mérite d'être tout en haut de la liste.

### 7. Construire la liste et la répartir sur la semaine
Classe tous les signaux trouvés (étapes 2 à 6) par priorité décroissante :
1. Réclamation non traitée ancienne ou client récurrent
2. Engagement de rappel non tenu (promesse du bot non honorée)
3. Dernier message WhatsApp en attente de réponse (`inbound` non répondu)
4. Avis avec un commentaire clairement négatif (pas juste "nrp"/"rappeler")
5. Reste des signaux

Si la liste dépasse ~15-20 contacts pour un jour raisonnable de travail, **répartis-la sur les 5 jours ouvrés (dimanche → jeudi)** en mettant les priorités 1-2 dès dimanche, et en étalant le reste. Ne répartis pas arbitrairement — explique le critère (ancienneté, valeur, type de signal).

### 8. Produire le rapport

```markdown
# Priorité de contact — semaine du [dimanche] au [jeudi]

## Dimanche (urgences)
| Client | Téléphone | Signal | Contexte | Action recommandée |
|---|---|---|---|---|

## Lundi
...

## Mardi à jeudi
...

## Engagements non tenus détectés
[Liste séparée et mise en avant : promesses du bot/agent non honorées]

## Clients récurrents insatisfaits
[Clients avec plusieurs signaux négatifs]

## Résumé
- X clients à contacter cette semaine
- X engagements de rappel non tenus
- X réclamations non traitées (dont la plus ancienne : [date])
```

9. Termine en proposant un export PDF (`dgstock_generate_pdf`) si la liste est longue.

## Pièges à éviter

- Ne traduis pas le contenu des messages mot à mot dans le rapport si c'est de la darija — **résume en français le sens et l'intention** (ex. "le client a demandé à être rappelé le soir, l'agent a confirmé" plutôt que de coller le texte arabe brut), pour que la personne qui lit le rapport puisse agir vite sans devoir lire l'arabe.
- Ne classe jamais un client comme "mécontent" sur la seule base d'une note basse sans avoir lu le commentaire — c'est la principale source d'erreur observée sur ce compte.
- Le champ `is_read`/`status` des conversations est peu fiable (souvent `null`) — base-toi sur `direction` du dernier message, pas sur ces champs.
