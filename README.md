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
