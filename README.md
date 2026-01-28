# Healthcare_Analysis
Since you are likely using a modern SQL dialect (like PostgreSQL or MySQL), these `CREATE TABLE` statements include the correct data types and **Foreign Key** relationships. This ensures data integrity—you can’t have a bill for an admission that doesn't exist!
---

## 🏥 Hospital System: Schema Setup

### 1. Patients Table

```sql
CREATE TABLE patients (
    patient_id INT PRIMARY KEY,
    name VARCHAR(100),
    dob DATE,
    gender VARCHAR(10),
    blood_type VARCHAR(5)
);

```

### 2. Doctors Table

```sql
CREATE TABLE doctors (
    doctor_id INT PRIMARY KEY,
    doctor_name VARCHAR(100),
    department VARCHAR(50)
);

```

### 3. Admissions Table

*Note: This table links patients and doctors.*

```sql
CREATE TABLE admissions (
    admission_id INT PRIMARY KEY,
    patient_id INT,
    doctor_id INT,
    admission_date DATE,
    discharge_date DATE,
    reason VARCHAR(255),
    FOREIGN KEY (patient_id) REFERENCES patients(patient_id),
    FOREIGN KEY (doctor_id) REFERENCES doctors(doctor_id)
);

```

### 4. Billing Table

*Note: This table has a 1-to-1 relationship with Admissions.*

```sql
CREATE TABLE billing (
    bill_id INT PRIMARY KEY,
    admission_id INT,
    amount DECIMAL(10, 2),
    status VARCHAR(20),
    insurance_provider VARCHAR(50),
    FOREIGN KEY (admission_id) REFERENCES admissions(admission_id)
);

```

---

## 📂 Importing the Data

Once you’ve created the tables, you need to load the CSVs.

**If using PostgreSQL (psql):**

```sql
COPY patients FROM '/path/to/patients.csv' DELIMITER ',' CSV HEADER;
COPY doctors FROM '/path/to/doctors.csv' DELIMITER ',' CSV HEADER;
COPY admissions FROM '/path/to/admissions.csv' DELIMITER ',' CSV HEADER;
COPY billing FROM '/path/to/billing.csv' DELIMITER ',' CSV HEADER;

```

**If using MySQL:**

```sql
LOAD DATA INFILE '/path/to/patients.csv' 
INTO TABLE patients 
FIELDS TERMINATED BY ',' 
IGNORE 1 ROWS;

```

---

### The "Master View" Script

```sql
CREATE OR REPLACE VIEW v_hospital_master_dashboard AS
SELECT 
    -- Patient Info
    p.patient_id,
    p.name AS patient_name,
    p.gender,
    EXTRACT(YEAR FROM AGE(p.dob)) AS age, -- PostgreSQL syntax
    p.blood_type,

    -- Admission Info
    a.admission_id,
    a.admission_date,
    a.discharge_date,
    a.reason AS admission_reason,
    COALESCE(a.discharge_date - a.admission_date, 0) AS length_of_stay,
    
    -- Clinical Info
    d.doctor_name,
    d.department,

    -- Financial Info
    b.bill_id,
    b.amount AS billing_amount,
    b.status AS payment_status,
    b.insurance_provider,

    -- Categorical Logic for Dashboard Filters
    CASE 
        WHEN (a.discharge_date - a.admission_date) > 7 THEN 'Long Stay'
        WHEN (a.discharge_date - a.admission_date) BETWEEN 3 AND 7 THEN 'Mid Stay'
        ELSE 'Short Stay'
    END AS stay_category

FROM admissions a
JOIN patients p ON a.patient_id = p.patient_id
JOIN doctors d ON a.doctor_id = d.doctor_id
LEFT JOIN billing b ON a.admission_id = b.admission_id;

```

---
Adding a **Stored Procedure** is a fantastic "Stretch Goal" because it moves your project from "Static Analysis" (looking at the past) to "Active Engineering" (building systems).

In a real hospital, you wouldn't manually type out a bill for every patient. You’d have an automated process that triggers upon discharge. This shows recruiters you can build **automated workflows**.

### The "Auto-Billing" Stored Procedure

This procedure automatically calculates a bill based on the department and the length of stay, then inserts it into the `billing` table.

```sql
CREATE OR REPLACE PROCEDURE generate_discharge_bill(p_admission_id INT)
LANGUAGE plpgsql
AS $$
DECLARE
    v_days_stayed INT;
    v_dept_rate DECIMAL;
    v_base_cost DECIMAL;
BEGIN
    -- 1. Calculate length of stay
    SELECT (discharge_date - admission_date) INTO v_days_stayed
    FROM admissions WHERE admission_id = p_admission_id;

    -- 2. Determine daily rate based on department
    -- (e.g., Cardiology is more expensive than Pediatrics)
    SELECT 
        CASE 
            WHEN d.department = 'Cardiology' THEN 1500
            WHEN d.department = 'Neurology'  THEN 1800
            WHEN d.department = 'Oncology'   THEN 2000
            WHEN d.department = 'Emergency'  THEN 1200
            ELSE 1000 
        END INTO v_dept_rate
    FROM admissions a
    JOIN doctors d ON a.doctor_id = d.doctor_id
    WHERE a.admission_id = p_admission_id;

    -- 3. Calculate total (Base cost + daily rate)
    v_base_cost := 500 + (v_days_stayed * v_dept_rate);

    -- 4. Insert into Billing table
    INSERT INTO billing (bill_id, admission_id, amount, status, insurance_provider)
    VALUES (
        nextval('bill_id_seq'), -- Assumes you have a sequence set up
        p_admission_id, 
        v_base_cost, 
        'Unpaid', 
        'Pending Insurance'
    );

    RAISE NOTICE 'Bill generated for Admission %: $% ', p_admission_id, v_base_cost;
END;
$$;

```

---
* **Logic beyond SELECT:** It demonstrates you know how to use variables (`DECLARE`), conditional logic (`CASE`), and DML operations (`INSERT`).
* **System Design:** It shows you understand the **Business Process** (Discharge → Calculate → Bill).
* **Cleanliness:** Used a stored procedure to "simulate a real-time hospital management system."

To move this from PostgreSQL to **MySQL**, we need to adjust a few syntax rules:

1. **Variable Declaration:** MySQL uses `DECLARE` inside a `BEGIN...END` block, but variables don't use the `v_` prefix by requirement (though it's good practice).
2. **Date Math:** MySQL uses `DATEDIFF(end, start)` instead of simple subtraction.
3. **Sequences:** MySQL doesn't use `nextval`. Instead, it relies on `AUTO_INCREMENT`. Ensure your `billing` table was created with `bill_id INT AUTO_INCREMENT PRIMARY KEY`.
4. **Delimiters:** You must change the delimiter so MySQL doesn't get confused by the semicolons inside the procedure.

### MySQL Stored Procedure

```sql
DELIMITER //

CREATE PROCEDURE generate_discharge_bill(IN p_admission_id INT)
BEGIN
    DECLARE v_days_stayed INT;
    DECLARE v_dept_rate DECIMAL(10,2);
    DECLARE v_base_cost DECIMAL(10,2);

    -- 1. Calculate length of stay (DATEDIFF returns the difference in days)
    -- We use GREATEST(..., 1) to ensure at least one day is billed.
    SELECT GREATEST(DATEDIFF(discharge_date, admission_date), 1) INTO v_days_stayed
    FROM admissions 
    WHERE admission_id = p_admission_id;

    -- 2. Determine daily rate based on department
    SELECT 
        CASE 
            WHEN d.department = 'Cardiology' THEN 1500
            WHEN d.department = 'Neurology'  THEN 1800
            WHEN d.department = 'Oncology'   THEN 2000
            WHEN d.department = 'Emergency'  THEN 1200
            ELSE 1000 
        END INTO v_dept_rate
    FROM admissions a
    JOIN doctors d ON a.doctor_id = d.doctor_id
    WHERE a.admission_id = p_admission_id;

    -- 3. Calculate total (Base cost + daily rate)
    SET v_base_cost = 500 + (v_days_stayed * v_dept_rate);

    -- 4. Insert into Billing table 
    -- (Note: bill_id is omitted assuming it is AUTO_INCREMENT)
    INSERT INTO billing (admission_id, amount, status, insurance_provider)
    VALUES (
        p_admission_id, 
        v_base_cost, 
        'Unpaid', 
        'Pending Insurance'
    );

    -- 5. Output message
    SELECT CONCAT('Bill generated for Admission ', p_admission_id, ': $', v_base_cost) AS StatusMessage;

END //

DELIMITER ;

```

---

### Important Table Adjustment for MySQL
```sql
CREATE TABLE billing (
    bill_id INT AUTO_INCREMENT PRIMARY KEY, -- Automatically handles IDs
    admission_id INT,
    amount DECIMAL(10, 2),
    status VARCHAR(20),
    insurance_provider VARCHAR(50),
    FOREIGN KEY (admission_id) REFERENCES admissions(admission_id)
);

```

### How to call it in MySQL:

```sql
CALL generate_discharge_bill(5001);

```




