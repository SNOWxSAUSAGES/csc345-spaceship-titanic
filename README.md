# Spaceship Titanic — Business View (EDA & Visualization)

Business-focused Exploratory Data Analysis and visualization on the
Kaggle **Spaceship Titanic** dataset, built for CSC345.

Instead of predicting the target (`Transported`), we look at the data
from a **business angle**: where the revenue comes from, and how
different passenger groups spend.

## Dataset
- Source: [Spaceship Titanic — Kaggle](https://www.kaggle.com/competitions/spaceship-titanic)
- 8,693 passengers x 14 columns
- 5 spending services: RoomService, FoodCourt, ShoppingMall, Spa, VRDeck

## Tools
- [Orange Data Mining](https://orangedatamining.com) — analysis & charts
- Canva — final chart/slide design

## Data Problems Handled
- ~2% missing values in every column
- No "total spend" column -> created `TotalSpend`
- CryoSleep passengers all spend 0 -> filtered out before the
  spending-taste analysis (otherwise the averages are skewed)

## Method (Orange workflow)
1. **CSV File Import** — load the dataset
2. **Formula** — `TotalSpend = RoomService + FoodCourt + ShoppingMall + Spa + VRDeck`
3. **Box Plot** — TotalSpend grouped by CryoSleep
4. **Select Rows** — keep `CryoSleep = False`
5. **Group by** — mean of each service, grouped by HomePlanet
6. **Formula (%)** — spending share per service per planet:

   ```
   pct_X = X_Mean / (RoomService_Mean + FoodCourt_Mean
           + ShoppingMall_Mean + Spa_Mean + VRDeck_Mean) * 100
   ```
7. **Select Columns** — keep the 5 pct_ columns + HomePlanet
8. **Bar Plot** — spending share by HomePlanet

## Key Insights
**1. One in three customers spend nothing**
- CryoSleep passengers spend exactly 0 (frozen during the trip)
- They are ~35% of all passengers -> a third of customers = zero revenue

**2. Each planet has a different spending taste**
| Planet | RoomService | FoodCourt | ShoppingMall | Spa | VRDeck |
|--------|:-----------:|:---------:|:------------:|:---:|:------:|
| Earth  | 20.0 | 19.8 | 19.5 | 20.8 | 20.0 |
| Europa | 4.0  | 42.6 | 4.4  | 24.1 | 24.9 |
| Mars   | 51.7 | 5.0  | 28.5 | 10.3 | 4.4  |

- Mars -> in-room & shopping
- Europa -> dining & luxury/leisure
- Earth -> evenly spread, no strong preference

## Business Takeaways
1. **Unlock zero-revenue customers** — bundle services with CryoSleep,
   or give incentives to spend while awake.
2. **Targeted marketing by segment** — promote Spa/FoodCourt to Europa,
   RoomService/ShoppingMall to Mars.

## Files
- `spaceship_titanic.ows` — Orange workflow
- `train.csv` — dataset (from Kaggle)

## Team
Chotiya Khawsanga 67130500806
Nicha Hongsrimuang 67130500810
Benyapon Saisong 67130500841
Kalyathorn Yakam 67130500850
Natthanicha Buasamlee 67130500854
