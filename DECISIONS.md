# DECISIONS.md — Pipeline de Nettoyage FinanceCore SA

> **Projet :** Nettoyage & Enrichissement du dataset `bank_transactions.csv`
> **Fichier source :** `bank.csv` (2 060 lignes × 16 colonnes)
> **Fichier produit :** `financecore_clean.csv` (2 060 lignes × 24 colonnes)
> **Auteur :** Équipe Data FinanceCore SA

---

## 1. Importation & Exploration

### Constats initiaux

| Colonne | Problème détecté |
|---|---|
| `montant` | Type `object` — séparateur décimal virgule au lieu de point |
| `date_transaction` | Type `object` — formats mixtes (JJ/MM/AAAA et AAAA-MM-JJ) |
| `solde_avant` | Type `object` — suffixe textuel ` EUR` présent sur certaines valeurs |
| `devise` | Casse incohérente (ex. `eur` au lieu de `EUR`) |
| `segment_client` | Casse incohérente (ex. `PREMIUM` au lieu de `Premium`) |
| `agence` | Espaces parasites en début/fin de chaîne |
| `score_credit_client` | 8,11 % de valeurs manquantes ; valeurs aberrantes (négatives, > 850) |
| `segment_client` | 5,10 % de valeurs manquantes |
| `agence` | 3,11 % de valeurs manquantes |
| `taux_interet` | 100 % de valeurs manquantes → colonne entièrement vide |

**Doublons :** détectés sur `transaction_id` ; analyse effectuée avant suppression.

---

## 2. Nettoyage des données

### 2.1 Suppression des doublons

- **Méthode :** `drop_duplicates(subset="transaction_id", keep="first")`
- **Décision :** Conservation de la première occurrence par `transaction_id`. Les occurrences suivantes (potentiellement issues de re-soumissions ou d'erreurs système) sont supprimées.
- **Justification :** Un identifiant de transaction doit être unique dans un système bancaire. La première occurrence est considérée comme la transaction originale et donc la plus fiable.

### 2.2 Correction des formats de dates

- **Méthode :** `pd.to_datetime(df["date_transaction"], errors="coerce")` puis `.dt.strftime("%Y-%m-%d %H:%M:%S")`
- **Décision :** Unification au format ISO 8601 (`AAAA-MM-JJ HH:MM:SS`). Les dates non parsables sont converties en `NaT` (`errors="coerce"`).
- **Justification :** Un format unique est indispensable pour le tri chronologique, les agrégations temporelles et l'alimentation d'une base de données relationnelle.

### 2.3 Correction des montants

- **Méthode :** `df["montant"].str.replace(',', '.')` puis conversion en `float` via `pd.to_numeric`
- **Décision :** Remplacement du séparateur décimal virgule par un point, puis cast en `float64`.
- **Justification :** Python et les bases de données SQL utilisent le point comme séparateur décimal. La virgule rendait la colonne non-numérique et bloquait tout calcul.

### 2.4 Nettoyage de `solde_avant`

- **Méthode :** `.str.replace("EUR", "").str.replace(',', '.')` puis `pd.to_numeric(..., errors='coerce')`
- **Décision :** Suppression du suffixe textuel ` EUR` et conversion en `float64`.
- **Justification :** Le suffixe textuel est une information redondante (la devise figure dans la colonne `devise`). La colonne doit être purement numérique pour les calculs de solde.

### 2.5 Normalisation des devises

- **Méthode :** (normalisé lors du nettoyage des casses textuelles)
- **Décision :** Passage en majuscules (`eur` → `EUR`, `gbp` → `GBP`, etc.).
- **Justification :** Les codes devises ISO 4217 sont conventionnellement en majuscules. La cohérence est nécessaire pour les jointures et les regroupements.

### 2.6 Harmonisation de `segment_client`

- **Méthode :** `df["segment_client"].str.capitalize()`
- **Décision :** Casse *Title Case* (`PREMIUM` → `Premium`, `standard` → `Standard`).
- **Justification :** Évite les doublons sémantiques lors des `groupby` et assure une présentation cohérente dans les dashboards.

### 2.7 Suppression des espaces parasites sur `agence`

- **Méthode :** `df["agence"].str.strip()`
- **Décision :** Suppression des espaces en début et fin de chaîne.
- **Justification :** Les espaces parasites créent des valeurs distinctes invisibles à l'œil (ex. `" Paris"` ≠ `"Paris"`) qui faussent les agrégations par agence.

### 2.8 Traitement des valeurs manquantes

| Colonne | Taux NaN | Méthode retenue | Décision |
|---|---|---|---|
| `score_credit_client` | 8,11 % | Imputation par **médiane** | La médiane est robuste aux valeurs extrêmes, adaptée à une distribution potentiellement asymétrique des scores crédit. |
| `segment_client` | 5,10 % | Imputation par **mode** | Variable catégorielle : on attribue la catégorie la plus fréquente, sans introduire de biais numérique. |
| `agence` | 3,11 % | Imputation par **mode** | Variable catégorielle : même logique que `segment_client`. |
| `taux_interet` | 100 % | **Conservation** (colonne vide) | La colonne est conservée pour des raisons de compatibilité de schéma. Elle pourra être alimentée lors d'une prochaine collecte de données. |
| `date_transaction` | ~3,9 % (NaT) | **Conservation** (NaT) | Les dates non parsables restent `NaT`. Elles sont exclues des features temporelles (année, mois, trimestre) mais la ligne est conservée pour les analyses non-temporelles. |

---

## 3. Détection & Traitement des Valeurs Aberrantes

### 3.1 Méthode utilisée : IQR (Interquartile Range)

La méthode IQR a été préférée au Z-score car elle est plus robuste aux distributions non normales, fréquentes sur les données financières (queues épaisses).

**Formule :**
```
Borne inférieure = Q1 - 1.5 × IQR
Borne supérieure = Q3 + 1.5 × IQR
```

### 3.2 Colonne `montant`

| Indicateur | Valeur |
|---|---|
| Q1 | ~-1 218,87 |
| Q3 | ~957,81 |
| IQR | ~2 176,68 |
| Borne inférieure | ~-4 484,89 |
| Borne supérieure | ~4 223,83 |

- **Décision :** Les transactions hors bornes (incluant les valeurs à 0 €, +99 999 €, -200 000 €) sont **flagguées** via la colonne `is_anomaly = True`. Elles **ne sont pas supprimées**.
- **Justification :** En contexte bancaire, une transaction aberrante peut être légitime (virement immobilier, prélèvement exceptionnel). La suppression aveugle risquerait de biaiser les analyses de risque. Le flag permet à la Direction des Risques de les examiner manuellement.

### 3.3 Colonne `score_credit_client`

- **Règle métier appliquée :** scores < 0 ou > 850 → `is_anomaly = True`
- **Décision :** Flag combiné avec l'anomalie montant. Scores négatifs (ex. -100) et scores > 850 (ex. 1 500) sont impossibles selon le référentiel crédit standard (0–850).
- **Justification :** Ces valeurs résultent d'erreurs de saisie ou de migration de données. Elles sont conservées pour traçabilité et examen métier, mais exclues des calculs de `categorie_risque`.

### 3.4 Colonne `is_anomaly`

- **Type :** `bool`
- **Valeur `True` :** transaction présentant au moins une anomalie (montant hors IQR OU score crédit hors plage métier)
- **Nombre d'anomalies détectées :** visible dans les outputs du notebook

---

## 4. Feature Engineering

### 4.1 Variables temporelles

Extraites depuis `date_transaction` (après reconversion en `datetime64`) :

| Colonne créée | Contenu | Décision |
|---|---|---|
| `annees` | Année (ex. 2022, 2023, 2024) | Permet les analyses de tendance annuelle |
| `mois` | Mois numérique (1–12) | Analyse de saisonnalité mensuelle |
| `jours_semaine` | Nom du jour (ex. Monday) | Détection de patterns hebdomadaires (fraude, affluence) |
| `trimestre` | Trimestre (1–4) | Agrégation trimestrielle pour reporting DG |

- **Note :** Ces colonnes sont `NaN`/`None` pour les 80 lignes dont la date était non parsable.

### 4.2 Vérification `montant_eur`

- **Calcul :** `montant_eur_ver = montant / taux_change_eur`
- **Colonne `diff` :** `diff = montant_eur_ver - montant_eur`
- **Décision :** La colonne `diff` est conservée à titre de contrôle qualité. Les écarts non nuls (hors arrondi flottant) signalent des incohérences dans le système source.
- **Justification :** La comparaison permet d'auditer la fiabilité des taux de change appliqués en amont.

### 4.3 `categorie_risque`

- **Méthode :** `pd.cut()` sur `score_credit_client`
- **Segmentation :**

| Catégorie | Plage de score | Nombre de clients |
|---|---|---|
| Low (Faible risque) | ≥ 700 | 481 |
| Medium (Risque modéré) | 580 – 699 | 1 157 |
| High (Risque élevé) | < 580 | 420 |

- **Décision :** Seuils alignés sur les standards Bâle III / scoring crédit retail. La catégorie `High` déclenche une vigilance renforcée dans le dashboard Risques.

### 4.4 Agrégations clients (`df_clients`)

Calculées séparément (non mergées dans le DataFrame principal) :

| Indicateur | Calcul |
|---|---|
| `nb_transactions` | Comptage par `client_id` |
| `montant_moyen` | Moyenne du montant par client |
| `nb_produits_distincts` | Nombre de produits uniques par client |

### 4.5 `taux_rejet` par agence

- **Calcul :** proportion de transactions avec `statut == "rejetée"` par agence
- **Note :** La valeur de statut `"rejetée"` doit être vérifiée — la valeur brute observée dans les données est `"Rejete"` (sans accent, casse variable). Une normalisation préalable du statut est recommandée pour fiabiliser ce calcul.

---

## 5. Export

- **Fichier exporté :** `financecore_clean.csv`
- **Format :** CSV UTF-8, séparateur virgule, sans index
- **Dimensions finales :** 2 060 lignes × 24 colonnes
- **Nouvelles colonnes ajoutées :** `is_anomaly`, `annees`, `mois`, `jours_semaine`, `trimestre`, `montant_eur_ver`, `diff`, `categorie_risque`

### Vérifications finales

| Contrôle | Résultat |
|---|---|
| Valeurs manquantes sur colonnes critiques | ✅ Traitées (médiane/mode) |
| Type `montant` | ✅ `float64` |
| Type `solde_avant` | ✅ `float64` |
| Type `date_transaction` | ✅ `datetime64[ns]` |
| Colonne `is_anomaly` présente | ✅ `bool` |
| Colonne `categorie_risque` présente | ✅ `category` |

---

*Document généré dans le cadre du projet de nettoyage de données FinanceCore SA.*
