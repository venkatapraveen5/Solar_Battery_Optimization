# Solar Battery Optimization: A Predict-Then-Optimize Pipeline

An end-to-end pipeline that forecasts solar power generation using 
Machine Learning and optimizes battery charge/discharge scheduling 
using Mixed Integer Linear Programming (MILP) to minimize grid 
electricity costs over a 24-hour horizon.

## Problem Statement

Solar generation is intermittent — it peaks at midday and drops to 
zero at night. A battery storage system can store excess daytime 
energy and discharge it during evening peak hours when grid 
electricity is expensive. The challenge is deciding *when* to charge, 
discharge, buy from the grid, and sell back — optimally.

This project solves that problem in two stages:
1. **Predict** — forecast solar generation for the next 24 hours
2. **Optimize** — schedule battery operation to minimize net cost

## Dataset

Solar Power Generation Data from Kaggle:
https://www.kaggle.com/datasets/anikannal/solar-power-generation-data

Two files used:
- `Plant_1_Generation_Data.csv` — AC/DC power output per inverter, 
   15-minute resolution
- `Plant_1_Weather_Sensor_Data.csv` — irradiation, ambient and module 
   temperature

Date range: May 15 – June 6, 2020  
Resolution: 15-minute intervals (96 time steps per day)  
Analysis performed on Plant 1 (PLANT_ID: 4136001). Extension to 
Plant 2 and multi-plant optimization is a natural next step.

## Pipeline

### Stage 1 — Solar Generation Forecasting (ML)

**Model:** Random Forest Regressor (100 estimators)

**Features:**
- Irradiation
- Ambient temperature
- Module temperature
- Hour of day
- Irradiation difference (rate of change)

**Result:**
- MAE: 1.2% of peak plant capacity (~350W)
- Irradiation dominates feature importance at 99.7% — consistent 
  with the physics of solar generation

### Stage 2 — Battery Scheduling Optimization (MILP)

**Decision Variables:**

| Variable | Type | Description |
|----------|------|-------------|
| ct | Binary | 1 if battery charging at time t |
| dt | Binary | 1 if battery discharging at time t |
| Pt | Continuous | Battery charge level at time t (Wh) |
| bt | Continuous | Power bought from grid at time t (W) |
| st | Continuous | Power sold to grid at time t (W) |

**Constraints:**
- Battery dynamics: Pt = Pt-1 + ct × charge_rate − dt × discharge_rate
- Battery limits: Pmin ≤ Pt ≤ Pmax
- Mutex: ct + dt ≤ 1 (cannot charge and discharge simultaneously)
- Power balance: Solar + Grid_buy + Discharge = Load + Grid_sell + Charge
- Total sold ≤ Total generated
- Final battery state ≥ Initial battery state

**Objective:** Minimize net grid cost


**Battery Parameters:**
- Capacity: 50 kWh
- Minimum charge: 10 kWh (20%)
- Maximum charge: 50 kWh (100%)
- Charge/discharge rate: 12.5 kW per 15-minute interval

**Electricity Prices (synthetic, time-of-use tariff):**
- Midnight to 8am: Rs 4/kWh (off-peak)
- 8am to 2pm: Rs 8/kWh (mid)
- 2pm to 6pm: Rs 12/kWh (peak)
- 6pm to midnight: Rs 6/kWh (mid)

**Solver:** Gurobi 13.0.2 (academic license)  
**Solution time:** ~0.05 seconds for 96 time steps

## Results

For a representative day:
- Solar generated: 635,797 Wh
- Sold to grid: 376,254 Wh (excess daytime solar)
- Bought from grid: 323,956 Wh (night and low-solar periods)
- Battery charges during midday solar peak
- Battery discharges during evening peak hours
- Net grid cost: Rs 1,644

The optimizer correctly identifies:
- Charge during cheap off-peak hours and solar surplus
- Discharge during expensive peak hours
- Buy from grid only when solar and battery cannot meet load

## Requirements
 - numpy
 - pandas
 - scikit-learn
 - gurobipy
 - matplotlib
Gurobi requires a license. Free academic licenses available at 
gurobi.com/academia/academic-program-and-licenses

## How to Run

1. Download dataset from Kaggle link above
2. Place CSV files in the same directory as the notebook
3. Run all cells in order

## Limitations and Next Steps

- Load profile is synthetic — real consumption data would improve 
  results
- Analysis uses actual weather as a perfect forecast — integrating 
  a real weather forecast API would make it production-ready
- Single plant analysis — multi-plant optimization is a natural 
  extension
- Battery parameters are illustrative — real battery specs would 
  improve accuracy

## Author

A V Praveen

github.com/venkatapraveen5

