---
name: dgstock-briefing-quotidien
description: Génère le briefing du jour pour celui qui l'exécute — tour d'horizon combinant le pipeline de prospections à rappeler, les ventes/impayés à traiter, et les clients à contacter en priorité (WhatsApp, avis, réclamations). Utilise OBLIGATOIREMENT ce skill quand l'utilisateur demande "mon briefing du jour", "qu'est-ce que je dois faire aujourd'hui", "fais-moi un résumé de la journée", "par quoi je commence", ou toute variante de planification de journée commerciale DGStock. Déclenche aussi sur "/dgstock-briefing-quotidien" ou "/dgstock-briefing".
---

# Briefing quotidien (DGStock)

## Ce que ce skill fait

C'est un skill **orchestrateur** : il exécute la logique des trois skills suivants et en condense le résultat en une seule page d'action pour la journée, plutôt que d'obliger l'utilisateur à lancer trois rapports séparés :

1. `dgstock-suivi-prospections` → qui rappeler dans le pipeline (PC-Rappel / SAD-Rappel / nouveaux leads sans suite)
2. `dgstock-confirmation-prestations` → ventes à confirmer + impayés à relancer
3. `dgstock-priorite-contact-clients` → clients à contacter d'après WhatsApp/avis/réclamations

Lis ces trois skills (ou applique directement leur logique si déjà en mémoire dans la conversation) puis condense — ne réexplique pas toute leur méthodologie ici, va droit au résultat actionnable.

## Comment exécuter ce skill

1. Exécute en parallèle (ou séquentiellement si les outils ne permettent pas le parallélisme) les requêtes clés de chacun des trois skills, en te limitant aux éléments **du jour** : pas besoin de recalculer l'historique complet à chaque fois si l'utilisateur veut juste "sa journée" — concentre-toi sur :
   - Les `PC-Rappel`/`SAD-Rappel` les plus anciens (top 5-10, pas la liste exhaustive de plusieurs centaines).
   - Les impayés les plus importants (top 5) et les ventes à confirmer en attente.
   - Les signaux WhatsApp/réclamations les plus urgents du jour (engagements non tenus, réclamations anciennes, messages en attente de réponse).

2. **Ne duplique pas un client qui apparaît dans plusieurs sections** — si un client a un impayé ET un message WhatsApp en attente, mentionne-le une seule fois avec les deux signaux combinés, en haut du briefing (c'est le profil le plus prioritaire qui existe : valeur + urgence relationnelle).

3. Produit ce format condensé :

```markdown
# Briefing du jour — [date]

## 🔴 Top priorités (signaux combinés)
[clients qui cumulent plusieurs urgences — impayé + réclamation, rappel ancien + message en attente, etc.]

## 📞 Pipeline — à rappeler aujourd'hui
[top 5-10 PC-Rappel/SAD-Rappel les plus anciens ou les plus gros montants]

## 💰 Ventes & paiements
[ventes à confirmer + top 5 impayés]

## 💬 Clients à contacter (WhatsApp/réclamations/avis)
[top signaux du jour, voir priorite-contact-clients]

## En un coup d'œil
- X dossiers à traiter aujourd'hui
- X DZD d'impayés en jeu sur le top 5
- X engagements de rappel non tenus détectés
```

4. Termine en proposant : "Voulez-vous le détail complet d'une de ces sections (liste intégrale, export PDF) ?" — le briefing quotidien doit rester court ; les rapports complets existent déjà dans les autres skills.

## Pourquoi rester court

Ce skill est fait pour être lu en 2 minutes avant de commencer la journée. Si tu listes tout ce que les autres skills peuvent produire, tu recrées un rapport de 5 pages que personne ne lira chaque matin — sélectionne, ne reproduis pas tout.
