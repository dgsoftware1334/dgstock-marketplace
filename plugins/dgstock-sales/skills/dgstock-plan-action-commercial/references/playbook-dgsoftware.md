# Playbook commercial DGSoftware — référence interne

Ce document encode la connaissance métier DGSoftware utilisée par le skill `plan-action-commercial`
(et réutilisable par les autres skills CRM). Il transforme des données brutes du CRM en décisions
de vente. Mets-le à jour quand la grille tarifaire, la gamme produit ou la méthode de vente changent.

---

## 1. Gamme produit & cartographie par montant

Le montant `totalTTC` d'une prospection révèle quel produit est vendu. Sur les données de juin 2026,
deux SKU dominent (135 prospects à 10 500 DA, 132 à 60 000 DA). Grille validée :

| Montant `totalTTC` | Produit déduit | Cible type | Lecture commerciale |
|---|---|---|---|
| **10 500 DA** (ou 10 200 promo) | **DGStock** (gestion stock/caisse de base) | Commerce, boutique, petite activité | Ticket d'entrée, cycle court, décision rapide |
| **60 000 / 63 000 DA** | **DGProduction** (gestion de production) | Usine, atelier, fabrication, conditionnement | Gros ticket, cycle plus long, décideur = patron |
| **45 000–53 550 DA** | **DGStock ENTREPRISE** (multi-points, app mobile) | Multi-boutiques, PME structurée | Souvent issu d'une négociation, valeur élevée |
| **15 000–17 000 DA** | **DGStock Distributeur** (gestion tournées/distribution) | Distributeur, grossiste, livraison | Argument clé : factures de route, suivi livreurs |
| **250 000 DA** ou sur devis | **DGProject** (BTP / projets) | BTP, immobilier, gestion de chantiers | Grand compte, vente complexe, spécialiste requis |
| **0 DA** | **Non qualifié** | — | Prospect créé sans produit défini → à qualifier, pas à chiffrer |

⚠️ Le montant de la **prospection** = tarif catalogue. Le montant réellement négocié apparaît souvent
dans la **conversation WhatsApp** (remises, offres Sara, paiement BaridiMob). Toujours préférer le
montant discuté en conversation s'il existe ; sinon, utiliser le tarif catalogue ci-dessus.

---

## 2. Taxonomie des états du pipeline (champ `etat`)

Voir aussi le skill `suivi-prospections`. Rappel condensé orienté action :

| État | Sens | Chaleur de base |
|---|---|---|
| `Premiere Entree` / `deuxieme entree` | Lead entrant pas (ou peu) travaillé | Tiède — à qualifier vite |
| `PC-Rappel` | Premier Contact établi, rappel convenu | Tiède→chaud selon conversation |
| `SAD-Rappel` | **Suivi Après Démonstration** — le client a vu une démo | **Chaud** — le plus proche du closing |
| `Intéressé` | Intérêt explicite déclaré | Chaud |
| `Encours de livraison` | Quasi conclu, livraison en cours | Très chaud / gagné |
| `PC-NRP` / `PC-INJ` / `PC-ligne ocupée` | Tentative de contact échouée | Froid — nouvelle tentative |
| `PC-pas interssé` / `SAD-Pas Intéressé` | Refus explicite | Mort — ne pas relancer sauf campagne dédiée |
| `Faux numero` / `Reclamation version d'essai` | À nettoyer / mal classé | Hors pipeline commercial |

Règle de séniorité : à conversation égale, un `SAD-*` passe devant un `PC-*` (il a déjà vu le produit).

---

## 3. Modèle de scoring de chaleur (heat score)

Objectif : distinguer « client *vraiment* chaud » d'un simple « rappel en attente ». Le score se
construit en lisant la **conversation WhatsApp** du prospect (le CRM seul ne suffit pas). Additionne :

**Signaux d'intention forts (WhatsApp) — +3 chacun :**
- Mentionne un moyen/une intention de paiement (« je paie par BaridiMob », « نخلص »).
- Demande explicitement une facture / un devis pour acheter.
- Demande à parler à un humain / un commercial (le bot ne lui suffit plus).
- Dit « ok », « d'accord », « je prends » ou équivalent darija (« واخا »، « نشري »).

**Signaux d'engagement moyens — +2 chacun :**
- A regardé la vidéo / le catalogue / l'essai envoyé.
- Pose des questions précises sur le prix, les fonctions, l'offline.
- A reçu une offre chiffrée (remise négociée).
- État `SAD-*` (démo déjà faite).

**Signaux de valeur / contexte — +1 chacun :**
- Produit gros ticket (DGProduction 60K, Entreprise, DGProject).
- Grand compte (>50 employés, plusieurs points de vente, marque connue).
- Dernier message *entrant* récent (< 3 jours) et resté sans réponse.

**Malus :**
- Dernier contact > 14 jours sans réaction : −2 (refroidissement).
- Objection bloquante non levée (prix trop élevé, offline indispensable) : −2.

**Conversion score → priorité :**
- **🔴 URGENT (Tier 1)** : score ≥ 6, OU un seul signal critique (demande humain / intention de paiement / essai à relancer sous 48h). À traiter en premier, par un commercial senior pour les gros tickets.
- **🟠 HAUTE (Tier 2)** : score 3–5. Chaud à faire avancer cette semaine.
- **🟡 NORMALE / 🟢 SUIVI (Tier 3)** : score ≤ 2. À qualifier ou relancer sans urgence.

Le score n'est pas une fin : **toujours afficher la *raison* en clair** (« essai envoyé le 21, pas
relancé » plutôt que « score 7 »). Un commercial agit sur la raison, pas sur le chiffre.

---

## 4. Règles d'escalade & de méthode (issues des conversations réelles)

Ces règles sont les recommandations récurrentes d'un directeur commercial. Le skill doit vérifier
si elles sont enfreintes dans les données et, si oui, les remonter en recommandations.

1. **Demande d'humain = alerte immédiate.** Si un client écrit qu'il veut parler à quelqu'un / que le bot ne convient pas, il doit être rappelé le jour même. Tout retard ici est une perte sèche.
2. **Essai / version d'essai → rappel sous 24–48 h.** Passé ce délai, l'intérêt retombe. Repérer tout prospect « essai envoyé » sans relance récente.
3. **Gros compte → commercial senior.** Tout prospect > 50 employés, multi-points de vente, ou ticket > 30 000 DA ne doit pas être laissé au bot seul : router vers un commercial expérimenté.
4. **Le bot qualifie, l'humain conclut.** Dès qu'un client exprime une intention d'achat (« ok », paiement), un humain prend le relais pour finaliser — ne pas laisser le bot « gérer » un closing.
5. **Objection offline récurrente.** Plusieurs prospects demandent si ça marche hors connexion. Préparer une réponse standard ; c'est une objection bloquante fréquente.
6. **Chaque rappel doit avoir un responsable + une date.** Les `*-Rappel` stagnent quand personne n'en est propriétaire. Recommander l'attribution nominative (donnée « commercial responsable » à activer côté DGStock).
7. **Pipeline dormant = argent qui dort.** Tout `PC-Rappel`/`SAD-Rappel` de gros ticket (60K) sans contact depuis >14 j est une priorité de réactivation, pas un dossier perdu.
8. **Cibler avant de prospecter.** Le taux de `pas intéressé` est élevé (~45 % de l'historique). Recommander de mieux qualifier en amont (secteur, taille, budget) — prioriser production, distribution, multi-boutiques.

---

## 5. Semaine de travail

Semaine ouvrée = **dimanche → jeudi** (vendredi/samedi = week-end). Le plan d'action couvre ces
5 jours. Créneaux d'appel type : 9h00–12h00 puis 14h00–17h30. Compter ~10–15 appels/jour/commercial,
les Tier 1 le matin (meilleur taux de décrochage et d'attention), les Tier 3 en fin de journée.

---

## 6. Pièges techniques du MCP (à connaître pour ne pas perdre de temps)

- L'opérateur `in` ne marche pas sur `etat` → une requête `=` par état, puis fusion côté analyse.
- `=` sur valeurs accentuées (`Intéressé`) peut renvoyer 0 → utiliser `like "ress"` puis filtrer le texte exact côté résultat (attention : « ress » matche aussi « pas interssé »).
- `group_by` sur un champ à forte cardinalité (téléphone sur 7 900 clients) → erreur « trop de tokens ». OK sur `etat`/`totalTTC` à condition de borner par dates.
- `limit` max 200, pas de pagination `offset` → travailler par lots (filtre sur `created_at`/`numero`).
- `dgstock_generate_pdf` est **en panne** (« Outil inconnu ») → produire un HTML imprimable, pas de PDF serveur.
- Le téléphone n'est pas toujours sur la relation `client` d'une prospection → le chercher via `dgstock_fetch_clients` (search par nom), sinon marquer « → à vérifier dans DGStock ».
