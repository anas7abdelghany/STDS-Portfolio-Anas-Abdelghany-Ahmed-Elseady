# 🏗️ GreenStream Energy: Automated ETL Pipeline Design

[cite_start]This project proposes a conceptual serverless ETL pipeline for **GreenStream Energy** to transform "dark data" from 50,000 smart meters into actionable insights.

---

## 📍 Task A: ETL Architecture Diagram
[cite_start]The design focuses on data flow logic and robust error handling to meet strategic goals[cite: 19, 36].

![ETL Architecture](./TaskA.png)

---

## 📜 Task B: Transformation Logic & Business Rules
[cite_start]These rules resolve real-world data quality issues like inconsistent units and Wi-Fi outages[cite: 10, 11, 15].

### **1. Unit Standardization**
| Rule ID | Condition | Action |
| :--- | :--- | :--- |
| **Rule 1** | `energy_unit == "W"` | Convert to `kW` (Value / 1000) [cite: 45] |
| **Rule 2** | `energy_unit` in `["kwh", "KWH"]` | Standardize naming to `"kW"` |
| **Rule 3** | `energy_unit` NOT IN `["W", "kW"]` | Default to `"kW"` & Flag `ASSUMED_UNIT` |

### **2. Handling Missing Values**
| Rule ID | Condition | Action / Status |
| :--- | :--- | :--- |
| **Rule 4** | `energy_value` IS NULL | [cite_start]Flag & Exclude from peak calculations [cite: 46] |
| **Rule 5** | `timestamp` IS NULL | [cite_start]Infer time (`prev + 15m`) due to Wi-Fi gaps  |
| **Rule 6** | `meter_id` IS NULL | 🚨 **REJECT RECORD COMPLETELY** |

### **3. Data Validation & Quality**
| Rule ID | Condition | Action / Status |
| :--- | :--- | :--- |
| **Rule 7** | `energy_value < 0` | Status: `INVALID_NEGATIVE` + Reject |
| **Rule 8** | `value > 50kW` (Res.) | Status: `SUSPICIOUSLY_HIGH` + Review |
| **Rule 9** | `ABS(change) > 30kW` | Status: `ABRUPT_CHANGE` |
| **Rule 10** | `timestamp > current` | Status: `TIME_CORRECTED` |

### **4. Faulty Meter Detection (Basic Logic)**
| Rule ID | Issue | Condition | Action / Status |
| :--- | :--- | :--- | :--- |
| **Rule 11** | **Zero Consumption** | [cite_start]`0` value for long periods [cite: 47] | `POTENTIAL_FAULTY` |
| **Rule 12** | **Stuck Meter** | Std Dev `< 0.01` over 48 hours | `POTENTIAL_STUCK` |
| **Rule 15** | **Connectivity** | No readings for `> 72 hours` | `OFFLINE` + Alert |

---

## ⚙️ Task C: Single Record Lifecycle
[cite_start]The lifecycle of a record from ingestion to archival[cite: 48, 49]:

1. [cite_start]**Raw Storage:** Record arrives in **CSV format** at the S3 Landing Zone[cite: 17, 51].
2. [cite_start]**Triggering:** Upload event triggers a **Serverless Lambda** function[cite: 52].
3. [cite_start]**Cleaning & Validation:** The system applies business rules to standardize units and validate ranges[cite: 53].
4. [cite_start]**Structured Storage:** Cleaned data is saved to **RDS** for immediate querying and validation[cite: 21, 54].
5. [cite_start]**Archival:** Data is converted to **Parquet format** for long-term predictive analytics[cite: 22, 55].
6. [cite_start]**Failure Handling:** Automatic retries are triggered; persistent failures move to a **DLQ**[cite: 23, 56].

---

### **Strategic Outcomes**
* [cite_start]✅ Prepared for Predictive Analytics[cite: 7].
* [cite_start]✅ Optimized for Large-Scale Historical Analysis[cite: 17].
