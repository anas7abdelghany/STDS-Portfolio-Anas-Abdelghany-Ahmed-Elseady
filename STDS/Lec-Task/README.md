# 🏗️ Energy Data ETL Pipeline Project

This repository contains the documentation and logic for an automated ETL (Extract, Transform, Load) pipeline designed for smart meter energy data.

---

## 📍 Task A: System Design
### **ETL Architecture Diagram**
The following diagram illustrates the data flow from raw smart meter ingestion to structured storage and archival.

![ETL Architecture](./TaskA.png)

---

## 📜 Task B: Business Logic & Data Governance
This section defines the rigorous rules applied during the transformation phase to ensure data integrity and reliability.

### **1. Unit Standardization**
| Rule ID | Condition | Action |
| :--- | :--- | :--- |
| **Rule 1** | `energy_unit == "W"` | Convert to `kW` (Value / 1000) |
| **Rule 2** | `energy_unit` in `["kwh", "KWH"]` | Standardize naming to `"kW"` |
| **Rule 3** | `energy_unit` NOT IN `["W", "kW"]` | Default to `"kW"` & Flag `ASSUMED_UNIT` |

### **2. Handling Missing Values**
* **Rule 4:** IF `energy_value` IS NULL → Status: `MISSING_VALUE` & Exclude from peak calculations.
* **Rule 5:** IF `timestamp` IS NULL → Infer time (`previous + 15m`) & Status: `INFERRED_TIME`.
* **Rule 6:** IF `meter_id` IS NULL → **REJECT RECORD COMPLETELY**.

### **3. Data Validation & Quality**
* **Rule 7:** Negative values are flagged as `INVALID_NEGATIVE` and rejected from analytics.
* **Rule 8:** Residential values `> 50kW` are flagged as `SUSPICIOUSLY_HIGH` for manual review.
* **Rule 9:** Abrupt changes (`> 30kW` in 15 mins) are flagged as `ABRUPT_CHANGE`.
* **Rule 10:** Future timestamps (>5 mins) are corrected to `current_time` and flagged.

### **4. Faulty Meter Detection**
> [!IMPORTANT]
> These rules help in proactive maintenance and identifying hardware issues.
* **Zero Consumption:** 24 consecutive hours of `0` value → `POTENTIAL_FAULTY`.
* **Stuck Meter:** Standard deviation `< 0.01` over 48 hours → `POTENTIAL_STUCK_METER`.
* **Connectivity:** No readings for `> 72 hours` → `OFFLINE` status + Alert.

---

## ⚙️ Task C: Data Pipeline Execution Flow

### **1. Ingestion (Raw Storage)**
Raw CSV data (e.g., `500W`) lands in an **S3 Bucket (Landing Zone)** via API. This preserves the "Single Source of Truth" for auditing and compliance.

### **2. Orchestration (Trigger)**
An S3 Event triggers an **AWS Lambda** function. 
* **Retry Logic:** Automatic 3x retry on initial trigger failure before logging manual intervention.

### **3. Transformation Layer**
In the transformation function:
1.  **Parsing:** Extract fields from the CSV.
2.  **Standardization:** Convert units (e.g., `500W` → `0.5kW`).
3.  **Null Checking:** Verify mandatory fields.
4.  **Enrichment:** Add UTC timestamps, processing IDs, and Quality Scores.

### **4. Structured Loading (RDS)**
Cleaned data is loaded into an **RDS SQL Table** (`cleaned_readings`). 
* **Purpose:** Immediate SQL queries for peak detection and real-time insights.
* **Resilience:** Retries with **Exponential Backoff** and Dead Letter Queue (DLQ).

### **5. Analytical Archiving (Parquet)**
Simultaneously, data is converted to **Apache Parquet** format.
* **Efficiency:** Reduces storage by ~80% and optimizes columnar analytics.
* **Organization:** Partitioned by `date/hour/meter_id`.

### **6. Error Handling & Success Policy**
* ✅ **On Success:** Completion metrics logged; data available for operational and analytical queries.
* ❌ **On Failure:** Records move to a **DLQ** for manual inspection, an **SNS Alert** is triggered, and bad data is excluded from downstream propagation.

---

### **Next Steps**
* [ ] Integration with Grafana/QuickSight for real-time monitoring.
* [ ] Implementation of predictive maintenance models using the Parquet archive.
