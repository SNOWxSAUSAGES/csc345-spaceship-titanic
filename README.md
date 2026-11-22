# Spaceship Titanic — CSC345 Data Science Project (Phases 1–3)

Team **TOM LENG SAEB** · CSC345 · Orange Data Mining 3.40

Kaggle competition: [Spaceship Titanic](https://www.kaggle.com/competitions/spaceship-titanic)
(8,693 training passengers, 4,277 test passengers)

| Phase | Goal | Main result |
|---|---|---|
| 1 | Business EDA: CryoSleep pricing and onboard revenue | Minimum CryoSleep fee ≈ 3,109 credits per adult |
| 2 | Predict `Transported` (first round of models) | Gradient Boosting · CV 80.8% · Kaggle 0.80500 · rank 681 / 1,893 |
| 3 | Improve the model (second round) | CatBoost (tuned) · CV 82.0% · Kaggle 0.80921 · rank 308 / 2,127 |

## Files

```
data/
  train.csv, test.csv                       Kaggle data
phase1/
  csc345_spaceship_titanic_phase1.ows       Phase 1 workflow (EDA)
phase2/
  csc345_spaceship_titanic_phase2.ows       Phase 2 workflow (4 models)
  submission.csv                            Kaggle 0.80500
phase3/
  csc345_spaceship_titanic_phase3.ows       Phase 3 workflow (also contains the Phase 2 branch)
  submission_phase3_R2.csv                  Final submission, CatBoost set 4, Kaggle 0.80921
  results/                                  Screenshots: Test & Score, confusion matrix, ROC,
                                            CV per fold, tuning results, Kaggle
```

> **Opening a workflow:** the `.ows` files store the path to `train.csv` / `test.csv`
> from the computer they were made on. If Orange cannot find the data, double-click
> the **CSV File Import** widgets (`train.csv`, `test.csv`) and choose the files in `data/`.

---

## Phase 3 — Improving the model

### What changed from Phase 2
1. Smarter, rule-based handling of missing values
2. New cabin and spending features (13 → 18 inputs)
3. Imputation moved **inside** cross-validation (no data leakage)
4. CatBoost instead of scikit-learn Gradient Boosting, plus tuning of 10 settings
5. Accuracy reported for every fold; spatial view of Deck × Side

### Pipeline (Orange)
```
train.csv → Formula P3-A (clean) → Formula P3-B (new features) → Select Columns P3 (18 features)
          → Test & Score P3 (10-fold stratified CV)
               learners: CatBoost, CatBoost tuned, GB (Phase 2 settings), Neural Network
               each learner gets Preprocess (Impute: average / most frequent) as its Preprocessor
          → Confusion Matrix · ROC Analysis · Formula + Group by (accuracy per fold)
CatBoost tuned (Model) + test.csv → Predictions P3 → Select Columns → Edit Domain
          → Save Data (submission CSV)
```

### Step 1 — Formula P3-A: cleaning rules
| New variable | Expression (Orange Formula) | Why |
|---|---|---|
| `RoomService0` … `VRDeck0` | `0 if isnan(RoomService) else RoomService` (same for each service) | Missing spending most likely means no spending |
| `Cryo` | `CryoSleep if CryoSleep in ("True", "False") else ("False" if (RoomService > 0 or FoodCourt > 0 or ShoppingMall > 0 or Spa > 0 or VRDeck > 0) else "True")` | Sleeping passengers cannot spend; 85% of passengers with zero spending were in CryoSleep |
| `Deck` | `Cabin.split("/")[0] if Cabin else "Unknown"` | Location on the ship |
| `Side` | `Cabin.split("/")[2] if Cabin else "Unknown"` | Port (P) or Starboard (S) |
| `CabNum` | `float(Cabin.split("/")[1]) if Cabin else float("nan")` | Position along the ship |
| `CabinMissing` | `0 if Cabin else 1` | Keep "missing" as information instead of guessing |

### Step 2 — Formula P3-B: new features
| New variable | Expression |
|---|---|
| `TotalSpend` | `RoomService0 + FoodCourt0 + ShoppingMall0 + Spa0 + VRDeck0` |
| `NoSpend` | `1 if (RoomService0 + FoodCourt0 + ShoppingMall0 + Spa0 + VRDeck0) == 0 else 0` |
| `CabRegion` | `CabNum // 300` |
| `DeckSide` | `Deck + "_" + Side` |

**18 input features:** HomePlanet, Cryo, Destination, Age, VIP, RoomService0, FoodCourt0,
ShoppingMall0, Spa0, VRDeck0, TotalSpend, NoSpend, Deck, Side, CabNum, CabRegion,
CabinMissing, DeckSide. Target: `Transported`. `PassengerId` is kept as a meta column for the submission.

### Step 3 — Leak-free evaluation
- Phase 2 imputed means / modes on all training data **before** CV, so the test folds were part of the averages.
- In Phase 3 a **Preprocess (Impute)** widget is connected to each learner's *Preprocessor* input,
  so means and modes are learned from the training folds only.
- Rule-based fills (0, "Unknown") use no statistics, so they cannot leak.
- Same evaluation as Phase 2: 10-fold stratified cross-validation on 8,693 passengers.

### Step 4 — Models (10-fold CV)
| Model | CA | AUC | F1 | MCC |
|---|---|---|---|---|
| **CatBoost tuned (set 4)** | **0.820** | 0.908 | **0.820** | **0.639** |
| CatBoost (default: 500 trees, lr 0.05, depth 6) | 0.816 | 0.909 | 0.816 | 0.633 |
| Gradient Boosting (Phase 2 settings) | 0.811 | 0.904 | 0.811 | 0.621 |
| Neural Network (impute + normalize) | 0.800 | 0.897 | 0.800 | 0.601 |

The same Gradient Boosting as Phase 2 rose from 80.8% to 81.1% with the new features alone.

### Step 5 — Tuning CatBoost (10 settings, 2 rounds)
| Set | Depth | λ (L2) | Feature fraction | Learning rate | Trees | Mean CV |
|---|---|---|---|---|---|---|
| 1 | 5 | 8 | 0.8 | 0.03 | 800 | 81.63% |
| 2 | 6 | 3 | 0.8 | 0.03 | 800 | 81.76% |
| 3 | 7 | 10 | 0.8 | 0.03 | 800 | 81.47% |
| **4** | **4** | **5** | **1.0** | **0.04** | **1000** | **81.96%** |
| 5 | 6 | 5 | 0.7 | 0.03 | 800 | 81.88% |
| 6 | 4 | 5 | 0.7 | 0.04 | 1000 | 81.63% |
| 7 | 3 | 3 | 0.7 | 0.05 | 1200 | 81.76% |
| 8 | 4 | 3 | 0.6 | 0.04 | 1000 | 81.63% |
| 9 | 5 | 3 | 0.7 | 0.03 | 800 | 81.90% |
| 10 | 4 | 1 | 0.7 | 0.04 | 1000 | 81.70% |

Set 4 was chosen because it had the **highest CV**, not because of the leaderboard.
CatBoost tuned (set 4): **81.96% ± 1.52%** across folds; every fold between 79.3% and 84.1%.

### Step 6 — Confusion matrix (CatBoost set 4 vs Phase 2)
| | Phase 2 (GB) | Phase 3 (set 4) |
|---|---|---|
| False positives | 911 | 846 |
| False negatives | 759 | 722 |
| Total errors | 1,670 | 1,568 (−102) |
| Accuracy | 80.8% | 82.0% |

### Kaggle submissions
| Submission | CV | Kaggle |
|---|---|---|
| Phase 2: Gradient Boosting | 80.8% | 0.80500 |
| Phase 3: CatBoost default | 81.6% | 0.80593 |
| Phase 3: CatBoost tuned v1 (depth 6, 800 trees) | 81.7% | 0.80874 |
| **Phase 3: CatBoost set 4 (final)** | **82.0%** | **0.80921** |
| Phase 3: CatBoost set 9 | 81.9% | 0.80640 |

Rank **681 / 1,893 → 308 / 2,127**. Set 9 had almost the same CV as set 4 but a lower
Kaggle score, so differences this small are noise.

### What we learned
1. Data quality beats model complexity: simple rules helped as much as a new model.
2. Evaluate honestly: impute inside CV, use the same CV for every model, choose by CV.
3. Small differences can be noise: folds vary by about ±1.5%.
4. Tuning has diminishing returns: our 10 settings spanned only 0.5 points.
5. Visualizing the data (Deck × Side) gives feature ideas.

---

## Phase 2 — First round of models

- **Target:** `Transported` (True 50.4% / False 49.6%)
- **13 features:** HomePlanet, CryoSleep, Destination, Age, VIP, 5 spending services,
  TotalSpend, Deck, Side (from `Cabin`). `Name` not used; `PassengerId` kept only for the submission file.
- **Preprocessing:** Impute (spending → 0, Age → mean, categorical → most frequent),
  Formula (TotalSpend, Deck, Side), one-hot encoding done automatically by Orange.
- **Evaluation:** 10-fold stratified cross-validation.

| Algorithm | Default CA | Tuned CA | Tuned settings |
|---|---|---|---|
| Logistic Regression | 0.791 | 0.791 | C = 1 (tried 0.1 and 10, no change) |
| Decision Tree | 0.760 | 0.787 | max depth 8 |
| Random Forest | 0.784 | 0.797 | 100 trees, replicable |
| **Gradient Boosting** | 0.801 | **0.808** | 200 trees, learning rate 0.05, depth 5 |

Best model: Gradient Boosting, accuracy 80.8%, AUC 0.892, MCC 0.616.
Kaggle public score **0.80500**, rank **681 / 1,893**.

---

## Phase 1 — Onboard revenue & CryoSleep pricing (EDA)

Business-focused EDA instead of prediction:
1. **How much should CryoSleep cost** to replace the onboard revenue lost while passengers are asleep?
2. **Where does onboard revenue come from** — by home planet and service?

### Data problems handled
| Problem | How we handled it |
|---|---|
| 2.1–2.5% missing in each column, but 24% of rows have at least one missing value | Missing spending values set to 0 (Impute). Sensitivity check with complete rows only gives a similar result (fee 3,164 vs 3,109) |
| Orange's Formula returns missing if any service is missing → 908 rows (10.4%) would be silently dropped | Impute runs **before** Formula |
| No total spending column | Built `TotalSpend = RoomService + FoodCourt + ShoppingMall + Spa + VRDeck` |
| Heavy right skew (skewness 4.4) and outliers (max 35,987) | Kept outliers — high spenders are real revenue; reported median alongside mean |

### Workflow
```
CSV File Import → Impute → Formula (TotalSpend)
                                  ├─ ① Select Rows (awake adults 13+)  → Group by → Data Table
                                  ├─ ② Select Rows (CryoSleep adults 13+) → Group by → Data Table
                                  ├─ ③ Select Rows (adults 13+)        → Distributions      ★ Chart 1
                                  └─ ④ Select Rows (HomePlanet known)  → Group by → Melt → Formula (%) → Bar Plot   ★ Chart 2
```

### Key insights
- CryoSleep adults come more often from Europa, the biggest spenders.
- **Minimum CryoSleep fee ≈ 3,109 credits per adult** (95% CI 3,004–3,215), weighted by the CryoSleep planet mix;
  a simple average of awake adults (2,444) would under-price by about 21%.
- Total onboard revenue 12.3M credits: Europa 59.8% (25% of passengers), Earth 25.2%, Mars 15.0%.
  Europa FoodCourt alone (3.13M) earns more than all of Earth (3.10M).

### Limitations
- Assumes CryoSleep adults would spend like awake adults from the same planet.
- Minimum fee only; excludes cryo pod costs and savings while asleep.
- Synthetic data; observational results show correlation, not causation.

---

## AI use
Claude (Anthropic) was used as an assistive tool in all phases.
- **Phase 1–2:** checking numbers, analysis support, Orange workflow guidance, drafting and proofreading slide text.
- **Phase 3:** researching tuning methods, pre-testing feature and model ideas in Python before building
  them in Orange, step-by-step Orange instructions and Formula expressions, drafting slide content and
  three charts (spatial map, pipeline draft, improvement chart).

The team built and ran all Orange workflows, ran the 10 tuning settings, produced all reported
CV, confusion-matrix and Kaggle results, and checked every number. See the AI Declaration slide
in each presentation for details.

## Team
- Chotiya Khawsanga 67130500806
- Nicha Hongsrimuang 67130500810
- Benyapon Saisong 67130500841
- Kalyathorn Yakam 67130500850
- Natthanicha Buasamlee 67130500854
