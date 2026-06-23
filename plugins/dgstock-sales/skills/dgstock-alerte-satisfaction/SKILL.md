---
name: dgstock-alerte-satisfaction
description: Liste les réclamations non traitées anciennes et les clients récurrents insatisfaits dans le CRM DGStock, pour éviter qu'un problème client reste sans réponse trop longtemps. Utilise OBLIGATOIREMENT ce skill quand l'utilisateur demande "y a-t-il des réclamations en attente", "quels clients sont mécontents", "alerte satisfaction", "réclamations non traitées", ou toute variante de surveillance qualité/satisfaction DGStock. Déclenche aussi sur "/dgstock-alerte-satisfaction".
---

# Alerte satisfaction (réclamations & avis)

## Comment exécuter ce skill

### 1. Réclamations non traitées
Interroge `dgstock_query_data` (resource=`reclamations`, relations=`["client"]`).

**Piège confirmé sur ce compte : le champ `traite` n'est pas un booléen propre.** On y trouve `false`, `0`, `1`, et des valeurs vides selon l'enregistrement. Une seule requête `filters: [{field:"traite", op:"=", value:"false"}]` rate les réclamations où `traite` est `0` ou vide. Fais donc au minimum deux passes (`= "false"` et `= "0"`), et regarde si des réclamations récupérées sans filtre ont un `traite` vide/null — traite tout ce qui n'est pas explicitement `1`/`true` comme non résolu.

Trie par ancienneté (`created_at` croissant) : la réclamation la plus vieille est la priorité n°1, pas la plus récente. Sur ce compte, une réclamation peut rester ouverte plusieurs mois (cas réel observé : 5,5 mois sans réponse) — un délai aussi long est en soi l'alerte, indépendamment du contenu.

**Détecte les clients récurrents** : si le même nom de client apparaît sur plusieurs réclamations (même non résolues différentes), signale-le séparément — c'est un signal de risque de perte bien plus fort qu'une réclamation isolée, et facile à manquer si on traite chaque ligne indépendamment.

### 2. Avis clients à surveiller
Interroge `avis_clients` filtré sur `note <= 2`. **Ne t'arrête pas à la note** : sur ce compte, beaucoup de notes basses correspondent en réalité à des notes d'appel commercial ("nrp" = pas de réponse, "appelle plus tard", "rappeler") plutôt qu'à une vraie insatisfaction produit. Lis systématiquement le champ `commentaire` et classe chaque avis en deux catégories :
- **Vraie insatisfaction** (plainte sur le produit/service, problème non résolu, ton clairement négatif) → à remonter.
- **Note d'appel/relance** (juste un statut de contact, "pas de réponse", "à rappeler plus tard") → à transférer plutôt vers le suivi pipeline (`dgstock-suivi-prospections`/`dgstock-priorite-contact-clients`) qu'à traiter comme une alerte qualité.

Ne présente jamais "note moyenne = X/5" seule sans cette mise en garde — un lecteur non averti la prendrait pour un vrai score de satisfaction.

### 3. Produire le rapport

```markdown
# Alerte satisfaction — [date]

## Réclamations non traitées (par ancienneté)
| Référence | Client | Sujet | Ouverte depuis | Récurrent ? |
|---|---|---|---|---|

## Avis nécessitant une vraie attention (insatisfaction réelle, pas note d'appel)
| Client | Note | Commentaire (résumé) | Date |
|---|---|---|---|

## Clients à risque (réclamations/avis négatifs multiples)
[liste des noms qui reviennent plusieurs fois]

## Résumé
- X réclamations non traitées, la plus ancienne depuis X jours
- X clients récurrents à surveiller
- X avis représentant une vraie insatisfaction (sur Y avis notés ≤2 au total)
```

4. Propose un export PDF si la liste est conséquente, en précisant que cet outil peut être temporairement indisponible (voir skill `dgstock-rapport-hebdomadaire-pdf`) — dans ce cas, présente le rapport en Markdown directement.

## Pourquoi ce skill est important même sans accès écriture

Ce skill ne peut pas répondre à une réclamation ni changer son statut, mais il évite le vrai risque : qu'une réclamation **disparaisse silencieusement** parce qu'elle n'est jamais ressortie dans aucun rapport. Le simple fait de la remonter régulièrement à quelqu'un est déjà la valeur du skill.
