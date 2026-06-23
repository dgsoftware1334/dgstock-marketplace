# DGStock — Marketplace de plugins commerciaux

Marketplace interne **DGSoftware** pour Claude Code / Cowork. Contient le plugin **`dgstock-sales`** :
16 skills d'aide à la vente et au pilotage commercial s'appuyant sur le CRM DGStock / DG-SALE
(données en **lecture seule** via le MCP DGStock — les skills préparent l'action, l'humain l'exécute).

> ⚠️ **Dépôt à garder privé** : il contient des éléments commerciaux internes (grille tarifaire,
> valeurs de pipeline, stratégie du bot WhatsApp, exemples clients).

## Installation

Dans un terminal Claude Code / Cowork :

```
/plugin marketplace add <owner>/<repo>
/plugin install dgstock-sales@dgstock
```

Les 16 commandes `/dgstock-…` deviennent alors disponibles.

## Les 16 skills

**🚀 Accélération quotidienne**
- `/dgstock-leads-chauds-maintenant` — prospects à forte intention des dernières 24-72h à rappeler vite
- `/dgstock-messages-relance-prets` — messages WhatsApp de relance prêts à copier-coller (darija/français)
- `/dgstock-fiche-preparation-appel` — fiche client 360° avant un appel
- `/dgstock-briefing-quotidien` — tour d'horizon du matin

**💰 Chiffre d'affaires**
- `/dgstock-reactivation-pipeline-dormant` — campagne de récupération des gros dossiers jamais relancés
- `/dgstock-confirmation-prestations` — ventes à confirmer + relance des impayés
- `/dgstock-suivi-prospections` — qui rappeler aujourd'hui (pipeline priorisé)
- `/dgstock-priorite-contact-clients` — contacts de la semaine (WhatsApp + avis + réclamations)

**📊 Pilotage & stratégie**
- `/dgstock-plan-action-commercial` — plan d'action hebdomadaire complet
- `/dgstock-prevision-ventes` — forecast pondéré par étape et produit
- `/dgstock-conversion-prospection-prestation` — funnel de conversion
- `/dgstock-rapport-hebdomadaire-pdf` — bilan hebdomadaire consolidé
- `/dgstock-mine-objections` — objections récurrentes et parades
- `/dgstock-performance-bot-whatsapp` — audit du bot WhatsApp

**🧹 Qualité & fidélisation**
- `/dgstock-hygiene-crm` — doublons, fiches incomplètes, prospects à nettoyer
- `/dgstock-alerte-satisfaction` — réclamations non traitées + avis négatifs

## Catalogue imprimable

Voir [`plugins/dgstock-sales/CATALOGUE.html`](plugins/dgstock-sales/CATALOGUE.html) (ouvrir dans un navigateur, bouton « Imprimer / PDF »).

## Structure

```
dgstock-marketplace/
├── .claude-plugin/marketplace.json
└── plugins/dgstock-sales/
    ├── .claude-plugin/plugin.json
    ├── CATALOGUE.html
    └── skills/   → 16 skills dgstock-…
```
