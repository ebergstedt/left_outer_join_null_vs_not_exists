# SQL Server performance - Get all NULL values - LEFT OUTER JOIN .. NULL vs NOT EXISTS

# About

This document outlines a performance benchmark on selecting all values from a larger table, filtered by a smaller table, where no joined values exists.

It compares LEFT OUTER JOIN with NULL including, versus NOT EXISTS, with index and no index on the joined column for both tables.

# Specs

**Database**: SQL Server 2012 R2

# Results

Winner is marked in **bold**.

### Run 1

| Type            | Index | CPU Time (ms) | Elapsed Time (ms) |
|-----------------|-------|---------------|-------------------|
| LEFT OUTER JOIN | No    | 78            | 138               |
| NOT EXISTS      | No    | 62            | 86                |
| LEFT OUTER JOIN | Yes   | 78            | 119               |
| NOT EXISTS      | Yes   | **32**        | **38**            |

### Run 2

| Type            | Index | CPU Time (ms) | Elapsed Time (ms) |   |
|-----------------|-------|---------------|-------------------|---|
| LEFT OUTER JOIN | No    | 78            | 158               |   |
| NOT EXISTS      | No    | 63            | 102               |   |
| LEFT OUTER JOIN | Yes   | 62            | 147               |   |
| NOT EXISTS      | Yes   | **47**            | **61**                |   |

### Run 3



# Script

```sql
-- ##
-- ## LEFT JOIN [...] NULL 
-- ## vs
-- ## NOT EXISTS
-- ##

-- ##
-- ## BASELINE
-- ##
IF EXISTS (SELECT * FROM INFORMATION_SCHEMA.TABLES WHERE TABLE_NAME = 'BigTable' AND TABLE_SCHEMA = 'TestPerfSchema')
BEGIN
 DROP TABLE TestPerfSchema.BigTable
END
GO

IF EXISTS (SELECT * FROM INFORMATION_SCHEMA.TABLES WHERE TABLE_NAME = 'SmallerTable' AND TABLE_SCHEMA = 'TestPerfSchema')
BEGIN
 DROP TABLE TestPerfSchema.SmallerTable
END
GO

IF EXISTS (SELECT * FROM sys.schemas WHERE name = 'TestPerfSchema')
BEGIN
 DROP SCHEMA TestPerfSchema 
END
GO

CREATE SCHEMA TestPerfSchema
GO

CREATE TABLE TestPerfSchema.BigTable (
	Id INT IDENTITY PRIMARY KEY,
	SomeColumn CHAR(4) NOT NULL,
)
 
CREATE TABLE TestPerfSchema.SmallerTable (
	Id INT IDENTITY PRIMARY KEY,
	LookupColumn CHAR(4) NOT NULL,
)
 
-- ##
-- ## INSERT DATA
-- ##
INSERT INTO 
 TestPerfSchema.BigTable (SomeColumn)
	SELECT TOP 
  250000
	CHAR(65+FLOOR(RAND(a.column_id *5645 + b.object_id)*10)) + CHAR(65+FLOOR(RAND(b.column_id *3784 + b.object_id)*12)) +
	CHAR(65+FLOOR(RAND(b.column_id *6841 + a.object_id)*12)) + CHAR(65+FLOOR(RAND(a.column_id *7544 + b.object_id)*8))
	FROM 
  master.sys.columns AS a 
  CROSS JOIN 
  master.sys.columns AS b
 
INSERT INTO TestPerfSchema.SmallerTable (LookupColumn)
	SELECT DISTINCT SomeColumn
	FROM TestPerfSchema.BigTable TABLESAMPLE (25 PERCENT)


-- ##
-- ## NO INDEX
-- ##


SET STATISTICS TIME ON
PRINT 'LEFT OUTER JOIN  - NO INDEX'
SELECT 
 bt.id
 , bt.SomeColumn
FROM 
 TestPerfSchema.BigTable AS bt
 LEFT OUTER JOIN 
 TestPerfSchema.SmallerTable AS st
	 ON bt.SomeColumn = st.LookupColumn
WHERE 
 LookupColumn IS NULL
SET STATISTICS TIME OFF
 

SET STATISTICS TIME ON
PRINT 'NOT EXISTS  - NO INDEX'
SELECT 
 bt.Id
 , bt.SomeColumn
FROM 
 TestPerfSchema.BigTable AS bt
WHERE NOT EXISTS (
  SELECT 
    1
  FROM 
    TestPerfSchema.SmallerTable AS st
  WHERE 
    st.LookupColumn = bt.SomeColumn
)
SET STATISTICS TIME OFF


-- ##
-- ## CREATE INDEX
-- ##
CREATE INDEX idx_BigTable_SomeColumn ON TestPerfSchema.BigTable (SomeColumn)
CREATE INDEX idx_SmallerTable_LookupColumn ON TestPerfSchema.SmallerTable (LookupColumn)


SET STATISTICS TIME ON
PRINT 'LEFT OUTER JOIN  - INDEX'
SELECT 
 bt.Id
 , bt.SomeColumn
FROM 
 TestPerfSchema.BigTable AS bt
 LEFT OUTER JOIN 
 TestPerfSchema.SmallerTable AS st
	 ON bt.SomeColumn = st.LookupColumn
WHERE 
 LookupColumn IS NULL
SET STATISTICS TIME OFF


SET STATISTICS TIME ON
PRINT 'NOT EXISTS  - INDEX'
SELECT 
 bt.id
 , bt.SomeColumn
FROM 
 TestPerfSchema.BigTable AS bt
WHERE NOT EXISTS (
  SELECT 
    1
  FROM 
    TestPerfSchema.SmallerTable AS st
  WHERE 
    st.LookupColumn = bt.SomeColumn
)
SET STATISTICS TIME OFF

```