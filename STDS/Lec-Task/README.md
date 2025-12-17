# Task A :-


### **ETL Architecture Diagram (System Design)**


![ETL Architecture](./TaskA.png)


#Task B :-



## **1. Unit Standardization Rules**

**Rule 1:** IF energy_unit = "W" THEN energy_value = energy_value / 1000 AND energy_unit = "kW"  

**Rule 2:** IF energy_unit = "kwh" OR energy_unit = "KWH" THEN energy_unit = "kW" (standardize naming)  

**Rule 3:** IF energy_unit NOT IN ["W", "kW"] THEN energy_unit = "kW" (assume default) AND flag = "ASSUMED_UNIT"



## **2. Missing Values Rules**

**Rule 4:** IF energy_value IS NULL THEN status = "MISSING_VALUE" AND exclude_from_peak_calculations = TRUE  

**Rule 5:** IF timestamp IS NULL THEN timestamp = previous_timestamp + 15_minutes AND status = "INFERRED_TIME"

**Rule 6:** IF meter_id IS NULL THEN REJECT_RECORD COMPLETELY  



## **3. Data Validation Rules**

**Rule 7:** IF energy_value < 0 THEN status = "INVALID_NEGATIVE" AND reject_from_analytics = TRUE  

**Rule 8:** IF energy_value_kw > 50 AND meter_type = "RESIDENTIAL" THEN status = "SUSPICIOUSLY_HIGH" AND requires_manual_review = TRUE

**Rule 9:** IF ABS(current_value - previous_value) > 30 AND time_gap = 15_min THEN status = "ABRUPT_CHANGE"

**Rule 10:** IF timestamp > current_time + 5_minutes THEN timestamp = current_time AND status = "TIME_CORRECTED"



## **4. Faulty Meter Detection Rules**

**Rule 11:** IF energy_value = 0 FOR 24_consecutive_hours THEN meter_status = "POTENTIAL_FAULTY" AND generate_maintenance_ticket = TRUE  

**Rule 12:** IF standard_deviation(last_48h_readings) < 0.01 THEN meter_status = "POTENTIAL_STUCK_METER"

**Rule 13:** IF reading_pattern = "ALL_IDENTICAL_VALUES" THEN meter_status = "SUSPICIOUS_PATTERN"

**Rule 14:** IF consumption < 10%_of_neighborhood_average AND duration > 7_days THEN meter_status = "ABNORMALLY_LOW"

**Rule 15:** IF last_reading > 72_hours_ago THEN meter_status = "OFFLINE" AND communication_alert = TRUE


# Task C :-

## **1. Upload to Raw Storage**

The record arrives as part of a raw CSV file (e.g., meter_id=123, timestamp=2025-12-17T12:00:00, energy=500, unit=W) from the smart meter via API or batch upload. It's stored unchanged in a raw storage bucket (object storage like AWS S3). This serves as the "landing zone" for dark data, preserving the original format for auditing and compliance.



## **2. Triggering of the Transformation Process**

The file upload event automatically triggers serverless orchestration (e.g., AWS Lambda). The orchestrator parses the CSV, extracts individual records, and invokes the transformation layer. If the initial trigger fails (due to high load or network issues), it automatically retries up to 3 times before logging an error for manual intervention.



## **3. Data Cleaning and Validation Steps**

In the transformation function:

Parsing: Extract fields from the CSV record

Unit Standardization: Convert 500 W → 0.5 kW (divide by 1000)

Null Checking: Verify no missing values

Range Validation: Ensure energy value is within acceptable limits (0-50 kW for residential)

Fault Detection: Check time-series context; if part of 24+ hour zero-streak, flag as potentially faulty

Metadata Enrichment: Add cleaned timestamp (UTC), processing ID, and quality score
If validation fails (e.g., invalid range or format), the record is flagged and routed to the error path.



## **4. Storage in Structured Format (RDS)**

The cleaned record is loaded into a structured database (RDS table like cleaned_readings with columns: meter_id, timestamp, energy_kW, flags, quality_score). This enables immediate SQL queries for validation (peak detection) and real-time insights. If the insert fails (database connection issue), the system retries 3 times with exponential backoff; on persistent failure, sends to Dead Letter Queue (DLQ) and triggers an alert.




## **5. Conversion and Archival in Parquet Format**

Simultaneously, the record is converted to Parquet format (optimized for columnar analytics) and appended to partitioned files in archival storage (organized by date/hour/meter_id). Parquet automatically handles compression (reducing storage by ~80%) and schema enforcement, preparing the data for long-term analysis like forecasting and trend detection.




## **6. How Success or Failure is Handled**

On Success: The orchestrator logs completion metrics. The record becomes available in both RDS (for operational queries) and Parquet storage (for analytical queries), enabling peak detection and dashboard updates immediately.

On Failure (during transform/validation): The system retries the failed step up to 3 times with exponential backoff. If all retries fail, the record moves to a DLQ for manual inspection, triggers an alert (email/SNS), and is excluded from downstream analytics to prevent bad data propagation. This ensures pipeline resilience with minimal manual intervention.



