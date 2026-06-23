---
name: dgstock-performance-bot-whatsapp
description: Audite la performance du bot WhatsApp DGSoftware — détecte les rappels promis non tenus, les leads chauds perdus faute de relais humain, et les faux prospects générés par le bot qui polluent le pipeline et faussent les statistiques. Utilise OBLIGATOIREMENT ce skill quand l'utilisateur demande "performance du bot", "le bot fait-il bien son travail", "leads perdus par le bot", "faux prospects", "qualité des prospections créées", "le bot promet-il des rappels non tenus", ou toute variante d'audit du bot/du tunnel WhatsApp→CRM. Déclenche aussi sur "/dgstock-performance-bot-whatsapp" ou "/dgstock-audit-bot".
---

# Performance du bot WhatsApp (audit du tunnel WhatsApp → CRM)

## Pourquoi c'est stratégique

Chez DGSoftware, le bot WhatsApp qualifie les clients et le backend les enregistre comme prospections.
Le bot est une force (il traite un gros volume 24/7), mais il a trois failles coûteuses que ce skill
met en lumière :
1. **Promesses non tenues** : le bot dit « notre commercial va vous rappeler » / « العشية » mais aucun
   rappel humain ne suit → le client se sent ignoré, la vente se perd.
2. **Leads chauds perdus** : un client exprime une intention d'achat forte, mais le relais humain n'a
   pas pris le relais à temps (« le bot qualifie, l'humain conclut » — ici l'humain a manqué).
3. **Bruit dans le pipeline** : le bot crée des prospections pour des contacts qui ne sont pas des
   clients (ex. observé : une personne cherchant un emploi, une autre voulant acheter un landau, des
   prospects créés à 0 DA). Ce bruit gonfle artificiellement le pipeline, fausse le taux de conversion
   et fait perdre du temps aux commerciaux.

Ce skill aide à **régler le bot et le process**, pas à contacter des clients.

## Lecture seule

Le MCP est en lecture seule : ce skill diagnostique, il ne modifie pas le bot ni les statuts. Les
réglages (script, règles d'escalade) sont décidés par l'humain. Le prompt/réglages du bot sont
consultables via `dgstock_fetch_whatsapp_config` (utile pour comprendre ce que le bot est censé faire).

## Déroulé

1. **Comprendre le comportement attendu du bot** : lis `dgstock_fetch_whatsapp_config`
   (`agent_instructions_md`) pour savoir ce que le bot promet (marqueurs type rappel commercial,
   escalade humaine) et comparer avec ce qui se passe réellement.

2. **Échantillonner les conversations récentes** : `dgstock_fetch_whatsapp_conversations` +
   `dgstock_fetch_whatsapp_messages`. Pour chacune, analyse :
   - **Promesse de rappel** : le bot a-t-il annoncé un rappel humain ? A-t-il eu lieu (un message
     `outbound` avec `sent_by_user_id` = humain, ou une suite cohérente) ? Sinon → promesse non tenue.
   - **Lead chaud non reprisé** : intention d'achat exprimée (paiement, accord, demande de facture)
     restée sans relais humain → lead chaud perdu.
   - **Qualité du prospect** : la conversation montre-t-elle un vrai besoin business, ou est-ce du
     bruit (cherche un emploi, hors-sujet, mineur, particulier sans activité) ?

3. **Croiser avec le pipeline** : compte les prospections créées à **0 DA** (souvent non qualifiées) et
   repère les noms manifestement non commerciaux. Mesure la part de bruit dans les prospections
   récentes.

4. **Quantifier** chaque faille (taux, volumes) sur l'échantillon et **identifier les cas concrets** à
   corriger tout de suite (les leads chauds perdus = à rappeler aujourd'hui ; renvoyer vers
   `dgstock-leads-chauds-maintenant`).

## Format de sortie

```markdown
# Audit du bot WhatsApp — [période] (échantillon de X conversations)

## Synthèse
- Rappels promis non tenus : X (= X% des conversations où un rappel a été promis)
- Leads chauds perdus (intention forte sans relais humain) : X
- Bruit pipeline : X prospections à 0 DA / non commerciales sur la période (= X%)

## 🔴 Leads chauds perdus — à récupérer maintenant
| Client | Téléphone | Ce qu'il avait dit | Date | Action |
|---|---|---|---|---|

## Rappels promis non tenus
| Client | Promesse du bot | Date promise | Statut |
|---|---|---|---|

## Bruit détecté (à exclure / disqualifier)
| Prospect | Pourquoi ce n'est pas un vrai client |
|---|---|

## Recommandations pour le bot et le process
- [Ex. : alerte humaine immédiate quand un client demande à parler à quelqu'un]
- [Ex. : ne créer une prospection chiffrée que si le besoin business est confirmé]
- [Ex. : règle de relais humain sous X h après une intention d'achat]
```

## Garde-fous

- Le bot reste un atout : présente l'audit comme une optimisation, pas un réquisitoire — pondère les
  failles par le volume traité.
- Sépare clairement ce qui relève du **bot** (script) et du **process humain** (relais, attribution).
- Échantillon honnête : indique sa taille et n'extrapole pas au-delà du raisonnable.
- Résume en français ; `dgstock_generate_pdf` en panne → HTML/Markdown si export demandé.
