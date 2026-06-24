---
name: dgstock-messages-relance-prets
description: Rédige des messages WhatsApp de relance personnalisés, prêts à copier-coller, pour une liste de prospects/clients DGStock — dans la langue du client (darija en lettres latines, darija en lettres arabes, ou français). Utilise OBLIGATOIREMENT ce skill quand l'utilisateur demande "rédige les messages de relance", "prépare les WhatsApp à envoyer", "écris un message pour ce client", "des messages prêts à envoyer", ou toute variante de rédaction de relances commerciales. Déclenche aussi sur "/dgstock-messages-relance-prets" ou "/dgstock-messages-relance".
---

# Messages de relance prêts à envoyer (WhatsApp)

## Ce que fait ce skill et sa limite

Pour une équipe qui vend par WhatsApp, le goulot n'est pas « qui relancer » mais « écrire chaque
message ». Ce skill élimine ce temps mort : il **rédige le message, le commercial relit et envoie
lui-même**. Le MCP est en lecture seule — **ce skill n'envoie JAMAIS de message**, il prépare un
texte à copier-coller. Dis-le clairement à l'utilisateur : il reste maître de l'envoi.

## Règle n°0 — la langue du message = la langue du client

C'est la règle la plus importante (alignée sur le bot WhatsApp DGSoftware). Avant de rédiger, regarde
les messages du client et réponds **dans sa langue** :
- Lettres arabes (سلام، شحال، نحب) → **darija algérienne en lettres arabes**.
- Darija en lettres latines (« slm », « wesh », « rak », « chhal », « bghit », « 3andi », « kifech »)
  → **darija en lettres latines** (miroir de son écriture).
- Français correct → **français**.
- Ambigu (salutation seule, un mot) → **darija algérienne par défaut** (clientèle majoritairement
  arabophone). Ne jamais répondre en arabe classique/fus'ha : uniquement la darija parlée d'Algérie.

Ne réponds jamais en français à un client qui écrit en arabe/darija, ni l'inverse.

## Déroulé

1. **Identifier les destinataires.** Soit l'utilisateur donne une liste / un client précis, soit il
   demande « les relances du jour » → t'appuyer sur les prospects à recontacter (états `PC-Rappel`,
   `SAD-Rappel`, `Premiere Entree`, `Intéressé`). Pour chacun, récupère sa conversation WhatsApp
   (`dgstock_fetch_whatsapp_messages`) pour t'imprégner du contexte réel.

2. **Comprendre le contexte de chaque client avant d'écrire** : son activité/secteur, le produit visé
   (déduit en rapprochant le montant des tarifs réels du compte, lus via le MCP — ne code aucun prix en
   dur), l'étape (premier contact vs après démo), la dernière chose dite, les objections déjà
   exprimées (abonnement vs définitif, prix, hors-ligne), et toute offre/remise déjà proposée.

3. **Rédiger un message personnalisé** qui :
   - reprend le fil de la dernière conversation (jamais un message générique « bonjour, vous êtes toujours intéressé ? ») ;
   - répond à l'objection ou la question laissée en suspens si elle existe ;
   - rappelle la valeur concrète pour SON activité (pas un argumentaire catalogue) ;
   - propose **une seule action claire** (rappel, démo, confirmation de commande, lien de paiement) ;
   - reste court, chaleureux, professionnel — ton WhatsApp algérien, pas un e-mail formel ;
   - n'invente aucun prix, fonctionnalité ou promesse non confirmés par les données / le playbook.

4. **Adapter le message au niveau de chaleur** :
   - Lead brûlant (prêt à payer) → message de **closing** : lever le dernier frein + faciliter le paiement.
   - Chaud après démo (`SAD-*`) → message de **relance d'engagement** : « on s'était dit… ».
   - Premier contact / tiède → message de **(re)qualification** : une question ouverte pour rouvrir le dialogue.
   - Dormant (ancien) → message de **réveil** : court, sans reproche, avec une raison de répondre maintenant (nouveauté, offre).

## Format de sortie

Pour chaque destinataire, présente un bloc clair que le commercial copie tel quel :

```markdown
### [Nom client] — [Téléphone] — [Produit / montant] — [langue détectée]
**Contexte** : [1 ligne : où on en est avec ce client]
**Message à envoyer :**
> [le texte exact, prêt à copier-coller, dans la langue du client]

**Variante (optionnelle)** : [une 2e formulation si l'utilisateur veut le choix]
```

Si plusieurs clients, regroupe-les et propose à la fin de générer aussi une version « relance courte »
ou « relance avec offre » selon ce que l'utilisateur préfère.

## Garde-fous

- **Vérité commerciale** : ne promets pas une remise, un prix ou une fonctionnalité que les données ou
  le playbook ne confirment pas. En cas de doute sur un prix, écris un message qui propose le rappel du
  commercial pour le chiffrage, plutôt qu'un montant inventé.
- **Pas de spam** : un message par client, ciblé. Si un client a clairement dit « pas intéressé » ou
  « faux numéro », ne rédige pas de relance sauf demande explicite de campagne de réactivation.
- **Validation humaine** : rappelle que les messages sont des propositions à relire avant envoi.
- `dgstock_generate_pdf` est en panne : si export demandé, produire HTML/Markdown.
