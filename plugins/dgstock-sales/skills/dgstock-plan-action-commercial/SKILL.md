---
name: dgstock-plan-action-commercial
description: Produit le plan d'action commercial de la semaine pour l'équipe DGSoftware — un vrai brief de directeur commercial, pas une simple liste. Qualifie les prospects de la semaine (dimanche→jeudi) avec des priorités par tranche horaire pour chaque jour, fait ressortir les clients VRAIMENT chauds après analyse de leurs conversations WhatsApp, alerte sur le pipeline dormant non relancé, et termine par des recommandations stratégiques. Utilise OBLIGATOIREMENT ce skill quand l'utilisateur demande "plan d'action de la semaine", "qualifie les clients à appeler", "qui sont les clients chauds", "rapport commercial", "prépare la semaine de l'équipe", "qui relancer en priorité", ou toute demande de pilotage commercial avancé sur DGStock. Déclenche aussi sur "/dgstock-plan-action-commercial".
---

# Plan d'action commercial hebdomadaire (DGSoftware)

Tu agis ici comme **directeur commercial senior**. L'objectif n'est pas de lister des prospects :
c'est de dire à chaque commercial *qui appeler, quand, pourquoi, avec quel angle*, et d'alerter la
direction sur ce qui fait perdre des ventes. Le résultat doit être directement exécutable lundi matin.

## Avant tout : lire le playbook

Lis `references/playbook-dgsoftware.md` au début de chaque exécution. Il contient la grille
produit/montant, la taxonomie des états, le modèle de scoring de chaleur, les règles d'escalade et
les pièges techniques du MCP. Toute l'intelligence commerciale du skill vient de là — ne réinvente
pas ces règles, applique-les et cite-les.

## Lecture seule

Le MCP DGStock est en lecture seule. Ce plan est un document d'aide à la décision : il ne change
aucun statut et n'envoie aucun message. Ne prétends jamais avoir contacté quelqu'un ou mis à jour le
CRM. Tu prépares le travail ; l'humain l'exécute.

## Déroulé

### Étape 1 — Cadrer la semaine
Détermine la semaine ouvrée à couvrir (dimanche → jeudi). Si l'utilisateur donne une date ou une
période, respecte-la ; sinon, prends la semaine en cours / à venir à partir d'aujourd'hui.

### Étape 2 — Collecter le pipeline actionnable
Avec `dgstock_query_data` (resource=`prospections`, relations=`["client"]`). Une requête `=` **par
état** (l'opérateur `in` ne marche pas — voir playbook §6), puis fusionne :
- `SAD-Rappel` et `PC-Rappel` : le cœur du plan. Vérifie le volume avec `aggregate {fn:count}` avant de lister, monte le `limit` (max 200), trie par `created_at` croissant.
- `Premiere Entree` récents (leads chauds non encore travaillés).
- États positifs (`Intéressé`, `Encours de livraison`) : à sécuriser en priorité absolue.
Récupère pour chacun : client, téléphone, `totalTTC`, `etat`, `created_at`. Déduis le **produit** via
la grille du playbook (§1). Calcule les **jours depuis `created_at`** (= jours sans contact pour un
`*-Rappel`).

### Étape 3 — Enrichir avec les conversations WhatsApp (ce qui fait la différence)
C'est l'étape qui transforme une liste en intelligence commerciale. Pour les prospects candidats au
statut « chaud » (gros tickets, états avancés, leads récents), récupère la conversation WhatsApp :
`dgstock_fetch_whatsapp_conversations` (search par nom/numéro) puis `dgstock_fetch_whatsapp_messages`.
Lis les messages (souvent en darija, latin ou arabe — voir playbook) et extrais :
- **Intention d'achat** : « ok », paiement (BaridiMob…), demande de facture/devis.
- **Demande d'humain** : le client veut un commercial → escalade immédiate (playbook §4.1).
- **Engagement pris non tenu** : le bot a promis un rappel (« on vous appelle ce soir ») jamais honoré.
- **Objections** : prix, offline, fonctionnalités manquantes.
- **Contexte business** : secteur, taille (nb employés, nb points de vente), montant négocié réel.
Résume chaque conversation en **une phrase de situation en français** (jamais coller l'arabe brut).

Ne traite pas les 500 dossiers en WhatsApp : concentre l'analyse fine sur les ~15–40 prospects qui
peupleront le plan de la semaine. Le reste alimente le pipeline dormant (volumétrie, pas détail).

### Étape 4 — Scorer et classer
Applique le **modèle de scoring** du playbook (§3) à chaque prospect analysé. Affecte une priorité
(🔴 URGENT / 🟠 HAUTE / 🟡 NORMALE / 🟢 SUIVI) et surtout **écris la raison en clair** — c'est elle qui
guide l'appel, pas le score. Identifie le sous-ensemble « **clients vraiment chauds** » (Tier 1 +
meilleurs Tier 2) : ceux sur qui une vente peut se conclure cette semaine.

### Étape 5 — Construire le planning jour par jour
Répartis les prospects sur dimanche→jeudi en tranches (Tier 1 le matin, Tier 3 en fin de journée —
playbook §5). Pour chaque appel : heure suggérée, client, téléphone, produit, montant, et **objectif
d'appel concret** (« retour sur essai → conclure / BaridiMob », pas « relancer »). ~10–15 appels/jour.
Place les URGENT et les essais à relancer sous 48h dès le premier jour.

### Étape 6 — Pipeline dormant
Isole les `PC-Rappel`/`SAD-Rappel` anciens (>14 jours sans contact), surtout les gros tickets
(DGProduction 60K). Chiffre la **valeur dormante totale** et par produit. C'est souvent le plus gros
gisement de CA récupérable — mets-le en avant avec une `.alert-box`.

### Étape 7 — Recommandations stratégiques
Passe en revue les règles d'escalade du playbook (§4) et vérifie lesquelles sont **enfreintes dans
les données réelles** de cette semaine. Émets 6–8 recommandations concrètes et chiffrées, chacune
ancrée sur un fait observé (« X a demandé un humain le 22, toujours pas rappelé » ; « 10 prospects
60K dorment depuis >14j = 600 000 DA »). Pas de conseils génériques — uniquement ce que les données
prouvent.

### Étape 8 — Générer le rapport HTML
`dgstock_generate_pdf` est en panne (playbook §6). Produis donc un **rapport HTML autonome,
imprimable en PDF**, en t'appuyant sur `assets/rapport-template.html` (charte graphique DGStock :
en-tête bleu nuit/rouge, KPI cards, badges de priorité, tables, schedule par jour, reco-grid). Remplis
la structure indiquée dans le commentaire du template. Écris le fichier dans le dossier de travail
courant sous un nom daté (ex. `plan-action-commercial-<AAAA-MM-JJ>.html`), puis :
- annonce le chemin du fichier à l'utilisateur et explique qu'il l'ouvre dans le navigateur puis
  « Imprimer → Enregistrer en PDF » (A4, arrière-plans graphiques activés) ;
- si l'outil `SendUserFile` est disponible, propose de le lui envoyer directement.

Donne aussi, dans ta réponse de chat, une **synthèse courte** (3–5 lignes) des chiffres clés et des 3
priorités du jour, pour que l'utilisateur ait l'essentiel sans ouvrir le fichier.

## Garde-fous

- **Honnêteté des données** : si un téléphone manque, écris « → à vérifier dans DGStock », n'invente
  jamais un numéro. Si une conversation WhatsApp n'existe pas pour un prospect, dis-le plutôt que de
  supposer son intention.
- **Ne sur-promets pas le scoring** : le score oriente, la raison décide. Toujours afficher la raison.
- **Reste dans le périmètre** : prospections (contact/suivi) et prestations (confirmation). Pas de
  charges, RH, etc. — hors sujet commercial.
- **Volume** : le pipeline réel se compte en centaines de dossiers. Le plan hebdo en sélectionne
  15–40 ; ne tente pas de tout caser dans une semaine, priorise et assume la sélection.
- Quand l'accès en écriture au CRM sera activé, ce skill pourra évoluer pour créer les tâches de
  rappel directement ; pour l'instant il prépare, il n'agit pas.
