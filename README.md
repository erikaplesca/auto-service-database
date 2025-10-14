# auto-service-database

# 🚗 Auto Service Management Database System

A comprehensive relational database system for managing automotive service center operations, developed in **Oracle 19c** using **SQL Developer**.

![Oracle](https://img.shields.io/badge/Oracle-19c-F80000?style=for-the-badge&logo=oracle&logoColor=white)
![SQL](https://img.shields.io/badge/SQL-PL%2FSQL-336791?style=for-the-badge&logo=postgresql&logoColor=white)
![Database](https://img.shields.io/badge/Database-Relational-4479A1?style=for-the-badge&logo=database&logoColor=white)

---

## 📋 Table of Contents

- [Project Overview](#-project-overview)
- [Key Features](#-key-features)
- [Database Architecture](#-database-architecture)
- [Technical Achievements](#-technical-achievements)
- [Advanced SQL Examples](#-advanced-sql-examples)
- [Database Schema](#️-database-schema)
- [Installation & Setup](#-installation--setup)
- [Learning Outcomes](#-learning-outcomes)

---

## 🎯 Project Overview

This project implements a complete relational database for an auto repair shop, covering the entire workflow from customer scheduling to service completion and invoicing. The system handles both individual and corporate clients, manages repair processes, tracks parts inventory, and automates billing.

**Core Functionality:**
- Client and vehicle management
- Appointment scheduling
- Cost estimation and quotes
- Repair process tracking
- Parts inventory management
- Automated invoice generation
- Employee management with role-based access

---

## ⭐ Key Features

### 🏗️ Database Design Excellence

- ✅ **Fully Normalized**: Complete normalization to 3NF (Third Normal Form)
- ✅ **Complex Relationships**: Multiple relationship types including:
  - Entity specialization (CLIENT → PERSOANA_FIZICA / PERSOANA_JURIDICA)
  - Many-to-many relationships via associative tables
  - Ternary relationships (MECANIC - PROCES_REP - SERVICIU)
- ✅ **Comprehensive Schema**: 17 interconnected tables
- ✅ **Data Integrity**: Foreign keys, check constraints, and unique constraints

### 💼 Business Logic Implementation

#### 1. **Client Management**
```sql
-- Supports both individual and corporate clients
-- CNP validation for individuals, CUI validation for companies
CREATE TABLE PERSOANA_FIZICA (
    cnp CHAR(13) UNIQUE NOT NULL,
    CONSTRAINT marime_cnp CHECK (LENGTH(cnp) = 13)
);
2. Smart Quote System
* Automatic quote generation based on appointments
* Service breakdown with detailed cost estimation
* Status tracking: In elaborare → Trimis → Acceptat / Respins
3. Intelligent Mechanic Assignment
-- Round-robin distribution ensures even workload across mechanics
-- Mechanics assigned alphabetically based on specialization
WITH MECANICI_ORD AS (
    SELECT id_angajat, tip_mecanic,
        ROW_NUMBER() OVER (PARTITION BY tip_mecanic 
                          ORDER BY nume, prenume) AS rn
    FROM MECANIC JOIN ANGAJATI USING(id_angajat)
)
-- Assignment logic using MOD for round-robin distribution
4. Automated Invoice Generation
-- Invoices created only when ALL services are completed
-- Dynamic cost calculation: services + parts (with supplier discounts)
-- Payment terms: 7 days (individuals) / 30 days (companies)
INSERT INTO FACTURA (...)
SELECT 
    ...,
    -- Cost calculation with supplier discounts
    SUM(s.cost) + SUM(p.pret_standard * (1 - f.procent_reducere)),
    -- Dynamic payment terms
    CASE 
        WHEN pf.id_client IS NOT NULL THEN SYSDATE + 7
        WHEN pj.id_client IS NOT NULL THEN SYSDATE + 30
    END AS data_scadenta
FROM ...
WHERE NOT EXISTS (
    SELECT 1 FROM SERVICIU_PROCES 
    WHERE id_proces = pr.id_proces AND status <> 'Finalizat'
);

🏛️ Database Architecture
Entity-Relationship Model
CLIENT (1) ──< MASINA (M)
   |            |
   ├─> PERSOANA_FIZICA       MASINA ──> PROGRAMARE ──> DEVIZ
   └─> PERSOANA_JURIDICA        |                        |
                                |                   PROCES_REP ──> FACTURA
ANGAJATI                        |                        |
   |                            └────────────────────────┘
   ├─> MECANIC ──> ANGAJATI_RESPONSABILI
   └─> CONTABIL ──────┘              |
                              SERVICIU_PROCES ──> PIESE_UTILIZATE
                                      |                  |
                                  SERVICIU           PIESA ──> FURNIZOR
Core Tables
Table	Records	Purpose
CLIENT	8	Base customer entity
PERSOANA_FIZICA	4	Individual clients with CNP
PERSOANA_JURIDICA	4	Corporate clients with CUI
MASINA	10	Vehicle registry
PROGRAMARE	10	Appointment scheduling
DEVIZ	10	Cost estimates
SERVICIU	30	Service catalog (3 vehicle types)
PROCES_REP	5	Repair processes
ANGAJATI	15	Employees (5 accountants, 10 mechanics)
FACTURA	5	Generated invoices
PIESA	30	Parts catalog
FURNIZOR	6	Supplier management
💡 Technical Achievements
🔍 Advanced SQL Techniques
1. Complex Synchronized Subqueries
-- Find mechanics working on 'Turism' cars owned by individuals
-- with above-average repair count
SELECT DISTINCT a.nume, a.prenume, m.tip_mecanic
FROM ANGAJATI a
JOIN MECANIC m ON a.id_angajat = m.id_angajat
WHERE EXISTS (
    SELECT 1 FROM ANGAJATI_RESPONSABILI ar
    JOIN SERVICIU_PROCES sp ON ar.id_proces = sp.id_proces
    JOIN PROCES_REP pr ON sp.id_proces = pr.id_proces
    JOIN DEVIZ d ON pr.id_deviz = d.id_deviz
    JOIN PROGRAMARE p ON d.id_programare = p.id_programare
    JOIN MASINA ma ON p.id_masina = ma.id_masina
    JOIN PERSOANA_FIZICA pf ON ma.id_client = pf.id_client
    WHERE ar.id_angajat = a.id_angajat
      AND ma.tip_masina = 'Turism'
      AND (SELECT COUNT(*) FROM ANGAJATI_RESPONSABILI 
           WHERE id_angajat = a.id_angajat) > 
          (SELECT AVG(COUNT(*)) FROM ANGAJATI_RESPONSABILI 
           GROUP BY id_angajat)
);
2. CTE (Common Table Expressions)
-- Services that never required parts
WITH servicii_fara_piese AS (
    SELECT sp.id_serviciu
    FROM SERVICIU_PROCES sp
    LEFT JOIN PIESE_UTILIZATE pu 
        ON sp.id_proces = pu.id_proces 
        AND sp.id_serviciu = pu.id_serviciu
    WHERE pu.id_piesa IS NULL
    GROUP BY sp.id_serviciu
)
SELECT s.nume_serviciu, COUNT(sp.id_serviciu) AS nr_aparitii
FROM SERVICIU_PROCES sp
JOIN servicii_fara_piese sfp ON sp.id_serviciu = sfp.id_serviciu
JOIN SERVICIU s ON sp.id_serviciu = s.id_serviciu
GROUP BY s.nume_serviciu
ORDER BY nr_aparitii DESC;
3. Window Functions & Analytics
-- Service completion analysis with CASE expressions
SELECT
    UPPER(TRIM(a.nume || ' ' || a.prenume)) AS mecanic_complet,
    TRUNC(sp.data_incepere) AS zi_incepere,
    TRUNC(sp.data_finalizare) AS zi_finalizare,
    TRUNC(sp.data_finalizare) - TRUNC(sp.data_incepere) AS zile_lucrate,
    CASE
        WHEN TRUNC(sp.data_finalizare) - TRUNC(sp.data_incepere) <= 3 
            THEN 'Rapid'
        WHEN TRUNC(sp.data_finalizare) - TRUNC(sp.data_incepere) BETWEEN 4 AND 7 
            THEN 'Normal'
        ELSE 'Întârziat'
    END AS status_termen
FROM SERVICIU_PROCES sp
WHERE sp.status = 'Finalizat'
  AND sp.data_finalizare >= TRUNC(SYSDATE) - 60;
4. Outer Join Operations (4+ tables)
-- Complete vehicle history including clients without appointments
SELECT
    m.nr_inmatriculare,
    NVL(pf.nume, pj.denumire) AS nume_client,
    prog.data_ora AS data_programare,
    pr.id_proces,
    p.denumire AS piesa_folosita
FROM MASINA m
LEFT OUTER JOIN CLIENT c ON m.id_client = c.id_client
LEFT OUTER JOIN PERSOANA_FIZICA pf ON c.id_client = pf.id_client
LEFT OUTER JOIN PERSOANA_JURIDICA pj ON c.id_client = pj.id_client
LEFT OUTER JOIN PROGRAMARE prog ON m.id_masina = prog.id_masina
LEFT OUTER JOIN DEVIZ d ON prog.id_programare = d.id_programare
LEFT OUTER JOIN PROCES_REP pr ON d.id_deviz = pr.id_deviz
LEFT OUTER JOIN PIESE_UTILIZATE pu ON pr.id_proces = pu.id_proces
LEFT OUTER JOIN PIESA p ON pu.id_piesa = p.id_piesa;
5. Division Operation
-- Mechanics responsible for ALL services in their assigned processes
SELECT DISTINCT a.id_angajat, a.nume, a.prenume
FROM ANGAJATI a
JOIN MECANIC m ON a.id_angajat = m.id_angajat
JOIN ANGAJATI_RESPONSABILI ar ON a.id_angajat = ar.id_angajat
WHERE NOT EXISTS (
    SELECT 1 FROM SERVICIU_PROCES sp
    WHERE sp.id_proces = ar.id_proces
      AND NOT EXISTS (
          SELECT 1 FROM ANGAJATI_RESPONSABILI ar2
          WHERE ar2.id_angajat = a.id_angajat
            AND ar2.id_proces = sp.id_proces
            AND ar2.id_serviciu = sp.id_serviciu
      )
);
6. Top-N Analysis
-- Most expensive repair jobs
SELECT * FROM (
    SELECT
        m.nr_inmatriculare,
        NVL(pf.nume, pj.denumire) AS nume_client,
        f.cost_total,
        f.data_emitere
    FROM FACTURA f
    JOIN MASINA m ON f.id_masina = m.id_masina
    JOIN CLIENT c ON f.id_client = c.id_client
    LEFT JOIN PERSOANA_FIZICA pf ON c.id_client = pf.id_client
    LEFT JOIN PERSOANA_JURIDICA pj ON c.id_client = pj.id_client
    ORDER BY f.cost_total DESC
)
WHERE ROWNUM <= 3;
🛡️ Data Integrity Features
-- Referential integrity with cascading
CONSTRAINT fk_proces_factura FOREIGN KEY (id_factura)
    REFERENCES FACTURA(id_factura)
    ON DELETE SET NULL

-- Domain constraints
CONSTRAINT check_status CHECK (status IN ('Platita', 'Neplatita'))
CONSTRAINT check_salariu CHECK (salariu >= 0)

-- Business rule constraints
CONSTRAINT check_cnp CHECK (LENGTH(cnp) = 13)
CONSTRAINT check_cui CHECK (LENGTH(cui) = 8)

📊 Advanced SQL Examples
Query Complexity Showcase
The project includes 15+ complex queries demonstrating:
Feature	Implementation	Query Count
Multi-table Joins	4-8 tables per query	5
Subqueries	Synchronized & Unsynchronized	8
Aggregations	GROUP BY with HAVING	4
Window Functions	ROW_NUMBER, COUNT OVER	3
String Functions	UPPER, TRIM, CONCAT	5
Date Functions	TRUNC, date arithmetic	4
Conditional Logic	CASE, DECODE, NVL	6
Set Operations	Division, Top-N	2
Sample: Complex Aggregation with Subqueries
-- Clients with above-average parts expenses
SELECT
    c.id_client,
    DECODE(NVL(pf.id_client, 0), 
           0, 'Persoana Juridica', 
           'Persoana Fizica') AS tip_client,
    NVL(COUNT(DISTINCT m.id_masina), 0) AS nr_masini,
    SUM(NVL(p.pret_standard, 0)) AS total_piese
FROM CLIENT c
LEFT JOIN MASINA m ON c.id_client = m.id_client
LEFT JOIN PROGRAMARE prog ON m.id_masina = prog.id_masina
LEFT JOIN DEVIZ d ON prog.id_programare = d.id_programare
LEFT JOIN PROCES_REP pr ON d.id_deviz = pr.id_deviz
LEFT JOIN PIESE_UTILIZATE pu ON pr.id_proces = pu.id_proces
LEFT JOIN PIESA p ON pu.id_piesa = p.id_piesa
LEFT JOIN PERSOANA_FIZICA pf ON c.id_client = pf.id_client
GROUP BY c.id_client, pf.id_client
HAVING SUM(NVL(p.pret_standard, 0)) > (
    SELECT AVG(val_total) FROM (
        SELECT SUM(NVL(p2.pret_standard, 0)) AS val_total
        FROM CLIENT c2
        LEFT JOIN MASINA m2 ON c2.id_client = m2.id_client
        LEFT JOIN PROGRAMARE prog2 ON m2.id_masina = prog2.id_masina
        LEFT JOIN DEVIZ d2 ON prog2.id_programare = d2.id_programare
        LEFT JOIN PROCES_REP pr2 ON d2.id_deviz = pr2.id_deviz
        LEFT JOIN PIESE_UTILIZATE pu2 ON pr2.id_proces = pu2.id_proces
        LEFT JOIN PIESA p2 ON pu2.id_piesa = p2.id_piesa
        GROUP BY c2.id_client
    )
)
ORDER BY total_piese DESC;

🗄️ Database Schema
Complete Table Structure
-- CLIENT (Base Entity)
CREATE TABLE CLIENT (
    id_client NUMBER PRIMARY KEY,
    adresa VARCHAR2(100) NOT NULL,
    telefon VARCHAR2(15) NOT NULL,
    email VARCHAR2(50)
);

-- PERSOANA_FIZICA (Specialization)
CREATE TABLE PERSOANA_FIZICA (
    id_client NUMBER PRIMARY KEY,
    nume VARCHAR2(30) NOT NULL,
    prenume VARCHAR2(30) NOT NULL,
    cnp CHAR(13) UNIQUE NOT NULL,
    CONSTRAINT fk_pf_client FOREIGN KEY (id_client) 
        REFERENCES CLIENT(id_client),
    CONSTRAINT check_cnp CHECK (LENGTH(cnp) = 13)
);

-- PROCES_REP (Repair Process)
CREATE TABLE PROCES_REP (
    id_proces NUMBER PRIMARY KEY,
    id_factura NUMBER,
    id_deviz NUMBER NOT NULL,
    CONSTRAINT fk_proces_factura FOREIGN KEY (id_factura)
        REFERENCES FACTURA(id_factura) ON DELETE SET NULL,
    CONSTRAINT fk_proces_deviz FOREIGN KEY (id_deviz)
        REFERENCES DEVIZ(id_deviz)
);

-- ANGAJATI_RESPONSABILI (Ternary Relationship)
CREATE TABLE ANGAJATI_RESPONSABILI (
    id_angajat NUMBER NOT NULL,
    id_proces NUMBER NOT NULL,
    id_serviciu NUMBER NOT NULL,
    PRIMARY KEY (id_angajat, id_proces, id_serviciu),
    CONSTRAINT fk_ar_angajat FOREIGN KEY (id_angajat)
        REFERENCES MECANIC(id_angajat),
    CONSTRAINT fk_ar_proces_serviciu 
        FOREIGN KEY (id_proces, id_serviciu)
        REFERENCES SERVICIU_PROCES(id_proces, id_serviciu)
);
Sequences for Auto-Increment
CREATE SEQUENCE SEQ_CLIENT START WITH 1 INCREMENT BY 1;
CREATE SEQUENCE SEQ_MASINA START WITH 1 INCREMENT BY 1;
CREATE SEQUENCE SEQ_PROGRAMARE START WITH 1 INCREMENT BY 1;
CREATE SEQUENCE SEQ_DEVIZ START WITH 1 INCREMENT BY 1;
CREATE SEQUENCE SEQ_SERVICIU START WITH 1 INCREMENT BY 1;
CREATE SEQUENCE SEQ_ANGAJATI START WITH 1 INCREMENT BY 1;
CREATE SEQUENCE SEQ_PROCES_REP START WITH 1 INCREMENT BY 1;
CREATE SEQUENCE SEQ_FACTURA START WITH 1 INCREMENT BY 1;
CREATE SEQUENCE SEQ_PIESA START WITH 1 INCREMENT BY 1;
CREATE SEQUENCE SEQ_FURNIZOR START WITH 1 INCREMENT BY 1;
Complex View
CREATE OR REPLACE VIEW v_client_masini_piese AS
SELECT
    c.id_client,
    NVL(pf.nume, pj.denumire) AS nume_client,
    m.nr_inmatriculare,
    SUM(NVL(p.pret_standard, 0)) AS total_piese
FROM CLIENT c
LEFT JOIN PERSOANA_FIZICA pf ON c.id_client = pf.id_client
LEFT JOIN PERSOANA_JURIDICA pj ON c.id_client = pj.id_client
LEFT JOIN MASINA m ON c.id_client = m.id_client
LEFT JOIN PROGRAMARE prog ON m.id_masina = prog.id_masina
LEFT JOIN DEVIZ d ON prog.id_programare = d.id_programare
LEFT JOIN PROCES_REP pr ON d.id_deviz = pr.id_deviz
LEFT JOIN PIESE_UTILIZATE pu ON pr.id_proces = pu.id_proces
LEFT JOIN PIESA p ON pu.id_piesa = p.id_piesa
GROUP BY c.id_client, NVL(pf.nume, pj.denumire), m.nr_inmatriculare;

🚀 Installation & Setup
Prerequisites
* Oracle Database 19c or higher (Enterprise Edition recommended)
* SQL Developer or SQL*Plus
* Minimum 2GB RAM for database instance
Installation Steps
1. Clone the repository
git clone https://github.com/yourusername/auto-service-db.git
cd auto-service-db
2. Connect to Oracle Database
sqlplus username/password@localhost:1521/ORCL
3. Create sequences
@133_Plesca_Maria-Erika-creare_inserare.txt
4. Verify installation
SELECT COUNT(*) FROM CLIENT;  -- Should return 8
SELECT COUNT(*) FROM MASINA;  -- Should return 10
SELECT COUNT(*) FROM SERVICIU;  -- Should return 30
5. Run sample queries
@133_Plesca_Maria-Erika-exemple.txt

📈 Database Statistics
Metric	Value
Total Tables	17
Total Sequences	10
Total Views	1
Total Records	150+
Foreign Keys	20+
Check Constraints	15+
Complex Queries	15+
Normalization Level	3NF
🎓 Learning Outcomes
This project demonstrates comprehensive understanding of:
Database Design
* ✅ Entity-Relationship modeling
* ✅ Normalization (1NF → 2NF → 3NF → BCNF)
* ✅ Constraint design and implementation
* ✅ Specialization and generalization
SQL Proficiency
* ✅ Complex multi-table joins
* ✅ Subqueries (synchronized and unsynchronized)
* ✅ Aggregate functions and GROUP BY
* ✅ Window functions and analytics
* ✅ Common Table Expressions (CTE)
* ✅ Set operations (UNION, INTERSECT, MINUS)
Oracle-Specific Features
* ✅ Sequences for auto-increment
* ✅ DECODE and NVL functions
* ✅ Hierarchical queries
* ✅ Advanced date functions
* ✅ Constraint management
* ✅ View creation and management
Business Logic
* ✅ Automated data generation
* ✅ Dynamic pricing calculations
* ✅ Role-based access control
* ✅ Process automation
* ✅ Data validation rules

📁 Project Structure
auto-service-db/
│
├── 133_Plesca_Maria-Erika-proiect.docx    # Complete project documentation
├── 133_Plesca_Maria-Erika-creare_inserare.txt  # DDL and DML scripts
├── 133_Plesca_Maria-Erika-exemple.txt     # Complex query examples
│
├── diagrams/
│   ├── entity-relationship-diagram.png
│   └── conceptual-diagram.png
│
└── README.md

🔍 Key Highlights
🎯 Smart Features
1. Automated Mechanic Assignment
    * Round-robin distribution ensures balanced workload
    * Mechanics assigned by specialization and alphabetically
    * Prevents overloading individual employees
2. Dynamic Invoice Calculation
    * Automatic summation of service costs
    * Supplier discounts applied to parts
    * Payment terms based on client type
3. Intelligent Parts Linking
    * Automated association of parts to services
    * Based on service requirements
    * Prevents manual data entry errors
4. Process Automation
    * Automatic process creation from approved quotes
    * Status tracking at multiple levels
    * Invoice generation only when work is complete
📊 Data Quality
* Referential Integrity: All relationships enforced via foreign keys
* Domain Validation: CHECK constraints on all critical fields
* Business Rules: Implemented via constraints and triggers
* Audit Trail: Timestamp tracking for all major operations

📝 Documentation
The project includes comprehensive documentation:
* 70+ pages of detailed analysis and implementation
* 17 diagrams (ER diagrams, conceptual models, normalization examples)
* 15+ complex SQL queries with explanations
* Complete data dictionary for all tables and columns
* Normalization examples (1NF, 2NF, 3NF, BCNF)

🏆 Project Achievements
* ✅ Complex Schema: 17 interconnected tables with proper relationships
* ✅ Advanced SQL: 15+ complex queries showcasing various techniques
* ✅ Business Logic: Automated workflows and intelligent data processing
* ✅ Data Integrity: Comprehensive constraint system
* ✅ Normalization: Full normalization to 3NF with BCNF analysis
* ✅ Real-world Application: Practical solution for actual business needs

👨‍💻 Author
Maria-Erika Pleșca Group 133 Faculty of Computer Science, Year 1 Database Fundamentals Course

📅 Academic Year
2024-2025, Semester 2

📄 License
This is an academic project developed for educational purposes.

🙏 Acknowledgments
* Course Instructor: Lect.dr. Letiția Marin
* Oracle Documentation and Community
* Parents' auto service business for inspiration and real-world insights


