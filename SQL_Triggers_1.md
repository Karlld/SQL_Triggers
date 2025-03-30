**SQL_Triggers_1**

A new schema with 2 tables is created. 

The first to insert very simple customer information into and the second for the trigger to log those inserts with a timestamp. 


```sql
CREATE SCHEMA trigger_practice;

CREATE TABLE customers (id INT, first_name VARCHAR(255), last_name VARCHAR (255));

CREATE TABLE audit (cutomer_id INT, entry_date VARCHAR(255));

```

Next a function is created called auditlog, this inserts an id and timestamp into the audit table.

Using this function a trigger is created which is activated when data is inserted into the customers table.

```sql

CREATE OR REPLACE FUNCTION auditlog() RETURNS TRIGGER AS $$
     BEGIN
	    INSERT INTO audit(customer_id, entry_date) VALUES (new.id, current_timestamp);
		   RETURN NEW;
		   END;
		   $$ LANGUAGE plpgsql;
		   
CREATE TRIGGER 
  TRIGGER_EXAMPLE1
 AFTER INSERT ON customers
 FOR EACH ROW
 EXECUTE FUNCTION auditlog();

```

The trigger can then be tested by inserting data into the customers table.

```sql

INSERT INTO customers (id, first_name, last_name)
VALUES (1,'Thomas', ‘Newman’);

INSERT INTO customers (id, first_name, last_name)
VALUES (2,'Randy', 'Newman');


SELECT * FROM customers;

```


| id  |	first_name  | last_name |
|-----|-------------|-----------|
| 1	  | Thomas  | Newman  |
| 2	  | Randy  |	Newman  |

```sql

SELECT * from audit;

```

|  customer_id |         entry_date             |
|--------------|--------------------------------|
| 1	           | 2025-03-30 12:36:07.766638+01  |
| 2	           | 2025-03-30 12:37:14.842566+01  |

```

```sql

ALTER TABLE audit

ADD COLUMN inserted_by VARCHAR(255);

CREATE OR REPLACE FUNCTION auditlog() 
RETURNS TRIGGER AS $$
BEGIN
   
    INSERT INTO audit(customer_id, entry_date, inserted_by) 
    VALUES (NEW.id, current_timestamp, CURRENT_USER);
    
   
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

INSERT INTO customers (id, first_name, last_name)
VALUES (3, 'Paul', 'Newman')

```

```sql

SELECT * FROM customers

```

| id   |	first_name  | last_name  |
|------|--------------|------------|
| 1	   | Thomas |	Newman   |
| 2	   | Randy  | Newman   |
| 3	   | Paul   |	   Newman |

```sql

SELECT * FROM audit

```

| customer_id   |   	    entry_date 	           |   inserted_by  |
|---------------|--------------------------------|----------------|
|   1	            | 2025-03-30 12:36:07.766638+01  |	       |
|   2	          | 2025-03-30 12:38:14.842566+01 |      |
|   3	         |  2025-03-30 15:14:38.022225+01 |   	postgres  |

