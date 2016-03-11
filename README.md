# SQL Server performance - Inverted INNER JOIN - LEFT OUTER JOIN .. NULL vs NOT EXISTS

# About

This document outlines a performance benchmark on selecting all values from a larger table, joined by a smaller table, where no joined values exists. 

In other words, an inverted INNER JOIN clause.

It compares LEFT OUTER JOIN with NULL including, versus NOT EXISTS, with index and no index on the joined column for both tables.

Scroll down to the bottom for the script details.

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

| Type            | Index | CPU Time (ms) | Elapsed Time (ms) |
|-----------------|-------|---------------|-------------------|
| LEFT OUTER JOIN | No    | 78            | 158               |
| NOT EXISTS      | No    | 63            | 102               |
| LEFT OUTER JOIN | Yes   | 62            | 147               |
| NOT EXISTS      | Yes   | **47**            | **61**                |

### Run 3

| Type            | Index | CPU Time (ms) | Elapsed Time (ms) |
|-----------------|-------|---------------|-------------------|
| LEFT OUTER JOIN | No    | 78            | 135               |
| NOT EXISTS      | No    | 62            | 78                |
| LEFT OUTER JOIN | Yes   | 78            | 128               |
| NOT EXISTS      | Yes   | **31**            | **42**                |


# Script

```sql
-- ##
-- ## LEFT OUTER JOIN [...] NULL 
-- ## vs
-- ## NOT EXISTS
-- ##

-- ##
-- ## BASELINE
-- ##
IF EXISTS (SELECT * FROM INFORMATION_SCHEMA.TABLES WHERE TABLE_NAME = 'LargerTable' AND TABLE_SCHEMA = 'TestPerformance')
BEGIN
 DROP TABLE TestPerformance.LargerTable
END
GO

IF EXISTS (SELECT * FROM INFORMATION_SCHEMA.TABLES WHERE TABLE_NAME = 'SmallerTable' AND TABLE_SCHEMA = 'TestPerformance')
BEGIN
 DROP TABLE TestPerformance.SmallerTable
END
GO

IF EXISTS (SELECT * FROM sys.schemas WHERE name = 'TestPerformance')
BEGIN
 DROP SCHEMA TestPerformance 
END
GO

CREATE SCHEMA TestPerformance
GO

CREATE TABLE TestPerformance.LargerTable (
	Id INT IDENTITY PRIMARY KEY,
	CompareColumn CHAR(4) NOT NULL,
)
 
CREATE TABLE TestPerformance.SmallerTable (
	Id INT IDENTITY PRIMARY KEY,
	LookupColumn CHAR(4) NOT NULL,
)
 
-- ##
-- ## INSERT DATA
-- ##
INSERT INTO 
 TestPerformance.LargerTable (CompareColumn)
	SELECT TOP 
  250000
	CHAR(65+FLOOR(RAND(a.column_id *5645 + b.object_id)*10)) + CHAR(65+FLOOR(RAND(b.column_id *3784 + b.object_id)*12)) +
	CHAR(65+FLOOR(RAND(b.column_id *6841 + a.object_id)*12)) + CHAR(65+FLOOR(RAND(a.column_id *7544 + b.object_id)*8))
	FROM 
  master.sys.columns AS a 
  CROSS JOIN 
  master.sys.columns AS b
 
INSERT INTO TestPerformance.SmallerTable (LookupColumn)
	SELECT DISTINCT CompareColumn
	FROM TestPerformance.LargerTable TABLESAMPLE (25 PERCENT)

-- ##
-- ## NO INDEX
-- ##

SET STATISTICS TIME ON
PRINT 'LEFT OUTER JOIN  - NO INDEX'

SELECT 
 bt.id
 , bt.CompareColumn
FROM 
 TestPerformance.LargerTable AS bt
 LEFT OUTER JOIN 
 TestPerformance.SmallerTable AS st
	 ON bt.CompareColumn = st.LookupColumn
WHERE 
 LookupColumn IS NULL
SET STATISTICS TIME OFF
 

SET STATISTICS TIME ON
PRINT 'NOT EXISTS  - NO INDEX'
SELECT 
 bt.Id
 , bt.CompareColumn
FROM 
 TestPerformance.LargerTable AS bt
WHERE NOT EXISTS (
	SELECT 
  1
	FROM 
  TestPerformance.SmallerTable AS st
	WHERE 
  st.LookupColumn = bt.CompareColumn
)
SET STATISTICS TIME OFF


-- ##
-- ## CREATE INDEX
-- ##
CREATE INDEX idx_LargerTable_CompareColumn ON TestPerformance.LargerTable (CompareColumn)
CREATE INDEX idx_SmallerTable_LookupColumn ON TestPerformance.SmallerTable (LookupColumn)


SET STATISTICS TIME ON
PRINT 'LEFT OUTER JOIN  - INDEX'
SELECT 
 bt.Id
 , bt.CompareColumn
FROM 
 TestPerformance.LargerTable AS bt
 LEFT OUTER JOIN 
 TestPerformance.SmallerTable AS st
	 ON bt.CompareColumn = st.LookupColumn
WHERE 
 LookupColumn IS NULL
SET STATISTICS TIME OFF


SET STATISTICS TIME ON
PRINT 'NOT EXISTS  - INDEX'
SELECT 
 bt.id
 , bt.CompareColumn
FROM 
 TestPerformance.LargerTable AS bt
WHERE NOT EXISTS (
	SELECT 
  1
	FROM 
  TestPerformance.SmallerTable AS st
	WHERE 
  st.LookupColumn = bt.CompareColumn
)
SET STATISTICS TIME OFF


```

# Credits

http://sqlinthewild.co.za/index.php/2010/03/23/left-outer-join-vs-not-exists/