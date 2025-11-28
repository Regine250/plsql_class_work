## 📘 README — AUCA Access System Control & Hospital Management System

This project implements system access restrictions, audit logging, and a Hospital Management Package using Oracle SQL and PL/SQL. It includes user and employee management, security triggers, violation logging, and a robust package for managing hospital patient data with bulk processing.

## 📂 Contents

Project Overview

Database Schema

Users Table

Employees Table

Access Violation Log

Access Control Triggers

Logging Trigger

Testing Access Restrictions

Hospital Management System

Patients & Doctors Tables

Hospital Package (Specification & Body)

Testing the Package

Conclusion



## 1. Project Overview

This project enforces strictly‐defined AUCA access policies:

Access Rules

❌ No system access on Saturday or Sunday (Sabbath)

❌ No access outside 8:00 AM – 5:00 PM (Monday–Friday)

✔ Allowed operations (INSERT/UPDATE/DELETE) during permitted hours

All violations are automatically blocked and logged

Additionally, the project includes a Hospital Management System with:

Bulk patient loading

Counting admitted patients

Returning patient lists using a cursor

Admitting individual patients

##  2. Database Schema
2.1 Users Table

Stores system users and roles.

CREATE TABLE users (
    user_id      NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    username     VARCHAR2(50) UNIQUE NOT NULL,
    password     VARCHAR2(50) NOT NULL,
    role         VARCHAR2(20),
    date_created DATE DEFAULT SYSDATE
);

Sample Data
INSERT INTO users (username, password, role) VALUES ('jdoe', 'pass123', 'Admin');
INSERT INTO users (username, password, role) VALUES ('mbenjamin', 'mb234', 'Staff');
INSERT INTO users (username, password, role) VALUES ('regine', 'rg567', 'Manager');

## 2.2 Employees Table

Stores employee details.

CREATE TABLE employees (
    emp_id        NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    full_name     VARCHAR2(100) NOT NULL,
    department    VARCHAR2(50),
    salary        NUMBER(10,2),
    hire_date     DATE DEFAULT SYSDATE
);

Sample Data
INSERT INTO employees (full_name, department, salary) VALUES ('John Kamali', 'IT', 850000);
INSERT INTO employees (full_name, department, salary) VALUES ('Sarah Mukamana', 'Finance', 920000);
INSERT INTO employees (full_name, department, salary) VALUES ('Peter Ndayisaba', 'HR', 780000);

## 2.3 Access Violation Log

Captures details of blocked user actions.

CREATE TABLE access_violation_log (
    log_id        NUMBER GENERATED ALWAYS AS IDENTITY,
    username      VARCHAR2(50),
    table_name    VARCHAR2(50),
    action_type   VARCHAR2(20),
    attempted_on  DATE,
    details       VARCHAR2(200)
);

## 3. Access Control Triggers
3.1 Trigger: Restrict Access on EMPLOYEES Table

Blocks INSERT/UPDATE/DELETE outside allowed hours.

CREATE OR REPLACE TRIGGER trg_enforce_access_policy
BEFORE INSERT OR UPDATE OR DELETE ON employees
FOR EACH ROW
DECLARE
    v_day  VARCHAR2(10);
    v_hour NUMBER;
BEGIN
    v_day  := TO_CHAR(SYSDATE, 'DY', 'NLS_DATE_LANGUAGE=ENGLISH');
    v_hour := TO_NUMBER(TO_CHAR(SYSDATE, 'HH24'));

    IF v_day IN ('SAT', 'SUN') THEN
        RAISE_APPLICATION_ERROR(-20001, 'ACCESS BLOCKED: No access allowed on weekends.');
    END IF;

    IF v_hour < 8 OR v_hour >= 17 THEN
        RAISE_APPLICATION_ERROR(-20002,
            'ACCESS BLOCKED: Allowed time is Monday–Friday, 08:00 to 17:00.');
    END IF;
END;
/

## 3.2 Trigger: Restrict Access on USERS Table

Same policy applied to user modifications.

CREATE OR REPLACE TRIGGER trg_restrict_access_users
BEFORE INSERT OR UPDATE OR DELETE ON users
DECLARE
    v_day VARCHAR2(10);
    v_hour NUMBER;
BEGIN
    v_day  := TO_CHAR(SYSDATE, 'DY', 'NLS_DATE_LANGUAGE=ENGLISH');
    v_hour := TO_NUMBER(TO_CHAR(SYSDATE, 'HH24'));

    IF v_day IN ('SAT', 'SUN') THEN
        RAISE_APPLICATION_ERROR(-20001, 'Access not allowed on weekends.');
    END IF;

    IF v_hour < 8 OR v_hour >= 17 THEN
        RAISE_APPLICATION_ERROR(-20002,
            'Access allowed only between 8 AM and 5 PM.');
    END IF;
END;
/

## 4. Trigger for Logging Violations

This trigger logs only access-related errors.

CREATE OR REPLACE TRIGGER trg_log_violations
AFTER SERVERERROR ON DATABASE
DECLARE
    v_username VARCHAR2(50);
    v_error_code NUMBER;
BEGIN
    v_username := SYS_CONTEXT('USERENV', 'SESSION_USER');
    v_error_code := ORA_SERVER_ERROR(1);

    IF v_error_code IN (-20001, -20002) THEN
        INSERT INTO access_violation_log (
            username, table_name, action_type, attempted_on, details
        )
        VALUES (
            v_username,
            ORA_SERVER_ERROR_MSG(1),
            'DML Action',
            SYSDATE,
            'Access denied. Error: ' || v_error_code
        );
    END IF;
END;
/

## 5. Testing Access Restrictions
Check data
SELECT * FROM employees;

Attempt a restricted update
UPDATE employees SET salary = 900000 WHERE emp_id = 2;

Check logged violations
SELECT * FROM access_violation_log;

## 6. Hospital Management System
6.1 Patients Table
CREATE TABLE patients (
    patient_id     NUMBER PRIMARY KEY,
    patient_name   VARCHAR2(50),
    age            NUMBER,
    gender         VARCHAR2(10),
    admitted       VARCHAR2(3) DEFAULT 'NO'
);

## 6.2 Doctors Table
CREATE TABLE doctors (
    doctor_id      NUMBER PRIMARY KEY,
    doctor_name    VARCHAR2(50),
    specialty      VARCHAR2(50)
);

## 6.3 Hospital Package (Specification)
CREATE OR REPLACE PACKAGE hospital_pkg IS
    
    TYPE patient_rec IS RECORD (
        patient_id     NUMBER,
        patient_name   VARCHAR2(50),
        age            NUMBER,
        gender         VARCHAR2(10),
        admitted       VARCHAR2(3)
    );

    TYPE patient_table_type IS TABLE OF patient_rec;

    PROCEDURE bulk_load_patients(p_patients IN patient_table_type);
    
    FUNCTION show_all_patients RETURN SYS_REFCURSOR;
    
    FUNCTION count_admitted RETURN NUMBER;

    PROCEDURE admit_patient(p_id IN NUMBER);

END hospital_pkg;
/

## 6.4 Hospital Package (Body)

Includes bulk inserts, cursor return, counting, and admitting.

CREATE OR REPLACE PACKAGE BODY hospital_pkg IS

    PROCEDURE bulk_load_patients(p_patients IN patient_table_type) IS
    BEGIN
        FORALL i IN 1 .. p_patients.COUNT
            INSERT INTO patients(patient_id, patient_name, age, gender, admitted)
            VALUES (
                p_patients(i).patient_id,
                p_patients(i).patient_name,
                p_patients(i).age,
                p_patients(i).gender,
                p_patients(i).admitted
            );
        COMMIT;
    END bulk_load_patients;

    FUNCTION show_all_patients RETURN SYS_REFCURSOR IS
        rc SYS_REFCURSOR;
    BEGIN
        OPEN rc FOR SELECT * FROM patients;
        RETURN rc;
    END show_all_patients;

    FUNCTION count_admitted RETURN NUMBER IS
        v_count NUMBER;
    BEGIN
        SELECT COUNT(*) INTO v_count
        FROM patients
        WHERE admitted = 'YES';

        RETURN v_count;
    END count_admitted;

    PROCEDURE admit_patient(p_id IN NUMBER) IS
    BEGIN
        UPDATE patients
        SET admitted = 'YES'
        WHERE patient_id = p_id;

        COMMIT;
    END admit_patient;

END hospital_pkg;
/

## 6.5 Testing the Hospital Package
Bulk Load
DECLARE
    patient_list hospital_pkg.patient_table_type :=
        hospital_pkg.patient_table_type();
BEGIN
    patient_list.EXTEND(3);

    patient_list(1).patient_id := 1;
    patient_list(1).patient_name := 'Alice';
    patient_list(1).age := 30;
    patient_list(1).gender := 'F';
    patient_list(1).admitted := 'NO';

    patient_list(2).patient_id := 2;
    patient_list(2).patient_name := 'Bob';
    patient_list(2).age := 45;
    patient_list(2).gender := 'M';
    patient_list(2).admitted := 'NO';

    patient_list(3).patient_id := 3;
    patient_list(3).patient_name := 'Clara';
    patient_list(3).age := 27;
    patient_list(3).gender := 'F';
    patient_list(3).admitted := 'NO';

    hospital_pkg.bulk_load_patients(patient_list);
END;
/

Show all patients
VARIABLE my_cursor REFCURSOR;

BEGIN
    :my_cursor := hospital_pkg.show_all_patients;
END;
/

PRINT my_cursor;

Admit a patient
BEGIN
    hospital_pkg.admit_patient(2);
END;

Count admitted patients
SET SERVEROUTPUT ON;

BEGIN
    DBMS_OUTPUT.PUT_LINE(
        'Number of admitted patients: ' || hospital_pkg.count_admitted
    );
END;
/

## 7. Conclusion

This project demonstrates:

✔ Time-based and day-based access restriction using triggers
✔ Automatic auditing of violations
✔ A fully functional PL/SQL package for managing hospital patient data
✔ Bulk processing using collections
✔ Secure and controlled database operations.
