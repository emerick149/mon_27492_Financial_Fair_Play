# ⚽ System of Financial Fair Play (FFP) 
### Oracle 21c PL/SQL Capstone Project 

**Names:** CYIZERE SIBORUREMA Elie  
**Student ID**: 27492   
**Group**: MON  



---

## 📘 Project Description  

This capstone project implements a **System of Financial Fair Play (FFP)** using **Oracle 21c PL/SQL**. The system is inspired by UEFA’s FFP policy, ensuring that football clubs operate within their financial means. It prevents overspending, enables club financial monitoring, restricts modifications during sensitive periods (weekdays & holidays), and audits all user actions.

This project incorporates **advanced PL/SQL techniques** such as:
- Table structure modification
- Conditional logic in DML
- Window functions for analytics
- Functions and packages
- Triggers with business rules
- Autonomous auditing and user action tracking

The project simulates the backend of a real-world **Management Information System (MIS)** that supports governance in professional sports organizations.

---

## 🧱 Schema Enhancements – Table Modification (DDL)

To support compliance checks, a new column `FFP_Status` is added to the `Financial_Records` table:

```sql
ALTER TABLE Financial_Records ADD FFP_Status VARCHAR2(20);
```

This column flags each record as `'Compliant'` or `'Non-Compliant'` based on net profit. This addition is crucial for evaluating each club's financial health and enforcing FFP rules.

![image alt](https://github.com/emerick149/mon_27492_Financial_Fair_Play/blob/main/phase%20vi/operation.png) 

---

## 🛠 Data Manipulation Operations (DML)

### ✅ Mark Clubs as Compliant or Non-Compliant

This update evaluates each club’s net profit and flags their FFP status:

```sql
UPDATE Financial_Records
SET FFP_Status = CASE
    WHEN Net_Profit >= 0 THEN 'Compliant'
    ELSE 'Non-Compliant'
END;

```

![image alt](https://github.com/emerick149/mon_27492_Financial_Fair_Play/blob/main/phase%20vi/update%2Cdelete%2Cinsert.png)



### ✅ Delete Outdated Sanctions

Incorrectly assigned sanctions can be removed:

```sql
DELETE FROM Sanctions WHERE Sanction_ID = 303;
```

### ✅ Insert New Owner Investment

Captures new financial contributions from club owners:

```sql
INSERT INTO Owners_Investments VALUES (411, 1, 60000000);
```
![image alt](https://github.com/emerick149/mon_27492_Financial_Fair_Play/blob/main/phase%20v/creating%20tables.png) 
![image alt](https://github.com/emerick149/mon_27492_Financial_Fair_Play/blob/main/phase%20v/inserting%20data.png)

---

## 📊 Analytical Reporting with Window Functions

To monitor financial aggressiveness among clubs, we use a window function to calculate total transfer fees grouped by receiving club:

```sql
SELECT To_Club, SUM(Fee) OVER (PARTITION BY To_Club) AS Total_Transfer_Fees
FROM Transfers;
```
![image alt](https://github.com/cedric299/Tue_27655_Financial_Fair_Play/blob/8215587b4af97ffc04f537b668aa1ee147005efb/Phase%20VII/Simple%20Problem%20Statement%20with%20Analytic%20(Window)%20Function.png) 

This helps detect which clubs are consistently spending the most during transfer windows.

---

## 📦 Function and Package: FinancialTools

### ✅ Function: `Get_FFP_Status`

Returns a club’s current compliance status with exception handling:

```sql
CREATE OR REPLACE FUNCTION Get_FFP_Status(p_club_id NUMBER) RETURN VARCHAR2 IS
    v_status VARCHAR2(20);
BEGIN
    SELECT FFP_Status INTO v_status
    FROM Financial_Records
    WHERE Club_ID = p_club_id AND ROWNUM = 1;
    RETURN v_status;
EXCEPTION
    WHEN NO_DATA_FOUND THEN RETURN 'Unknown';
    WHEN OTHERS THEN RETURN 'Error';
END;
/
```
![image alt](https://github.com/cedric299/Tue_27655_Financial_Fair_Play/blob/8215587b4af97ffc04f537b668aa1ee147005efb/Phase%20VII/FinancialTools.png) 
![image alt](https://github.com/cedric299/Tue_27655_Financial_Fair_Play/blob/8215587b4af97ffc04f537b668aa1ee147005efb/Phase%20VII/Fetch%20Financial%20Records%20by%20Club.png)

### ✅ Package and Package Body

The `FinancialTools` package groups procedures and functions related to club finance:

```sql
CREATE OR REPLACE PACKAGE FinancialTools AS
    PROCEDURE Get_Club_Financials(p_club_id IN NUMBER);
    FUNCTION Get_FFP_Status(p_club_id NUMBER) RETURN VARCHAR2;
END FinancialTools;
/
```

```sql
CREATE OR REPLACE PACKAGE BODY FinancialTools AS
    PROCEDURE Get_Club_Financials(p_club_id IN NUMBER) IS
    BEGIN
        FOR rec IN (SELECT * FROM Financial_Records WHERE Club_ID = p_club_id) LOOP
            DBMS_OUTPUT.PUT_LINE('Year: ' || rec.Year || ', Profit: ' || rec.Net_Profit);
        END LOOP;
    EXCEPTION
        WHEN OTHERS THEN
            DBMS_OUTPUT.PUT_LINE('Error: ' || SQLERRM);
    END Get_Club_Financials;

    FUNCTION Get_FFP_Status(p_club_id NUMBER) RETURN VARCHAR2 IS
        v_status VARCHAR2(20);
    BEGIN
        SELECT FFP_Status INTO v_status
        FROM Financial_Records
        WHERE Club_ID = p_club_id AND ROWNUM = 1;
        RETURN v_status;
    EXCEPTION
        WHEN NO_DATA_FOUND THEN RETURN 'Unknown';
        WHEN OTHERS THEN RETURN 'Error';
    END Get_FFP_Status;
END FinancialTools;
/
```

---

## 📅 Holiday Restrictions and Access Controls

### ✅ Step 1: Create Holiday Dates Table

Holidays are stored in a static table:

```sql
CREATE TABLE Holidays (
    Holiday_Date DATE PRIMARY KEY,
    Description VARCHAR2(100)
);

INSERT INTO Holidays VALUES (TO_DATE('2025-06-01', 'YYYY-MM-DD'), 'Heroes Day');
INSERT INTO Holidays VALUES (TO_DATE('2025-06-15', 'YYYY-MM-DD'), 'Independence Day');
```
![image alt](https://github.com/cedric299/Tue_27655_Financial_Fair_Play/blob/8215587b4af97ffc04f537b668aa1ee147005efb/Phase%20VII/Restrict_Weekday_Holiday_Changes.png) 

### ✅ Step 2: Trigger to Restrict DML on Weekdays and Holidays

This trigger blocks all financial data modifications during workdays and public holidays:

```sql
CREATE OR REPLACE TRIGGER Restrict_Weekday_Holiday_Changes
BEFORE INSERT OR UPDATE OR DELETE ON Financial_Records
DECLARE
    v_count NUMBER;
BEGIN
    SELECT COUNT(*) INTO v_count
    FROM Holidays
    WHERE Holiday_Date = TRUNC(SYSDATE);

    IF TO_CHAR(SYSDATE, 'DY', 'NLS_DATE_LANGUAGE=ENGLISH') IN ('MON','TUE','WED','THU','FRI') 
       OR v_count > 0 THEN
        RAISE_APPLICATION_ERROR(-20001, 'DML operations are restricted on weekdays and holidays.');
    END IF;
END;
/
```
---
![image alt](https://github.com/cedric299/Tue_27655_Financial_Fair_Play/blob/8215587b4af97ffc04f537b668aa1ee147005efb/Phase%20VII/Our%20trigger%20worked.png)

## 🔍 Audit Trail and User Action Logging

### ✅ Audit Log Table

Captures user operations and access results:

```sql
CREATE TABLE Audit_Track_Log (
    Log_ID NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    User_Name VARCHAR2(50),
    Action_Type VARCHAR2(10),
    Table_Name VARCHAR2(50),
    Action_Date DATE,
    Status VARCHAR2(20),
    Description VARCHAR2(200)
);
```

### ✅ Audit Trigger for Financial_Records

Automatically logs every DML operation with status:

```sql
CREATE OR REPLACE TRIGGER Audit_Financial_Changes
AFTER INSERT OR UPDATE OR DELETE ON Financial_Records
FOR EACH ROW
DECLARE
    v_status VARCHAR2(20);
    v_action VARCHAR2(10);
    v_count NUMBER;
BEGIN
    IF INSERTING THEN v_action := 'INSERT';
    ELSIF UPDATING THEN v_action := 'UPDATE';
    ELSIF DELETING THEN v_action := 'DELETE';
    END IF;

    SELECT COUNT(*) INTO v_count
    FROM Holidays
    WHERE Holiday_Date = TRUNC(SYSDATE);

    IF TO_CHAR(SYSDATE, 'DY', 'NLS_DATE_LANGUAGE=ENGLISH') IN ('MON','TUE','WED','THU','FRI') 
       OR v_count > 0 THEN
        v_status := 'Denied';
    ELSE
        v_status := 'Allowed';
    END IF;

    INSERT INTO Audit_Track_Log (
        User_Name, Action_Type, Table_Name, Action_Date, Status, Description
    ) VALUES (
        SYS_CONTEXT('USERENV', 'SESSION_USER'),
        v_action,
        'Financial_Records',
        SYSDATE,
        v_status,
        'User attempted ' || v_action || ' on Financial_Records'
    );
END;
/
```
![image alt](https://github.com/cedric299/Tue_27655_Financial_Fair_Play/blob/8215587b4af97ffc04f537b668aa1ee147005efb/Phase%20VII/Auditing.png)

---

## 🧪 Testing and Validation

Below are sample tests to ensure the trigger and audit mechanisms work properly:

### ✅ Check If Today Is Restricted
```sql
SELECT TO_CHAR(SYSDATE, 'DAY', 'NLS_DATE_LANGUAGE=ENGLISH') AS Today FROM DUAL;
SELECT * FROM Holidays WHERE Holiday_Date = TRUNC(SYSDATE);
```

### ✅ Temporarily Disable the Trigger (for testing)
```sql
ALTER TRIGGER Restrict_Weekday_Holiday_Changes DISABLE;
```

### ✅ Insert Sample Data
```sql
INSERT INTO Financial_Records (Record_ID, Club_ID, Year, Revenue, Expenses, Net_Profit, FFP_Status)
VALUES (999, 1, 2025, 300000000, 250000000, 50000000, 'Compliant');
```

### ✅ Enable the Trigger Again
```sql
ALTER TRIGGER Restrict_Weekday_Holiday_Changes ENABLE;
```

### ✅ View Audit Log
```sql
SELECT * FROM Audit_Track_Log ORDER BY Action_Date DESC;
```

---

## ✅ Conclusion

The **Financial Fair Play Enforcement System** provides a robust, rule-based PL/SQL solution that enforces financial discipline, prevents unauthorized changes, and maintains a secure audit trail. It showcases advanced database techniques, such as:

- DDL and DML scripting
- PL/SQL functions and packages
- Triggers with business logic
- Holiday and weekday-based access restrictions
- Audit logging with autonomous transaction

The system supports real-world use cases for MIS in sports governance and financial compliance.

> **"Discipline is the bridge between goals and accomplishment." – Jim Rohn**

---
