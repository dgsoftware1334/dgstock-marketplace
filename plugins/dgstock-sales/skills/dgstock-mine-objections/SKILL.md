---
name: dgstock-mine-objections
description: Analyse les conversations WhatsApp DGStock pour extraire les objections récurrentes des clients (abonnement vs licence définitive, prix, fonctionne hors-ligne, demande de démo, etc.), les classer par fréquence et par impact sur la vente. Utilise OBLIGATOIREMENT ce skill quand l'utilisateur demande "quelles objections reviennent", "pourquoi les clients hésitent", "analyse des freins à l'achat", "ce qui bloque les ventes", "améliorer le script du bot/des commerciaux", ou toute variante d'analyse des objections/freins. Déclenche aussi sur "/dgstock-mine-objections" ou "/dgstock-objections".
---

# Mine d'objections (analyse des freins à l'achat)

## À quoi ça sert (vision directeur commercial)

Les mêmes objections reviennent dans presque toutes les conversations et font perdre des ventes
évitables. Exemples déjà observés sur ce compte : « c'est un abonnement ou je l'achète une fois pour
toutes ? » (source de frustration et d'abandon), « est-ce que ça marche hors-ligne ? », « vous avez
une version démo ? », « c'est trop cher ». Ce skill transforme des dizaines de conversations en une
**carte des objections** exploitable à trois niveaux :
1. **Bot** : corriger/compléter le script WhatsApp pour traiter l'objection dès qu'elle apparaît.
2. **Commerciaux** : leur donner les meilleures réponses (coaching, fiche de réponses-types).
3. **Marketing** : produire du contenu (vidéo, post, FAQ) qui désamorce l'objection en amont.

Ce n'est pas un rapport de contacts : c'est un rapport d'**apprentissage** pour améliorer le discours.

## Lecture seule

Le MCP est en lecture seule. Ce skill lit les conversations et en tire une analyse ; il ne répond à
personne et ne modifie pas le bot. Les changements de script/contenu sont décidés par l'humain.

## Déroulé

1. **Échantillonner les conversations** : `dgstock_fetch_whatsapp_conversations` puis
   `dgstock_fetch_whatsapp_messages` sur un échantillon représentatif (privilégier les conversations
   avec un `messages_count` élevé — c'est là que se logent les objections, pas dans les échanges d'un
   message). Vise un échantillon utile (ex. 20-40 conversations) plutôt que l'exhaustivité ; annonce la
   taille de l'échantillon dans le rapport.

2. **Lire et catégoriser les objections.** Les messages sont en darija (latin/arabe) ou français —
   comprends le sens, ne te limite pas aux mots-clés. Catégories de départ (à enrichir selon ce que tu
   observes réellement) :
   - **Modèle commercial** : abonnement annuel vs licence définitive (« اشتراك ولا دائم »), récurrence du paiement.
   - **Prix** : trop cher, demande de remise, comparaison.
   - **Technique** : fonctionne hors-ligne ? sur quel appareil ? migration des données ?
   - **Confiance / preuve** : veut une démo / version d'essai, veut voir avant de payer, doute sur le sérieux.
   - **Décision** : « je réfléchis », doit consulter un associé, pas le décideur.
   - **Inadéquation** : le produit ne correspond pas au besoin (souvent = mauvais ciblage en amont).

3. **Mesurer fréquence ET impact.** Pour chaque objection : combien de conversations la contiennent, et
   surtout **quelle issue** (le client a-t-il abandonné, été perdu, ou conclu malgré l'objection ?).
   Une objection fréquente ET corrélée aux abandons est prioritaire ; une objection fréquente mais que
   le bot lève bien ne l'est pas. Distingue ces deux cas — c'est le cœur de la valeur du skill.

4. **Pour chaque objection prioritaire, proposer la parade** : la meilleure réponse observée (si une
   conversation l'a bien traitée, cite-la comme modèle), et une recommandation d'action (script bot /
   coaching / contenu marketing).

## Format de sortie

```markdown
# Mine d'objections — [période] (échantillon de X conversations)

## Top objections par fréquence et impact
| Objection | Fréquence | Issue dominante | Impact ventes | Priorité |
|---|---|---|---|---|
| Abonnement vs définitif | X conv. | abandon fréquent | 🔴 élevé | 1 |
| Hors-ligne ? | ... | ... | ... | ... |

## Pour chaque objection prioritaire
### [Objection]
- **Ce que disent les clients** : [exemples résumés en français]
- **Comment c'est traité aujourd'hui** : [bien / mal / éludé]
- **Meilleure réponse / parade recommandée** : [réponse-type]
- **Action** : 🤖 script bot · 🎓 coaching commercial · 📣 contenu marketing

## Recommandations transverses
[2-4 actions à plus fort effet : ex. clarifier le modèle d'abonnement partout, créer une FAQ vidéo offline...]
```

## Garde-fous

- Distingue **objection réelle** (frein à l'achat) de simple **question d'information** — toutes les
  questions ne sont pas des objections.
- Ne sur-généralise pas depuis 2-3 cas : indique la fréquence réelle et la taille de l'échantillon ;
  reste honnête sur ce que l'échantillon permet de conclure.
- Résume en français, ne colle pas le texte arabe brut.
- `dgstock_generate_pdf` en panne : produire HTML/Markdown si export demandé.
