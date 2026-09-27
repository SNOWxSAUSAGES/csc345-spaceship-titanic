# Spaceship Titanic — Onboard Revenue & CryoSleep Pricing (EDA & Visualization)

Business-focused Exploratory Data Analysis and visualization on the
Kaggle **Spaceship Titanic** dataset, built for CSC345 (Project Phase 1).

Instead of predicting the target (`Transported`), we look at the data
from a **business angle**:

1. **How much should CryoSleep cost** to replace the onboard revenue lost
   while passengers are asleep?
2. **Where does onboard revenue come from** — by home planet and service?

## Dataset
- Source: [Spaceship Titanic — Kaggle](https://www.kaggle.com/competitions/spaceship-titanic)
- 8,693 passengers × 14 columns
- 5 spending services: RoomService, FoodCourt, ShoppingMall, Spa, VRDeck
- Onboard revenue = sum of the 5 services (unit: credits)

## Tools
- [Orange Data Mining 3.40](https://orangedatamining.com) — analysis & both charts
  (CSV File Import, Impute, Formula, Select Rows, Group by, Melt, Distributions, Bar Plot)
- Canva — slide design

## Data Problems Handled
| Problem | How we handled it |
|---|---|
| 2.1–2.5% missing in each column, but 24% of rows have at least one missing value | Missing spending values set to 0 (Impute). Sensitivity check with complete rows only gives a similar result (fee 3,164 vs 3,109) |
| Orange's Formula returns missing if any service is missing → 908 rows (10.4%) would be silently dropped | Impute runs **before** Formula |
| No total spending column | Built `TotalSpend = RoomService + FoodCourt + ShoppingMall + Spa + VRDeck` |
| Heavy right skew (skewness 4.4) and outliers (max 35,987) | Kept outliers — high spenders are real revenue; reported median alongside mean |
| Composite fields (`Cabin`, `PassengerId`) | Not used in this phase (ignored at import) |

## Method (Orange workflow)
```
CSV File Import → Impute → Formula (TotalSpend)
                                  │
                                  ├─ ① Select Rows (awake adults 13+)  → Group by → Data Table
                                  ├─ ② Select Rows (CryoSleep adults 13+) → Group by → Data Table
                                  ├─ ③ Select Rows (adults 13+)        → Distributions      ★ Chart 1
                                  └─ ④ Select Rows (HomePlanet known)  → Group by → Melt → Formula (%) → Bar Plot   ★ Chart 2
```

1. **CSV File Import** — load `train.csv`; `PassengerId`, `Cabin`, `Name` set to *Ignore*
2. **Impute** — default *Don't impute*; the 5 services use *Fixed value = 0*
3. **Formula** — `TotalSpend = RoomService + FoodCourt + ShoppingMall + Spa + VRDeck`
4. **Branch ① Awake adults** — `CryoSleep is False`, `Age is at least 13`, `HomePlanet is defined` (4,824 rows)
   → **Group by** HomePlanet: `TotalSpend` Mean, Standard deviation, Count
5. **Branch ② CryoSleep adults** — `CryoSleep is True`, `Age is at least 13`, `HomePlanet is defined` (2,511 rows)
   → **Group by** HomePlanet: `Age` Count, `TotalSpend` Max. value (= 0 for everyone)
6. **Branch ③ Adults** — `Age is at least 13`, `HomePlanet is defined`, `CryoSleep is defined` (7,335 rows)
   → **Distributions**: Variable `CryoSleep`, Split by `HomePlanet`, *Stack columns* + *Show probabilities*
7. **Branch ④ Planet known** — `HomePlanet is defined` (8,492 rows)
   → **Group by** HomePlanet: Sum of the 5 services
   → **Melt** (row identifier: HomePlanet)
   → **Formula**: `pct = value / 12303272 * 100`
   → **Bar Plot**: Values `pct`, Group by `HomePlanet`, Color `item`

## Key Insights

### 1. How much should CryoSleep cost?
CryoSleep passengers (3,037) spend **0** onboard. We estimate the revenue they
would have spent using awake adults from the **same home planet**.

| Home planet | Awake adults | CryoSleep adults | Avg spend per awake adult |
|---|:---:|:---:|:---:|
| Europa | 23% | **35%** | 6,367 |
| Earth  | 57% | 43% | 1,067 |
| Mars   | 20% | 22% | 1,890 |

- CryoSleep adults come more often from Europa, the biggest spenders
- **Minimum CryoSleep fee ≈ 3,109 credits per adult** (95% CI 3,004–3,215),
  weighted by the CryoSleep planet mix
- A simple average of awake adults (2,444) would **under-price by ~21%**
- Children (<13) spend 0 even when awake → their fee should be based on cryo cost only

Fee calculation:
```
Fee = (1,068 × 1,067 + 880 × 6,367 + 563 × 1,890) ÷ 2,511 ≈ 3,109
SE  = √( Σ weight² × SD² ÷ n ) = 53.8   →   95% CI = 3,109 ± 105
```

### 2. Where does the revenue come from?
Total onboard revenue: **12.3M credits** (201 passengers with unknown HomePlanet, 1.8% of revenue, excluded)

| Home planet | Revenue | Share | Share of passengers | Top services |
|---|---:|:---:|:---:|---|
| Europa | 7.36M | **59.8%** | 25% | FoodCourt 25.5% · VRDeck 14.9% · Spa 14.4% |
| Earth  | 3.10M | 25.2% | 54% | ~5% from every service |
| Mars   | 1.85M | 15.0% | 21% | RoomService 7.7% · ShoppingMall 4.3% |

- Europa FoodCourt alone (3.13M) earns more than all of Earth (3.10M)

## Business Takeaways
1. **Price CryoSleep to protect revenue** — adults: minimum fee ≈ 3,109 credits
   (one price for all planets); children: price on cryo service cost.
2. **Protect and target the revenue core** — Europa FoodCourt, Spa, VRDeck
   (55% of all revenue); RoomService / ShoppingMall for Mars; grow spend per
   passenger on Earth (largest group, lowest spend).

## Limitations
- Assumes CryoSleep adults would spend like awake adults from the same planet
- Minimum fee only — excludes cryo pod costs and savings while asleep
- One price affects groups unequally (98% of Earth adults spend below 3,109 vs 22% of Europa adults)
- Synthetic data; observational results show correlation, not causation

## Files
- `csc345_spaceship_titanic_phase1.ows` — Orange workflow (Orange 3.40)
- `train.csv` — dataset (from Kaggle)

> Keep both files in the same folder. The workflow loads `train.csv`
> relative to the workflow file, so it opens on any computer.

## AI Use
Claude (Anthropic) was used as an assistive tool for checking numbers,
analysis support, Orange workflow guidance, and drafting/proofreading slide text.
All numbers were re-checked in our own Orange workflow. See the AI Declaration
slide in our presentation for details.

## Team
- Chotiya Khawsanga 67130500806
- Nicha Hongsrimuang 67130500810
- Benyapon Saisong 67130500841
- Kalyathorn Yakam 67130500850
- Natthanicha Buasamlee 67130500854
