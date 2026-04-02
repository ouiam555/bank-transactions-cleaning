# FinanceCore SA — Pipeline de Nettoyage de Données Transactionnelles

## Présentation du projet

Ce projet constitue la **Phase 1** du pipeline data de FinanceCore SA. Il prend en entrée le fichier brut `bank_transactions.csv` exporté quotidiennement par le système bancaire central, applique un pipeline de nettoyage et d'enrichissement rigoureux, et produit un dataset propre `financecore_clean.csv` prêt à être chargé dans le modèle relationnel PostgreSQL et visualisé via le dashboard Streamlit.

---

## Structure du projet

```
financecore-pipeline/
├── bank.csv                  # Fichier source brut (2 060 lignes × 16 colonnes)
├── bank.ipynb                # Notebook principal — pipeline complet
├── financecore_clean.csv     # Dataset nettoyé et enrichi (sortie)
├── DECISIONS.md              # Journal des décisions de nettoyage
└── README.md                 # Ce fichier
```

---

## Prérequis

- Python 3.8+
- Bibliothèques :

```bash
pip install pandas matplotlib
```

---

## Lancer le pipeline

Ouvrir et exécuter le notebook dans l'ordre des cellules :

```bash
jupyter notebook bank.ipynb
```

Ou exécuter en ligne de commande :

```bash
jupyter nbconvert --to notebook --execute bank.ipynb
```

---

## Description du dataset source

| Propriété | Valeur |
|---|---|
| Fichier | `bank.csv` |
| Nombre de lignes | 2 060 |
| Nombre de colonnes | 16 |
| Période couverte | 2022 – 2024 |

### Colonnes du fichier source

| Colonne | Description | Type brut |
|---|---|---|
| `transaction_id` | Identifiant unique de transaction | `object` |
| `client_id` | Identifiant client | `object` |
| `date_transaction` | Date et heure de la transaction | `object` (formats mixtes) |
| `montant` | Montant de la transaction | `object` (virgule décimale) |
| `devise` | Code devise (EUR, GBP, USD…) | `object` (casse variable) |
| `taux_change_eur` | Taux de change vers EUR | `float64` |
| `montant_eur` | Montant converti en EUR | `float64` |
| `categorie` | Catégorie de transaction | `object` |
| `produit` | Produit bancaire concerné | `object` |
| `agence` | Agence traitante | `object` (espaces parasites) |
| `type_operation` | Crédit / Débit | `object` |
| `statut` | Statut de la transaction | `object` |
| `score_credit_client` | Score crédit (0–850) | `float64` (8 % NaN) |
| `segment_client` | Segment (Standard, Premium, Risque) | `object` (casse variable, 5 % NaN) |
| `solde_avant` | Solde avant transaction | `object` (suffixe EUR) |
| `taux_interet` | Taux d'intérêt | `float64` (100 % NaN) |

---

## Étapes du pipeline

### Étape 1 — Importation & Exploration

- Lecture du fichier source avec `pandas`
- Inspection : `shape`, `dtypes`, `head()`, `describe()`
- Calcul du taux de valeurs manquantes par colonne
- Visualisation en barplot (matplotlib)
- Identification des doublons sur `transaction_id`

### Étape 2 — Nettoyage des données

| Action | Colonne(s) concernée(s) |
|---|---|
| Suppression des doublons (`keep="first"`) | `transaction_id` |
| Unification du format de date → `AAAA-MM-JJ HH:MM:SS` | `date_transaction` |
| Remplacement virgule → point, conversion `float` | `montant` |
| Suppression suffixe ` EUR`, conversion `float` | `solde_avant` |
| Normalisation en majuscules | `devise` |
| Harmonisation casse (`capitalize`) | `segment_client` |
| Suppression espaces parasites (`strip`) | `agence` |
| Imputation par médiane | colonnes numériques (`float64`) |
| Imputation par mode | colonnes catégorielles |

### Étape 3 — Détection & Traitement des Valeurs Aberrantes

- Méthode **IQR** appliquée sur `montant` et `score_credit_client`
- Règle métier : scores crédit < 0 ou > 850 → anomalie
- Création de la colonne `is_anomaly` (booléen)
- **Décision retenue : conservation avec flag** (pas de suppression)

### Étape 4 — Feature Engineering

| Colonne créée | Description |
|---|---|
| `annees` | Année extraite de `date_transaction` |
| `mois` | Mois extrait de `date_transaction` |
| `jours_semaine` | Jour de la semaine (nom anglais) |
| `trimestre` | Trimestre (1–4) |
| `montant_eur_ver` | `montant / taux_change_eur` (vérification) |
| `diff` | Écart entre `montant_eur_ver` et `montant_eur` |
| `categorie_risque` | Low / Medium / High selon `score_credit_client` |

Agrégations additionnelles (DataFrame séparé `df_clients`) :
- `nb_transactions` par client
- `montant_moyen` par client
- `nb_produits_distincts` par client
- `taux_rejet` par agence

### Étape 5 — Export

- Vérification cohérence finale (`isnull`, `dtypes`, `describe`)
- Export : `financecore_clean.csv` (UTF-8, sans index)

---

## Dataset de sortie

| Propriété | Valeur |
|---|---|
| Fichier | `financecore_clean.csv` |
| Nombre de lignes | 2 060 |
| Nombre de colonnes | 24 |

### Colonnes ajoutées par le pipeline

| Colonne | Type | Description |
|---|---|---|
| `is_anomaly` | `bool` | Vrai si transaction aberrante (IQR ou score hors plage) |
| `annees` | `float64` | Année de la transaction |
| `mois` | `float64` | Mois de la transaction |
| `jours_semaine` | `object` | Jour de la semaine |
| `trimestre` | `float64` | Trimestre de la transaction |
| `montant_eur_ver` | `float64` | Montant EUR recalculé (audit) |
| `diff` | `float64` | Écart entre montant EUR recalculé et déclaré |
| `categorie_risque` | `category` | Profil de risque : Low / Medium / High |

---

## Distribution des profils de risque

| Catégorie | Seuil score | Effectif |
|---|---|---|
| Low (Faible risque) | ≥ 700 | 481 |
| Medium (Risque modéré) | 580 – 699 | 1 157 |
| High (Risque élevé) | < 580 | 420 |

---

## Décisions de nettoyage

Toutes les décisions méthodologiques (choix d'imputation, traitement des anomalies, règles de normalisation) sont documentées dans le fichier [`DECISIONS.md`](./DECISIONS.md).

---

## Suite du projet

| Phase | Description |
|---|---|
| **Phase 2** | Modélisation relationnelle PostgreSQL — chargement de `financecore_clean.csv` |
| **Phase 3** | Dashboard interactif Streamlit pour la Direction Générale et la Direction des Risques |

---

*FinanceCore SA — Équipe Data*
