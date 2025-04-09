**SQL_Triggers_1**

The following trigger creates a log of new rows added to a table.

A new schema with 2 tables is created. 

The first to insert very simple customer information into and the second for the trigger to log those inserts with a timestamp. 


```sql
CREATE SCHEMA trigger_practice;

CREATE TABLE customers (id INT, first_name VARCHAR(255), last_name VARCHAR (255));

CREATE TABLE audit ( id INT, entry_date VARCHAR(255));

```

Next a function is created called auditlog, this inserts an id and timestamp into the audit table.

Using this function a trigger is created which is activated when data is inserted into the customers table.

```sql

CREATE OR REPLACE FUNCTION auditlog() RETURNS TRIGGER AS $$
     BEGIN
	    INSERT INTO audit( id, entry_date) VALUES (new.id, date_trunc('second', current_timestamp));
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
VALUES (1,'Thomas', 'Newman');

INSERT INTO customers (id, first_name, last_name)
VALUES (2,'Randy', 'Newman');


SELECT * FROM customers;

```


| id  |	first_name  | last_name |
|-----|-------------|-----------|
| 1	  | Thomas  | Newman  |
| 2	  | Randy  |	Newman  |

```sql

SELECT * FROM audit;

```

|   id |         entry_date             |
|--------------|--------------------------------|
| 1	           | 2025-03-30 12:36:07+01  |
| 2	           | 2025-03-30 12:37:14+01  |

By adding a new column to the audit table and modifying the auditlog function the trigger can now log which user made the input into the customer table.

```sql

ALTER TABLE audit
ADD COLUMN inserted_by VARCHAR(255);



CREATE OR REPLACE FUNCTION auditlog() 
RETURNS TRIGGER AS $$
BEGIN
   
    INSERT INTO audit( id, entry_date, inserted_by) 
    VALUES (NEW.id, date_trunc('second', current_timestamp), CURRENT_USER);
    
   
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;



INSERT INTO customers (id, first_name, last_name)
VALUES (3, 'Paul', 'Newman');

```


```sql

SELECT * FROM customers;

```

| id   |	first_name  | last_name  |
|------|--------------|------------|
| 1	   | Thomas |	Newman   |
| 2	   | Randy  | Newman   |
| 3	   | Paul   |	   Newman |

```sql

SELECT * FROM audit;

```

|        id   |   	    entry_date 	           |   inserted_by  |
|---------------|--------------------------------|----------------|
|   1	            | 2025-03-30 12:36:07+01  |	       |
|   2	          | 2025-03-30 12:38:14+01 |      |
|   3	         |  2025-03-30 12:40:38+01 |   	postgres  |



The audit table can be modified further to create logs for added, deleted and updated rows.

To avoid any overlap in triggers,  the previous trigger and function are removed.

```sql
DROP TRIGGER trigger_example1 on customers; 

DROP FUNCTION auditlog();

```

An extre column is also added to the audit table, which will log the action details of each entry e.g.  ‘deleted’.

```sql
ALTER TABLE table_name
ADD column_name datatype;
```


With the following we can create a function and trigger for inserting rows;

```sql

CREATE OR REPLACE FUNCTION audit_insert_function() 
RETURNS TRIGGER AS $$
BEGIN

    INSERT INTO audit (id, entry_date, inserted_by, action) 
    VALUES (NEW.id, date_trunc('second', current_timestamp), CURRENT_USER, 'Added');


    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE OR REPLACE TRIGGER audit_ins_trig
    AFTER INSERT ON customers
    FOR EACH ROW
    EXECUTE FUNCTION audit_insert_function();

```

With the following we can create a function and trigger for deleting rows;

```sql
CREATE OR REPLACE FUNCTION audit_delete_function() 
RETURNS TRIGGER AS $$
BEGIN

    INSERT INTO audit (id, entry_date, inserted_by, action) 
    VALUES (old.id, date_trunc('second', current_timestamp), CURRENT_USER, 'Deleted');


    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE OR REPLACE TRIGGER audit_del_trig
    AFTER DELETE ON customers
    FOR EACH ROW
    EXECUTE FUNCTION audit_delete_function();
```

With the following we can create a function and trigger for updating rows;

```sql

CREATE OR REPLACE FUNCTION audit_change_function() 
RETURNS TRIGGER AS $$
BEGIN
    
    IF OLD.first_name <> NEW.first_name THEN
        INSERT INTO audit (id, entry_date, inserted_by, action) 
        VALUES (NEW.id, date_trunc('second', current_timestamp), CURRENT_USER, 
                'Changed first_name from ' || OLD.first_name || ' to ' || NEW.first_name);
    END IF;

    
    IF OLD.last_name <> NEW.last_name THEN
        INSERT INTO audit (id, entry_date, inserted_by, action) 
        VALUES (NEW.id, date_trunc('second', current_timestamp), CURRENT_USER, 
                'Changed last_name from ' || OLD.last_name || ' to ' || NEW.last_name);
    END IF;

  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE OR REPLACE TRIGGER audit_chg_trig
    AFTER UPDATE ON customers
    FOR EACH ROW
    EXECUTE FUNCTION audit_change_function();

```

The new triggers can be tested with the following;

```sql

INSERT INTO customers
VALUES (4,'Steve','Martin');

DELETE FROM customers 
WHERE id = 1;

UPDATE customers 
SET first_name = 'Stacey'
WHERE id = 4 ;

UPDATE customers 
SET last_name = 'Wilson'
WHERE id = 4;

```

```sql

SELECT * FROM customers;

```

| id   |	first_name  | last_name  |
|------|--------------|------------|
| 2	   | Randy  | Newman   |
| 3	   | Paul   |	   Newman |
| 4       | Stacey | Wilson  |

```sql
SELECT * FROM audit;
```

|        id   |   	    entry_date 	           |   inserted_by  |  action    |
|---------------|--------------------------------|----------------|--------------|
|   1	            | 2025-03-30 12:36:07+01  |	       |  |
|   2	          | 2025-03-30 12:38:14+01 |      |   |
|   3	         |  2025-03-30 12:40:38+01 |   	postgres |  |
| 4	|  2025-04-07 17:06:40+01 	| postgres |	Added  |
| 1	| 2025-04-07 17:07:13+01 	| postgres |	Deleted |
| 4	| 2025-04-07 17:08:46+01 	| postgres |	Changed first_name from Steve to Stacey |
| 4	| 2025-04-07 17:09:33+01 	| postgres |	Changed last_name from Martin to Wilson |



All of the above triggers are **after row-level triggers** meaning that the triggered function occurs after the trigger event occurs. 

It is also possible to create triggers that iniate before the trigger event occurs, these are known as  **before row-level triggers**.

The following is an example of a before row-level trigger.

First a new example 'Employees' table is created, with basic employee details such as name and salary.

```sql

CREATE TABLE Employees (emp_id INT, 
				first_name VARCHAR(20),
				emp_last_name VARCHAR (20),
				department_no INT,
				salary INT, 
				bonus INT);

```

Then a function is created to calculate the amount of bonus an employee is entitled to based on their department and salary.
In this example all employees from department 2 are entitled to 25% of their salary as a bonus. This function is followed by a trigger which runs before any inserts into the Employees table.

```sql

CREATE OR REPLACE FUNCTION emp_bonus_function()
RETURNS TRIGGER AS $$
BEGIN
    
    IF NEW.department_no = 2 THEN
       NEW.bonus := NEW.salary * 0.25;
    END IF;
    RETURN NEW;
    END;
$$ LANGUAGE plpgsql;

CREATE OR REPLACE TRIGGER emp_bonus_trig
    BEFORE INSERT ON Employees
    FOR EACH ROW
    EXECUTE FUNCTION emp_bonus_function();

```
Now when data is inserted into the Employees table, the bonuses are calculated for the rows with department number 2.

```sql

INSERT INTO Employees 
VALUES (1,'Mary', 'Adams', 2 , 35000, null),
 (2, 'Barbara', 'Matthews', 1, 40000, null ),
(3, 'Tony', 'Thomas', 2, 30000, null);

SELECT * FROM Employees;
```
| emp_id  |	first_name  |	 emp_last_name  |	department_no  |   salary  |	bonus  |
|---------|-----------------|-------------------|-----------------|-----------|-------|
|1	| Mary  | Adams  |	2	| 35000 |	8750 |
|2	 | Barbara |	Matthews |	1 |  40000 |	|
|3	| Tony | Thomas |	2 | 30000 |	7500 |






