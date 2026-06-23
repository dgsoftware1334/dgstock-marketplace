---
name: dgstock-fiche-preparation-appel
description: Prépare une fiche complète sur UN client/prospect DGStock avant de l'appeler ou de le rencontrer — vue 360° : qui il est, son historique, sa conversation WhatsApp, son produit/montant, ses objections, et l'angle conseillé pour l'appel. Utilise OBLIGATOIREMENT ce skill quand l'utilisateur demande "prépare-moi l'appel avec [client]", "fiche client", "tout sur ce client avant de l'appeler", "brief avant rendez-vous", "résume l'historique de [client]", ou toute variante de préparation d'un contact précis. Déclenche aussi sur "/dgstock-fiche-preparation-appel" ou "/dgstock-fiche-appel".
---

# Fiche de préparation d'appel (Client 360°)

## Ce que fait ce skill

Avant d'appeler un client, un bon commercial veut tout savoir en 30 secondes : qui est cette personne,
ce qu'on lui a déjà dit, ce qu'elle veut, ce qui la bloque, et par quoi commencer l'appel. Ce skill
rassemble tout ça sur **un seul client** en une fiche dense et actionnable. C'est l'inverse des skills
de liste : ici on va en profondeur sur une personne.

## Lecture seule

Le MCP est en lecture seule : la fiche prépare l'appel, elle ne contacte pas le client et ne modifie
rien.

## Déroulé

1. **Identifier le client.** L'utilisateur donne un nom, un numéro ou une référence de prospection.
   Recherche-le : `dgstock_fetch_clients` (search) et/ou `dgstock_query_data` sur `prospections`
   (search). S'il y a plusieurs homonymes, liste-les et demande lequel (ou prends le plus pertinent en
   l'indiquant).

2. **Rassembler tout ce qui le concerne :**
   - **Identité** : nom, téléphone, (e-mail/adresse si présents — souvent vides).
   - **Pipeline** : sa/ses prospection(s) — état, montant `totalTTC` → produit déduit (10 500 = DGStock ;
     60 000 = DGProduction ; 45-53K = Entreprise ; 15-17K = Distributeur ; 0 = non qualifié), date de
     création, ancienneté.
   - **Ventes** : prestations éventuelles (confirmée ? impayé `restant` > 0 ?) → un client avec un
     impayé en cours change l'angle de l'appel.
   - **Conversation WhatsApp** : `dgstock_fetch_whatsapp_messages` — lis tout l'historique. Extrais :
     son secteur/activité, sa taille (employés, points de vente), ce qu'il a demandé, les objections
     exprimées, les engagements pris (par lui ou par le bot/commercial), une offre/remise déjà proposée,
     et la dernière chose dite + qui doit jouer le prochain coup.
   - **Satisfaction** : réclamations ou avis liés à ce client, le cas échéant.

3. **Synthétiser en intelligence d'appel** : ne te contente pas de juxtaposer les données — déduis
   l'**état réel de la relation** (où on en est, chaud/tiède/froid, pourquoi), le **prochain pas
   logique**, l'**objection à anticiper** et la **meilleure accroche** pour démarrer l'appel.

## Format de sortie

```markdown
# Fiche d'appel — [Nom client]
☎️ [Téléphone] · 🏷️ [Produit] · 💰 [Montant] · 📊 [État pipeline] · ⏱️ [ancienneté]

## En une phrase
[Où on en est avec ce client et ce qu'il faut obtenir de cet appel.]

## Profil & activité
[Secteur, taille, contexte business tiré du WhatsApp.]

## Historique de la relation
- [Chronologie courte : premier contact → démo → offre → silence, etc.]
- Dernier échange : [date] — [qui doit rappeler / ce qui était convenu]

## Ce qu'il veut / ses objections
- Besoin : ...
- Objection(s) à anticiper : ... (+ parade recommandée)

## Situation financière (si pertinent)
- Impayé en cours : oui/non — [montant] → [impact sur l'appel]

## Plan d'appel
1. Accroche : "[phrase d'ouverture concrète, dans sa langue si utile]"
2. Objectif : [conclure / démo / lever objection / chiffrer]
3. Prochaine action attendue : ...
```

Si l'utilisateur le souhaite, propose d'enchaîner avec `dgstock-messages-relance-prets` (pour préparer aussi
le message WhatsApp de suivi après l'appel).

## Garde-fous

- N'invente rien : si une info manque (téléphone, secteur), écris-le (« non renseigné ») plutôt que de
  supposer. Si aucune conversation WhatsApp n'existe, base-toi sur le pipeline et dis-le.
- Reste synthétique : la fiche doit se lire avant un appel, pas être un dossier de 3 pages.
- Résume le WhatsApp en français ; cite une phrase clé du client en VO seulement si c'est utile à l'appel.
- `dgstock_generate_pdf` en panne : produire HTML/Markdown si export demandé.
