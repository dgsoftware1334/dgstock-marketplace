---
name: dgstock-rapport-hebdomadaire-pdf
description: Génère le rapport hebdomadaire consolidé de l'activité commerciale DGStock (pipeline, chiffre d'affaires, impayés, satisfaction client) et le produit en PDF partageable. Utilise OBLIGATOIREMENT ce skill quand l'utilisateur demande "le rapport de la semaine", "fais-moi un PDF du bilan commercial", "résumé hebdo pour la direction", "bilan de la semaine", ou toute variante de reporting hebdomadaire DGStock. Déclenche aussi sur "/dgstock-rapport-hebdomadaire-pdf" ou "/dgstock-rapport-hebdo".
---

# Rapport hebdomadaire consolidé (PDF)

## Ce que ce skill fait

Contrairement à `dgstock-briefing-quotidien` (court, pour l'action immédiate), ce skill produit un **document de bilan** destiné à être lu une fois par semaine, partagé et archivé (export PDF via `dgstock_generate_pdf`). Il doit donner une vision chiffrée, pas une liste d'actions.

## Comment exécuter ce skill

1. **Pipeline (prospections)** — sur la semaine écoulée (`date_from`/`date_to`) :
   - Nombre de nouvelles prospections créées.
   - Répartition par grande famille d'état (Premier Contact / Suivi Après Démo / positif / négatif) — voir `dgstock-suivi-prospections` pour le détail des codes `PC-*`/`SAD-*`.
   - Volume cumulé des dossiers `*-Rappel` en attente (tout l'historique, pas juste la semaine — c'est un stock, pas un flux) pour montrer si le backlog grossit ou se résorbe par rapport au rapport précédent si disponible.

2. **Ventes & finance (prestations)** :
   - CA facturé sur la semaine (`sum totalTTC` filtré sur la période).
   - Total impayé cumulé (`sum restant`, sans filtre de date — c'est un encours, pas un flux hebdo) et part du CA facturé total qu'il représente.
   - Nombre de ventes en attente de confirmation (`etat = "Premiere Entree"`).
   - **Si une agrégation `sum` sur un champ stocké en texte renvoie 0 de façon suspecte, signale l'anomalie technique plutôt que de présenter "0 DZD" comme un vrai résultat.**

3. **Qualité de service** :
   - Réclamations non traitées (toutes anciennetés confondues), en signalant la plus ancienne et les clients récurrents.
   - Note moyenne des avis clients, **avec la réserve que cette note seule ne reflète pas toujours un vrai niveau de satisfaction sur ce compte** (beaucoup de notes basses sont en réalité des notes d'appel "pas de réponse"/"à rappeler", pas une insatisfaction) — lis quelques commentaires récents pour qualifier le chiffre plutôt que de le donner brut.

4. **Construire le contenu Markdown complet**, puis appeler `dgstock_generate_pdf` avec :
   - `title`: "Rapport hebdomadaire — semaine du [date début] au [date fin]"
   - `content`: l'intégralité du rapport en Markdown (tableaux, sections), structuré ainsi :

```markdown
## Pipeline commercial
[tableau répartition par état + total]

## Ventes & finance
[CA, impayés, % impayé, ventes en attente de confirmation]

## Qualité de service
[réclamations non traitées, satisfaction]

## Points d'attention de la semaine
[2-4 points concrets qui ressortent vraiment des chiffres, pas une liste générique]
```
   - `summary`: 3-4 phrases de synthèse pour quelqu'un qui n'a pas le temps de lire tout le document.

5. **Confirme à l'utilisateur le lien/référence du PDF généré** et propose de l'envoyer ou de l'archiver (selon les outils disponibles à ce moment — si aucun outil d'envoi n'existe, dis-le simplement).

   **⚠️ Panne connue (testée le 23/06/2026) : `dgstock_generate_pdf` renvoie systématiquement l'erreur "Outil inconnu: generate_pdf"**, quel que soit le contenu envoyé — c'est un problème côté connecteur DGStock, pas quelque chose que ce skill peut corriger. Si l'appel échoue avec cette erreur (ou toute erreur similaire), **ne bloque pas** : présente le rapport complet en Markdown directement dans la réponse à l'utilisateur, signale que l'export PDF est indisponible pour le moment, et suggère de réessayer plus tard ou de copier le Markdown manuellement. Si un jour l'outil fonctionne à nouveau, utilise-le normalement.

## Pièges à éviter

- Ne mélange pas des indicateurs de **stock** (impayés cumulés, backlog de rappels) avec des indicateurs de **flux** (nouvelles ventes de la semaine) sans le préciser clairement — ce sont deux natures de chiffres différentes et les confondre rend le rapport trompeur.
- Ne génère le PDF qu'une fois que tu as toutes les données en main — `dgstock_generate_pdf` n'est pas fait pour être appelé plusieurs fois en brouillon, structure le contenu avant l'appel.
