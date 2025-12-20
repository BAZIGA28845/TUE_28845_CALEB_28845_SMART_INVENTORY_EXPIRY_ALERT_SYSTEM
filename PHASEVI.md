
## **PHASE VI – Logging, Procedures, Functions & Package**

---

## **1️⃣ Error Log Table**
<img width="946" height="275" alt="z create error log table" src="https://github.com/user-attachments/assets/2170098b-973b-41d2-bcc5-6e88c7c9e600" />

**Purpose:** Stores errors that occur inside procedures for debugging and auditing.

```sql
CREATE TABLE error_log (
    error_id       NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    error_date     TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    procedure_name VARCHAR2(100),
    error_message  VARCHAR2(4000),
    error_code     NUMBER,
    username       VARCHAR2(50)
) TABLESPACE SIEAS_DATA;
```

---

## **2️⃣ Procedures (Business Operations)**
<img width="960" height="506" alt="z adjustbatchquantity in and out" src="https://github.com/user-attachments/assets/40617d48-3613-4e45-b062-863141e8858e" />

### **Adjust Batch Quantity (IN OUT)**

**Purpose:** Updates batch quantity and returns the new quantity. Logs error if batch doesn’t exist.

```sql
CREATE OR REPLACE PROCEDURE adjust_batch_quantity (
    p_batch_id IN VARCHAR2,
    p_change   IN OUT NUMBER
) IS
    v_current_qty NUMBER;
BEGIN
    SELECT quantity INTO v_current_qty
    FROM batch WHERE batch_id = p_batch_id;

    UPDATE batch
    SET quantity = v_current_qty + p_change
    WHERE batch_id = p_batch_id;

    p_change := v_current_qty + p_change;
    COMMIT;

EXCEPTION
    WHEN NO_DATA_FOUND THEN
        INSERT INTO error_log
        VALUES (DEFAULT, CURRENT_TIMESTAMP,
                'adjust_batch_quantity',
                'Batch not found', -20002, USER);
END;
/
```

---

### **Add Product**
<img width="960" height="503" alt="z add_product" src="https://github.com/user-attachments/assets/5f71b299-b5b6-44c4-8627-756b2ff650e9" />

**Purpose:** Inserts a new product and logs any unexpected error.

```sql
CREATE OR REPLACE PROCEDURE add_product (
    p_name       IN VARCHAR2,
    p_category   IN VARCHAR2,
    p_unit_price IN NUMBER
) IS
    v_product_id VARCHAR2(36);
BEGIN
    v_product_id := uuid();

    INSERT INTO product
    VALUES (v_product_id, p_name, p_category, p_unit_price);

    COMMIT;

EXCEPTION
    WHEN OTHERS THEN
        INSERT INTO error_log
        VALUES (DEFAULT, CURRENT_TIMESTAMP,
                'add_product',
                SQLERRM, SQLCODE,
                SYS_CONTEXT('USERENV','SESSION_USER'));
END;
/
```

---

### **Delete Processed Alerts**
<img width="960" height="505" alt="z Delete Processed Alerts (IN parameter)" src="https://github.com/user-attachments/assets/98d2d8ff-76ad-40b2-b849-7ecfb4a702a3" />

**Purpose:** Removes processed expiry alerts and returns how many were deleted.

```sql
CREATE OR REPLACE PROCEDURE delete_processed_alerts (
    p_deleted OUT NUMBER
) IS
BEGIN
    DELETE FROM expiry_alert WHERE processed = 'Y';
    p_deleted := SQL%ROWCOUNT;
    COMMIT;
END;
/
```

---

## **3️⃣ Functions (Validation & Lookup)**

### **Email Validation**
<img width="960" height="503" alt="z Email Validation Function" src="https://github.com/user-attachments/assets/748e9c62-70fc-4294-bb3f-500473aa06f3" />

**Purpose:** Checks if an email format is valid.

```sql
CREATE OR REPLACE FUNCTION is_valid_email (
    p_email IN VARCHAR2
) RETURN BOOLEAN IS
BEGIN
    RETURN REGEXP_LIKE(
        p_email,
        '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$'
    );
END;
/
```

---

### **Get Product Name by Batch**
<img width="960" height="504" alt="z Lookup Product Name by Batch" src="https://github.com/user-attachments/assets/08c6f32e-c5ea-4685-8a1d-36256581cb05" />

**Purpose:** Returns product name linked to a batch.

```sql
CREATE OR REPLACE FUNCTION get_product_name (
    p_batch_id IN VARCHAR2
) RETURN VARCHAR2 IS
    v_name VARCHAR2(120);
BEGIN
    SELECT p.name INTO v_name
    FROM product p JOIN batch b
    ON p.product_id = b.product_id
    WHERE b.batch_id = p_batch_id;

    RETURN v_name;
EXCEPTION
    WHEN NO_DATA_FOUND THEN RETURN NULL;
END;
/
```

---

### **Total Stock per Product**
<img width="960" height="504" alt="z Calculate Total Stock per Product" src="https://github.com/user-attachments/assets/211fb620-c05d-492e-8b8f-92bdf3148a8b" />

**Purpose:** Calculates total quantity of all batches for a product.

```sql
CREATE OR REPLACE FUNCTION total_stock (
    p_product_id IN VARCHAR2
) RETURN NUMBER IS
    v_total NUMBER;
BEGIN
    SELECT SUM(quantity) INTO v_total
    FROM batch WHERE product_id = p_product_id;

    RETURN NVL(v_total, 0);
END;
/
```

---

## **4️⃣ Analytical (Window) Queries**

### **Rank Products by Stock**
<img width="960" height="506" alt="z Rank Products by Stock Quantity" src="https://github.com/user-attachments/assets/e279a9f0-3e7e-444d-ad9d-ef09e1fde3ce" />

**Purpose:** Ranks products based on total stock quantity.

```sql
SELECT p.name,
       SUM(b.quantity) total_qty,
       RANK() OVER (ORDER BY SUM(b.quantity) DESC) rank_no
FROM product p JOIN batch b
ON p.product_id = b.product_id
GROUP BY p.name;
```

---

### **Batch Expiry Order**
<img width="956" height="503" alt="z Batch Expiry Order per Product" src="https://github.com/user-attachments/assets/91c1e8cf-28f6-4c5c-8faa-2d29a44b3ced" />

**Purpose:** Orders batches by expiry date per product.

```sql
SELECT product_id, batch_id, expiry_date,
       ROW_NUMBER() OVER
       (PARTITION BY product_id ORDER BY expiry_date) seq
FROM batch;
```

---

### **Expiry Gap Analysis**
<img width="957" height="502" alt="z Expiry Trend Comparison (LAG" src="https://github.com/user-attachments/assets/a8e64eba-2327-427d-a021-f3a86974ee6f" />

**Purpose:** Shows number of days between batch expiry dates.

```sql
SELECT batch_id, expiry_date,
       expiry_date -
       LAG(expiry_date) OVER (ORDER BY expiry_date) gap_days
FROM batch;
```

---

## **5️⃣ Inventory Package**

**Purpose:** Groups inventory operations into one reusable unit.
<img width="958" height="504" alt="z Package Specification" src="https://github.com/user-attachments/assets/94376106-3e48-47a3-b6f5-fe9bbf73cf64" />

### **Package Specification**

```sql
CREATE OR REPLACE PACKAGE inventory_pkg IS
    PROCEDURE generate_alerts;
    PROCEDURE expire_batches;
    FUNCTION product_stock (p_product_id VARCHAR2) RETURN NUMBER;
END inventory_pkg;
/
```

---
<img width="958" height="505" alt="z Package Body" src="https://github.com/user-attachments/assets/3d5e3ffe-a932-449a-ad98-e817bf75f492" />

### **Package Body**

```sql
CREATE OR REPLACE PACKAGE BODY inventory_pkg IS

    PROCEDURE generate_alerts IS
    BEGIN
        generate_expiry_alerts;
    END;

    PROCEDURE expire_batches IS
    BEGIN
        move_expired_stock;
    END;

    FUNCTION product_stock (p_product_id VARCHAR2)
    RETURN NUMBER IS
    BEGIN
        RETURN total_stock(p_product_id);
    END;

END inventory_pkg;
/
```
**cursors**
cursor allows PL/SQL to fetch and process multiple rows individually from a query result.
<img width="959" height="503" alt="0000CURSORS" src="https://github.com/user-attachments/assets/8616996a-f7d6-4378-bd1c-b94422c05b6e" />

---

## **6️⃣ Testing**

**Purpose:** Verifies that the package works correctly.
<img width="937" height="204" alt="0 testing" src="https://github.com/user-attachments/assets/f61aae9c-f8b4-4bb9-a34e-ebb65106a9de" />
<img width="937" height="204" alt="0 testing" src="https://github.com/user-attachments/assets/f07f91d6-c9b4-4fb1-a89b-125eaa0fc698" />

```sql
EXEC inventory_pkg.generate_alerts;
EXEC inventory_pkg.expire_batches;

SELECT inventory_pkg.product_stock(product_id)
FROM product;
```

---

### **Test 1: Check Error Log Table**

**Purpose:** Verify that errors are being recorded.
<img width="960" height="190" alt="0 2 testing Check Error Log Table" src="https://github.com/user-attachments/assets/0658227e-c0e6-4cd5-aad5-18ed818d1d03" />

```sql
SELECT * FROM error_log
ORDER BY error_date DESC;
```

---

### **Test 2: Test `add_product` Procedure**

**Purpose:** Confirm a new product is inserted.
<img width="938" height="282" alt="0 3 Test add_product Procedure" src="https://github.com/user-attachments/assets/5e66e65f-9403-43f7-b0d6-fbe8b1bd6886" />

```sql
EXEC add_product('Paracetamol', 'Medicine', 1200);

SELECT * FROM product
WHERE name = 'Paracetamol';
```

---

### **Test 3: Test `adjust_batch_quantity` Procedure**

**Purpose:** Increase or decrease batch quantity.
<img width="960" height="505" alt="0 4Test adjust_batch_quantity Procedure" src="https://github.com/user-attachments/assets/69b27879-aee4-441d-bb41-5cc8fc366527" />

```sql
DECLARE
    v_new_qty NUMBER := 20;
BEGIN
    adjust_batch_quantity('BATCH001', v_new_qty);
    DBMS_OUTPUT.PUT_LINE('New Quantity: ' || v_new_qty);
END;
/
```

---

### **Test 4: Test Invalid Batch (Error Logging)**
<img width="954" height="277" alt="0 5 Test Invalid Batch (Error Logging)" src="https://github.com/user-attachments/assets/e7f2915c-313a-4964-b7e4-cf1a53951305" />

**Purpose:** Confirm error is logged when batch does not exist.

```sql
DECLARE
    v_qty NUMBER := 10;
BEGIN
    adjust_batch_quantity('INVALID_BATCH', v_qty);
END;
/

SELECT * FROM error_log
WHERE procedure_name = 'adjust_batch_quantity';
```

---

### **Test 5: Test `delete_processed_alerts` Procedure**
<img width="958" height="309" alt="0 6Test delete_processed_alerts Procedure" src="https://github.com/user-attachments/assets/c3633a78-ae31-4163-9010-d13e23ccbcba" />

**Purpose:** Check how many processed alerts are removed.

```sql
DECLARE
    v_deleted NUMBER;
BEGIN
    delete_processed_alerts(v_deleted);
    DBMS_OUTPUT.PUT_LINE('Deleted Alerts: ' || v_deleted);
END;
/
```

---

### **Test 6: Test Email Validation Function**

**Purpose:** Validate correct and incorrect emails.
<img width="949" height="214" alt="0 7 Test Email Validation Function" src="https://github.com/user-attachments/assets/9c7d994f-5b16-4622-a2ed-d458ac3f3030" />

```sql
SELECT is_valid_email('user@gmail.com')   AS valid_email,
       is_valid_email('user@@gmail.com') AS invalid_email
FROM dual;
```

---

### **Test 7: Test `get_product_name` Function**
<img width="947" height="194" alt="0 8Test get_product_name Function" src="https://github.com/user-attachments/assets/59ba1958-83af-49cd-9bf2-46d281379093" />

**Purpose:** Retrieve product name using batch ID.

```sql
SELECT get_product_name('BATCH001') AS product_name
FROM dual;
```

---

### **Test 8: Test `total_stock` Function**
<img width="946" height="339" alt="0 9Test total_stock Function" src="https://github.com/user-attachments/assets/beb9f652-d689-4052-9e92-69e6ef5040d3" />

**Purpose:** Calculate total stock for one product.

```sql
SELECT total_stock(product_id) AS total_quantity
FROM product;
```

---

### **Test 9: Test Package Function `product_stock`**
<img width="941" height="287" alt="0 10Test Package Function product_stock" src="https://github.com/user-attachments/assets/3e8f3925-3c1e-4746-ac0b-9138d0bab649" />

**Purpose:** Confirm package function works.

```sql
SELECT inventory_pkg.product_stock(product_id) AS stock
FROM product;
```

---

### **Test 10: Verify Expired Batches Moved**
<img width="957" height="293" alt="0 11Verify Expired Batches Moved" src="https://github.com/user-attachments/assets/2525c7ae-7036-46b2-8249-fa825b6bcb7f" />

**Purpose:** Ensure expired batches are processed correctly.

```sql
SELECT * FROM expired_stock;
```

---

### **Test 11: Verify Generated Expiry Alerts**
<img width="945" height="303" alt="0 12Verify Generated Expiry Alerts" src="https://github.com/user-attachments/assets/76c3f72f-1b99-4291-8c48-28ae983dfb4f" />

**Purpose:** Check if alerts were created.

```sql
SELECT * FROM expiry_alert
ORDER BY alert_date DESC;
```

---

### **Test 12: Analytics Verification**

**Purpose:** Confirm window functions return results.
<img width="944" height="353" alt="0 13 product ranking" src="https://github.com/user-attachments/assets/d5466527-0b1e-4b95-a604-c5b6be095889" />
<img width="949" height="332" alt="0 13b Batch expiry order" src="https://github.com/user-attachments/assets/548837be-8a08-47df-9359-f3e9e65867ca" />

```sql
-- Product ranking
SELECT p.name,
       SUM(b.quantity) total_qty,
       RANK() OVER (ORDER BY SUM(b.quantity) DESC) rank_no
FROM product p JOIN batch b
ON p.product_id = b.product_id
GROUP BY p.name;

-- Batch expiry order
SELECT product_id, batch_id, expiry_date,
       ROW_NUMBER() OVER
       (PARTITION BY product_id ORDER BY expiry_date) seq
FROM batch;
```

---

### **Test 13: Data Integrity Checks**
<img width="946" height="147" alt="0 14Data Integrity Checks" src="https://github.com/user-attachments/assets/19af3079-4b86-46f4-ba91-07c27ee81617" />

**Purpose:** Confirm no negative stock values exist.

```sql
SELECT * FROM batch
WHERE quantity < 0;
```

---

### **Test 14: User & Session Test**
<img width="951" height="144" alt="0 15User   Session Test" src="https://github.com/user-attachments/assets/b6f54071-519a-4b89-8457-81f0c1a258a3" />

**Purpose:** Verify correct username logging.

```sql
SELECT DISTINCT username
FROM error_log;
```
---

✅ These tests **cover**:

* Procedures
* Functions
* Error handling
* Packages
* Analytics
* Data integrity
  

---
