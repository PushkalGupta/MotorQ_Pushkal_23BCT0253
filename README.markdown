# Vehicle Telematics Analysis README

## Overview
This project processes vehicle telematics data to extract ignition and charging status events, performing data sanity checks, event extraction, and battery level association as per the hackathon requirements. The code is implemented in Python using pandas, numpy, matplotlib, seaborn, and scipy for data processing and analysis.

## Discovered Schemas & Data Issues
- **TLM (Telemetry Data)**:
  - Columns: `VEHICLE_ID`, `TIMESTAMP`, `IGNITION_STATUS`, `EV_BATTERY_LEVEL`, `SPEED`, `ODOMETER`.
  - Issues:
    - Timestamps stored as strings, converted to UTC datetime.
    - High missing data: 87.5% for `IGNITION_STATUS`, 78.8% for `EV_BATTERY_LEVEL`, 80.8% for `SPEED`.
    - Odometer outliers (15.14% of non-missing values) indicate sensor faults; removed using IQR method.
- **TRG (Triggers Data)**:
  - Columns: `PNID`, `CTS`, `NAME`, `VAL`.
  - Issues:
    - `CTS` timestamps in IST; converted to UTC for consistency.
    - Contains `IGN_CYL` and `CHARGE_STATE` events; `VAL` values inconsistent (e.g., 'ON'/'OFF' vs. 1/0).
    - Duplicate events detected and removed.
- **MAP (Mapping Data)**:
  - Columns: `ID`, `IDS`.
  - Issues:
    - `IDS` stored as JSON strings; parsed into lists.
    - 16 unique vehicles, 13 with PNID mappings; 3 vehicles unmapped.
- **SYN (Synthetic Data)**:
  - JSON with `vehicleId`, `timestamp`, `type` (all `ignitionoff`).
  - Issues: Timestamps converted to UTC; no missing values.

## Design Choices
- **Data Cleaning**:
  - Standardized `IGNITION_STATUS` to binary (1/0).
  - Handled clock drift by converting all timestamps to UTC.
  - Parsed `IDS` in MAP to create an exploded mapping table for joins.
- **Ignition Event Extraction**:
  - Used transition-based logic (on→off, off→on) for TLM and TRG.
  - Prioritized sources: SYN > TRG > TLM to resolve conflicts.
  - Removed duplicates based on `vehicle_id` and `event_ts`.
- **Charging Event Extraction**:
  - Complex logic: Grouped TRG `CHARGE_STATE` events into sessions (≤300s gaps).
  - Events classified as `Active` (≥1% battery increase), `Completed` (≥80% or stabilized), or `Abort` (<80% and <5% change).
  - Simple logic: Assumed pre-labeled events in TRG (not used due to data mismatch).
- **Battery Level Association**:
  - Matched closest battery reading within ±300s.
  - Tie-breaker: Prefer post-event reading, then highest battery level, then earliest timestamp.
- **Charging Event Detection**:
  - Applied Savitzky-Golay filter for noise reduction.
  - Defined sessions with ≥3% battery increase over ≤6 hours.

## What I'd Improve Next
- Enhance outlier detection with machine learning (e.g., isolation forests) for robustness.
- Implement sliding window or ML-based charging detection (e.g., CNN-RF model) for noisy data.
- Add validation step to cross-check events against external logs.
- Optimize performance for large datasets using parallel processing or SQL.
- Handle unmapped vehicles by imputing PNIDs or flagging for manual review.

## Execution
Run the script with:
```bash
python vehicle_telematics_analysis.py
```
Outputs: `IgnitionEvents.csv`, `ChargingStatusEvents.csv`, `charging_sessions_smoothed.csv`.