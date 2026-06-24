---
name: dgstock-leads-chauds-maintenant
description: Repère en priorité absolue les prospects DGStock à forte intention d'achat des dernières 24-72h qui attendent un contact humain — pour les rappeler AVANT qu'ils refroidissent. Utilise OBLIGATOIREMENT ce skill quand l'utilisateur demande "les leads chauds", "qui veut acheter maintenant", "qui attend un rappel urgent", "les prospects prêts à payer", "qui rappeler en urgence aujourd'hui", ou toute variante d'urgence commerciale temps réel. Déclenche aussi sur "/dgstock-leads-chauds-maintenant" ou "/dgstock-leads-chauds".
---

# Leads chauds maintenant (détection temps réel)

## Pourquoi ce skill est le plus rentable pour aller vite

Chez DGSoftware, un bot WhatsApp qualifie les clients puis le backend crée une prospection ; un
commercial humain doit ensuite appeler pour conclure. **Le délai entre « le client est chaud » et
« un humain le rappelle » est là où se perdent les ventes.** Ce skill réduit ce délai : il sort, à un
instant T, la liste courte des prospects qui viennent d'exprimer une intention forte et attendent un
humain. Ce n'est PAS la liste hebdo (voir `dgstock-priorite-contact-clients`) ni le pipeline complet (voir
`dgstock-suivi-prospections`) — c'est l'urgence du jour, à traiter dans l'heure.

## Lecture seule — ce skill ne contacte personne

Le MCP DGStock est en lecture seule. Tu produis une liste d'appels à passer ; l'humain appelle.
Ne prétends jamais avoir contacté ou répondu à quelqu'un.

## Signaux de « lead chaud » (à détecter dans les conversations WhatsApp)

Lis le contenu des messages (souvent en darija latine/arabe ou français). Un lead est CHAUD s'il
présente au moins un de ces signaux **récents (dernières 24-72h)** :

- **Intention de paiement explicite** : mentionne BaridiMob, « نخلص », « je paie », accepte une offre/remise, demande comment payer.
- **Demande d'humain** : veut parler à un commercial, dit que le bot ne suffit pas → escalade immédiate, c'est le signal le plus fort.
- **Accord verbal** : « ok », « d'accord », « واخا », « نشري », « je prends ».
- **Le bot a promis un rappel humain** (« notre commercial va vous appeler », « العشية ») et il n'a pas encore eu lieu → engagement à honorer aujourd'hui.
- **Demande de facture / devis pour acheter** (pas juste le prix par curiosité).
- **A reçu une offre chiffrée** (souvent par un humain : message avec `sent_by_user_id`) et n'a pas encore répondu/conclu.

Refroidisseurs (à signaler mais priorité moindre) : dernier échange > 3 jours, objection bloquante
non levée (prix, hors-ligne, abonnement vs définitif).

## Déroulé

1. **Lister les conversations récentes** : `dgstock_fetch_whatsapp_conversations` (limit élevé, regarde `last_message_at` — concentre-toi sur les dernières 24-72h). Le champ `is_read`/`status` est peu fiable (souvent null) — ne t'y fie pas.
2. **Ouvrir les conversations actives** : pour chaque conversation récente avec un `messages_count` significatif, `dgstock_fetch_whatsapp_messages`. Repère si le **dernier message est `inbound`** (le client attend) et lis les derniers échanges pour détecter les signaux ci-dessus.
3. **Croiser avec la prospection** : recherche le prospect dans `prospections` (par nom/numéro) pour récupérer l'état et le montant (`totalTTC`). Déduis le produit en rapprochant ce montant des tarifs réels du compte (lis-les via le MCP, voir le playbook — ne code aucun prix en dur) ; un montant à 0 = prospect non qualifié. Un `SAD-Rappel`/`Intéressé` chaud passe avant un `PC-Rappel` chaud.
4. **Classer** : 🔴 BRÛLANT (intention de paiement / demande d'humain / offre chiffrée en attente) puis 🟠 CHAUD (accord verbal, questions d'achat précises). Toujours écrire **la phrase de situation en clair** (« prêt à payer par BaridiMob, attend confirmation », pas « score élevé »).

## Format de sortie (court, actionnable)

```markdown
# 🔥 Leads chauds — [date et heure]

## 🔴 BRÛLANT — à rappeler dans l'heure
| Client | Téléphone | Produit | Montant | Ce qu'il a dit / attend | Action |
|---|---|---|---|---|---|

## 🟠 CHAUD — à rappeler aujourd'hui
| Client | Téléphone | Produit | Montant | Situation | Action |
|---|---|---|---|---|---|

## En bref
- X leads brûlants, X chauds — valeur cumulée estimée : X DZD
- Engagements de rappel promis par le bot non encore honorés : X
```

Garde la liste courte (les vrais chauds, pas tout le pipeline). Si rien n'est brûlant, dis-le
clairement plutôt que de remplir artificiellement.

## Rappels techniques (MCP DGStock)

- L'opérateur `in` ne marche pas sur `etat` ; `=` sur valeurs accentuées peut échouer (utiliser `like`).
- Téléphone parfois absent de la relation `client` → chercher via `dgstock_fetch_clients` (search par nom), sinon « → à vérifier ».
- Résume l'intention en français, ne colle pas le texte arabe brut.
- `dgstock_generate_pdf` est en panne : si un export est demandé, produire un HTML/Markdown.
