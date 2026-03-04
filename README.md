# eMES CROSS API Bug Fix 記錄

## 日期：2026-03-04

## 問題描述
eMES 自動擷取資訊計時器維護功能出現以下錯誤：
- `item.get Import error!`
- `supplier.get Import error!`
- `item.supplier.get Import error!`

---

## 問題根因分析

### 1. SQL 單引號注入錯誤
**錯誤訊息**：`Invalid SQL statement or JDBC escape, terminating ''' not found`

**原因**：ERP 回傳的資料包含單引號（如 `'8850646100525`），導致 SQL 語法錯誤。

### 2. CROSS API 超時錯誤
**錯誤訊息**：`org.apache.axis2.AxisFault: Read timed out`

**原因**：SendFormCross.java 沒有設定超時時間，預設時間不足。

### 3. 資料截斷錯誤 (Data truncation)
**錯誤訊息**：`java.sql.DataTruncation: Data truncation`

**原因**：ERP 回傳的資料長度超過 eMES 資料庫欄位長度限制。

### 4. SQL 欄位名稱錯誤
**錯誤訊息**：`Invalid column name 'ID'`

**原因**：`upItemSupplierCROSS` 方法的 UPDATE 語句使用 `WHERE ID=:ID`，但 `material_catalog` 表的主鍵是 `material_id`。

---

## 修正內容

### 修正 1：RegularESB.java - 單引號跳脫處理

**檔案**：`SFT_dataImport/src/com/dci/sft/sql/RegularESB.java`

在 12 個位置加入 `.replace("'", "''")`：

| 行號 | 欄位變數 | 方法 |
|------|---------|------|
| 1184 | itemNo | upItemCROSS |
| 2287 | docType | upDocumentTypeCROSS |
| 2288 | docTypeNo | upDocumentTypeCROSS |
| 2350 | machineNo | upEquitpmentCROSS |
| 2351 | supplierNoForMachine | upEquitpmentCROSS |
| 2455-2456 | op_no, workstation_no | upOPWorkStationCROSS |
| 2609 | warehouseNo | upWarehouseCROSS |
| 2693 | workstationNo | upWorkStationCROSS |
| 2795 | supplierNo | upWorkStationCROSS_supplier |
| 2882 | customerNo | upCustomerCROSS |
| 3740 | itemNoForSupplier | upItemSupplierCROSS |

**修改範例**：
```java
// 修改前
String itemNo = con.getString("item_no").trim();

// 修改後
String itemNo = con.getString("item_no").trim().replace("'", "''");
```

---

### 修正 2：SendFormCross.java - 超時設定

**檔案**：`SFT_ERPIntegrate/src/com/dci/sft/erp/webservice/SendFormCross.java`

**修改位置**：第 245-250 行

```java
public void SendToCross() throws RemoteException{
    IntegrationEntryStub stub = new IntegrationEntryStub();

    // 設定 CROSS API 超時時間 (300秒)
    long timeout = 300 * 1000;
    stub._getServiceClient().getOptions().setTimeOutInMilliSeconds(timeout);
    // ... 後續程式碼
}
```

---

### 修正 3：RegularESB.java - 詳細 LOG 記錄

**檔案**：`SFT_dataImport/src/com/dci/sft/sql/RegularESB.java`

在 13 個 CROSS 方法加入詳細錯誤記錄：

| 方法名稱 | API 名稱 | 追蹤欄位 |
|---------|---------|---------|
| upItemCROSS | item.get | item_no |
| upDocumentTypeCROSS | document.type.get | doc_type/doc_type_no |
| upEquitpmentCROSS | machine.get | machine_no |
| upOperationCROSS | op.get | op_no/workstation_no |
| upCmsnlCROSS | storage.spaces.get | warehouse_no/storage_spaces_no |
| upWarehouseCROSS | warehouse.get | warehouse_no |
| upWorkStationCROSS_workstation | workstation.get | workstation_no |
| upWorkStationCROSS_supplier | supplier.get | supplier_no |
| upCustomerCROSS | customer.get | customer_no |
| upUserCustomerCROSS | user.get | employee_no |
| upMoldInfoCROSS | mold.fixture.get | mold_no |
| upBomCROSS | process.get | process_item/process_code |
| upItemSupplierCROSS | item.supplier.get | item_no |

**LOG 格式範例**：
```
[RegularESB][upItemCROSS][ERROR] item_no='8850646100525, 錯誤訊息: Invalid SQL statement...
[RegularESB][upItemCROSS][DETAIL] Exception類型: java.sql.SQLException
```

---

### 修正 4：資料庫欄位長度擴大

**檔案**：`SFT_web/sql/2-2. COMPANY_WF.sql`

```sql
PRINT'解決 CROSS API Data truncation 錯誤 - 擴大欄位長度'
-- WORKSTATION.NAME: 40 → 100
SET @SQL='IF EXISTS (SELECT 1 FROM ['+@SFTDBNAME+'].INFORMATION_SCHEMA.COLUMNS WHERE TABLE_NAME=''WORKSTATION'' AND COLUMN_NAME=''NAME'' AND CHARACTER_MAXIMUM_LENGTH < 100) '
SET @SQL=@SQL+'ALTER TABLE ['+@SFTDBNAME+']..[WORKSTATION] ALTER COLUMN NAME NVARCHAR(100) '
EXEC (@SQL)

-- material_catalog.material_name: 50 → 100
SET @SQL='IF EXISTS (SELECT 1 FROM ['+@SFTDBNAME+'].INFORMATION_SCHEMA.COLUMNS WHERE TABLE_NAME=''material_catalog'' AND COLUMN_NAME=''material_name'' AND CHARACTER_MAXIMUM_LENGTH < 100) '
SET @SQL=@SQL+'ALTER TABLE ['+@SFTDBNAME+']..[material_catalog] ALTER COLUMN material_name NVARCHAR(100) '
EXEC (@SQL)

-- material_catalog.norm: 50 → 255
SET @SQL='IF EXISTS (SELECT 1 FROM ['+@SFTDBNAME+'].INFORMATION_SCHEMA.COLUMNS WHERE TABLE_NAME=''material_catalog'' AND COLUMN_NAME=''norm'' AND CHARACTER_MAXIMUM_LENGTH < 255) '
SET @SQL=@SQL+'ALTER TABLE ['+@SFTDBNAME+']..[material_catalog] ALTER COLUMN norm NVARCHAR(255) '
EXEC (@SQL)
```

---

### 修正 5：RegularESB.java - SQL 欄位名稱修正

**檔案**：`SFT_dataImport/src/com/dci/sft/sql/RegularESB.java`

**修改位置**：第 3885 行 (upItemSupplierCROSS 方法)

```java
// 修改前
+ " WHERE ID=:ID ";

// 修改後
+ " WHERE material_id=:ID ";
```

---

## 需要重新編譯的模組

1. **SFT_dataImport** - RegularESB.java 修改
2. **SFT_ERPIntegrate** - SendFormCross.java 修改

---

## 資料庫執行 SQL

單一資料庫可直接執行：
```sql
ALTER TABLE WORKSTATION ALTER COLUMN NAME NVARCHAR(100);
ALTER TABLE material_catalog ALTER COLUMN material_name NVARCHAR(100);
ALTER TABLE material_catalog ALTER COLUMN norm NVARCHAR(255);
```

或執行 `2-2. COMPANY_WF.sql` 更新所有公司資料庫。

---

### 修正 6：SFT_BOMMF 資料庫欄位長度擴大

**問題**：`upBomCROSS` 方法 INSERT 到 `SFT_BOMMF` 表時發生 Data truncation 錯誤。

**檔案**：`SFT_web/sql/2-2. COMPANY_WF.sql`

**注意**：MF001, MF002, MF003, MF006 是 PRIMARY KEY，不可修改。

```sql
-- SFT_BOMMF 非 PK 欄位擴大
ALTER TABLE SFT_BOMMF ALTER COLUMN MF004 NVARCHAR(40);
ALTER TABLE SFT_BOMMF ALTER COLUMN MF007 NVARCHAR(100);
ALTER TABLE SFT_BOMMF ALTER COLUMN MF015 NVARCHAR(20);
ALTER TABLE SFT_BOMMF ALTER COLUMN MF017 NVARCHAR(20);
ALTER TABLE SFT_BOMMF ALTER COLUMN MF034 NVARCHAR(40);
ALTER TABLE SFT_BOMMF ALTER COLUMN MF040 NVARCHAR(40);
```

| 欄位 | 原長度 | 新長度 | 備註 |
|------|--------|--------|------|
| MF004 | 10 | 40 | ✅ 非 PK |
| MF007 | 40 | 100 | ✅ 非 PK |
| MF015 | 4 | 20 | ✅ 非 PK |
| MF017 | 6 | 20 | ✅ 非 PK |
| MF034 | 10 | 40 | ✅ 非 PK |
| MF040 | 15 | 40 | ✅ 非 PK |

---

### 修正 7：RegularESB.java - PRIMARY KEY 重複錯誤修正 (UPDATE + INSERT + TRY-CATCH)

**問題**：`upItemCROSS` 方法在處理 `material_catalog` 時，同一批次 ERP 傳入重複品號會報 PRIMARY KEY 重複錯誤。

**錯誤訊息**：
```
Violation of PRIMARY KEY constraint 'PK_material_catalog'.
Cannot insert duplicate key in object 'dbo.material_catalog'.
The duplicate key value is (ZZ09-J01 ZN Yellow Cr+6 (????????)).
```

**根因分析**：
- ERP 同一批次傳入重複的品號（如 `ZZ09-J01 ZN Yellow Cr+6 (????????)` 出現兩次）
- 原本使用 MERGE 語句，但在同批次重複資料時仍會報錯
- 第一筆 MERGE 成功 INSERT，第二筆 MERGE 又嘗試 INSERT 導致 PK 衝突

**檔案**：`SFT_dataImport/src/com/dci/sft/sql/RegularESB.java`

**解決方案**：改用 UPDATE + INSERT + TRY-CATCH 模式

```java
// 修改前：MERGE（同批次重複資料仍會報錯）
sql = " MERGE INTO material_catalog AS target "
    + " USING (SELECT :ID AS material_id) AS source "
    + " ON target.material_id = source.material_id "
    + " WHEN MATCHED THEN UPDATE SET ... "
    + " WHEN NOT MATCHED THEN INSERT ... ";

// 修改後：UPDATE + INSERT + TRY-CATCH
// 1. 先嘗試 UPDATE
String updateSql = " UPDATE material_catalog SET material_name = :NAME, "
        + " norm = :DESCRIPTION, unit_no = :UNIT, "
        + " ENABLED = :ENABLED, MODI_DATE = N'...' "
        + " WHERE material_id = :ID ";
int updateCount = conm.sqlUpdate(updateSql, con);

// 2. 如果沒更新到任何記錄，嘗試 INSERT
if (updateCount == 0) {
    try {
        String insertSql = " INSERT INTO material_catalog (...) VALUES (...) ";
        conm.sqlUpdate(insertSql, con);
    } catch (Exception insertEx) {
        // 3. 如果 INSERT 報 PRIMARY KEY 錯誤（同批次重複），改用 UPDATE
        if (insertEx.getMessage().contains("PRIMARY KEY")) {
            logger.warn("[RegularESB][upItemCROSS][WARN] 品號已存在，改用UPDATE: " + currentItemNo);
            conm.sqlUpdate(updateSql, con);
        } else {
            throw insertEx;
        }
    }
}
sql = ""; // 已處理完成，清空 sql 避免重複執行
```

**處理邏輯**：
```
同批次有重複資料時：
第一筆: UPDATE (0筆) → INSERT (成功)
第二筆: UPDATE (1筆) → 完成 (不會報錯)

或者：
第一筆: UPDATE (0筆) → INSERT (成功)
第二筆: UPDATE (0筆) → INSERT (PK錯誤) → catch → UPDATE (成功)
```

**效果**：
- 資料存在 → UPDATE
- 資料不存在 → INSERT
- 同批次重複 → 第一筆 INSERT，後續 UPDATE
- **不會再報 PRIMARY KEY 重複 ERROR**

---

## 完整修改清單

| # | 修正項目 | 檔案 |
|---|---------|------|
| 1 | 單引號跳脫處理 | RegularESB.java |
| 2 | CROSS API 超時設定 | SendFormCross.java |
| 3 | 詳細 LOG 記錄 | RegularESB.java |
| 4 | WORKSTATION/material_catalog 欄位擴大 | 2-2. COMPANY_WF.sql |
| 5 | SQL 欄位名稱修正 (ID→material_id) | RegularESB.java |
| 6 | SFT_BOMMF 欄位擴大 | 2-2. COMPANY_WF.sql |
| 7 | PRIMARY KEY 重複錯誤修正 (UPDATE+INSERT+TRY-CATCH) | RegularESB.java |

---

## 版本同步

所有修改已同步至：
- **標準版**：`SFT_*` 模組
- **亨利五金版**：`SFT_*_0000675600_1611` 模組
