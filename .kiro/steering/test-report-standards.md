# 測試報表標準規範

## 概述

本文件定義了銀行客戶優惠規則引擎專案的測試報表標準，確保測試結果能夠以清晰、結構化的方式呈現給業務分析師 (BA) 和系統分析師 (SA)。

## 報表格式要求

### 支援格式
- **HTML** - 互動式網頁報表，支援篩選和排序
- **PDF** - 正式文件格式，適合存檔和分享
- **Excel** - 資料分析格式，支援進一步的數據處理
- **JSON** - 機器可讀格式，供其他系統整合使用

### 報表結構

#### 1. 執行摘要 (Executive Summary)
```
測試執行摘要
├── 測試執行時間
├── 測試環境資訊
├── 總體統計
│   ├── 總測試案例數
│   ├── 通過案例數
│   ├── 失敗案例數
│   ├── 跳過案例數
│   └── 通過率 (%)
└── 關鍵指標
    ├── 程式碼覆蓋率
    ├── 功能覆蓋率
    └── 需求覆蓋率
```

#### 2. 功能測試結果 (Functional Test Results)
```
功能模組測試結果
├── 優惠查詢服務
│   ├── 測試案例清單
│   ├── 執行結果統計
│   └── 失敗案例詳情
├── 規則管理服務
├── 客戶關聯管理
├── 條件規則引擎
├── 系統整合
└── 安全機制
```

#### 3. 效能測試結果 (Performance Test Results)
```
效能測試結果
├── 回應時間分析
│   ├── 平均回應時間
│   ├── 95% 百分位數
│   └── 最大回應時間
├── 吞吐量分析
│   ├── 每秒交易數 (TPS)
│   ├── 併發用戶數
│   └── 錯誤率
└── 資源使用率
    ├── CPU 使用率
    ├── 記憶體使用率
    └── 資料庫連線數
```

## 測試案例報表格式

### 測試案例詳細資訊
每個測試案例應包含以下資訊：

| 欄位 | 說明 | 範例 |
|------|------|------|
| 測試案例 ID | 唯一識別碼 | TC_PROMO_001 |
| 測試案例名稱 | 簡短描述 | 查詢高級客戶適用優惠 |
| 測試類型 | 測試分類 | 功能測試/整合測試/效能測試 |
| 需求追溯 | 對應需求編號 | REQ_1.1, REQ_1.2 |
| 前置條件 | 執行前的準備 | 客戶 CUST001 已建立，優惠規則已配置 |
| 測試步驟 | 詳細執行步驟 | 1. 呼叫 API 2. 驗證回應 3. 檢查資料庫 |
| 測試資料 | 輸入資料 | customerId: "CUST001", accountType: "PREMIUM" |
| 預期結果 | 期望的輸出 | 回傳 2 個適用優惠，包含 PROMO001 和 PROMO002 |
| 實際結果 | 真實的輸出 | 回傳 2 個適用優惠，符合預期 |
| 執行狀態 | 通過/失敗/跳過 | 通過 |
| 執行時間 | 測試執行時長 | 245ms |
| 錯誤訊息 | 失敗時的錯誤 | N/A |
| 截圖/日誌 | 相關證據 | 連結到詳細日誌 |

### 正向測試案例範例

```markdown
## TC_PROMO_001 - 查詢高級客戶適用優惠

**測試目標**: 驗證高級客戶能夠正確查詢到適用的優惠方案

**需求追溯**: REQ_1.1, REQ_4.1

**前置條件**:
- 客戶 CUST001 已建立，帳戶類型為 PREMIUM
- 優惠規則 PROMO001 (高級客戶專屬) 已啟用
- 優惠規則 PROMO002 (一般優惠) 已啟用

**測試步驟**:
1. 準備測試資料：customerId = "CUST001", accountType = "PREMIUM"
2. 呼叫 POST /api/v1/promotions/query API
3. 驗證 HTTP 狀態碼為 200
4. 驗證回應包含 2 個優惠項目
5. 驗證優惠 ID 包含 PROMO001 和 PROMO002

**預期結果**:
```json
{
  "customerId": "CUST001",
  "applicablePromotions": [
    {
      "promotionId": "PROMO001",
      "promotionName": "高級客戶專屬優惠",
      "promotionType": "PREMIUM_EXCLUSIVE"
    },
    {
      "promotionId": "PROMO002", 
      "promotionName": "一般客戶優惠",
      "promotionType": "GENERAL"
    }
  ]
}
```

**實際結果**: ✅ 符合預期，回傳 2 個優惠項目

**執行時間**: 245ms

**狀態**: 通過
```

### 反向測試案例範例

```markdown
## TC_PROMO_002 - 無效客戶編號查詢

**測試目標**: 驗證系統對無效客戶編號的錯誤處理

**需求追溯**: REQ_1.3, REQ_5.3

**前置條件**:
- 客戶編號 "INVALID_CUST" 不存在於系統中

**測試步驟**:
1. 準備測試資料：customerId = "INVALID_CUST"
2. 呼叫 POST /api/v1/promotions/query API
3. 驗證 HTTP 狀態碼為 200 (業務邏輯：不存在客戶回傳空清單)
4. 驗證回應包含空的優惠清單
5. 驗證錯誤日誌已記錄但不包含敏感資訊

**預期結果**:
```json
{
  "customerId": "INVALID_CUST",
  "applicablePromotions": []
}
```

**實際結果**: ✅ 符合預期，回傳空優惠清單

**執行時間**: 123ms

**狀態**: 通過
```

## 失敗案例分析格式

### 失敗案例詳細分析
```markdown
## 失敗案例分析

### TC_PROMO_003 - 規則引擎效能測試

**失敗原因**: 回應時間超過 500ms 閾值

**錯誤詳情**:
- 實際回應時間: 750ms
- 預期回應時間: < 500ms
- 錯誤類型: 效能不符合需求

**根因分析**:
1. 資料庫查詢未使用索引，導致全表掃描
2. 規則評估邏輯中存在 N+1 查詢問題
3. 快取機制未正確啟用

**修復建議**:
1. 在 customer_id 和 promotion_id 欄位上建立複合索引
2. 使用批次查詢替代迴圈查詢
3. 檢查 Redis 快取配置，確保正確啟用

**影響評估**:
- 影響範圍: 所有客戶查詢功能
- 嚴重程度: 高 (效能需求不符合)
- 修復優先級: P1 (立即修復)

**修復狀態**: 待修復
```

## 報表生成配置

### Maven 配置
```xml
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <configuration>
        <outputDirectory>${project.reporting.outputDirectory}/jacoco</outputDirectory>
        <formats>
            <format>HTML</format>
            <format>XML</format>
        </formats>
    </configuration>
</plugin>

<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <configuration>
        <reportFormat>xml</reportFormat>
        <includes>
            <include>**/*Test.java</include>
            <include>**/*Tests.java</include>
        </includes>
    </configuration>
</plugin>
```

### 報表模板配置
```yaml
test-report:
  output-formats: [HTML, PDF, EXCEL]
  template-path: src/test/resources/templates
  output-directory: target/test-reports
  include-screenshots: true
  include-logs: true
  ba-sa-friendly: true
  language: zh-TW
```

## 報表發布流程

### 自動化報表生成
1. **測試執行完成後自動觸發**
2. **生成多格式報表檔案**
3. **上傳到共享位置 (如 SharePoint, Confluence)**
4. **發送郵件通知相關人員**
5. **在 CI/CD 流程中展示報表連結**

### 報表存檔規範
- 報表檔案命名: `test-report-{version}-{timestamp}.{format}`
- 保存期限: 6 個月
- 存放位置: 專案文件庫的 test-reports 目錄
- 版本控制: 與程式碼版本對應

## 品質閘門

### 報表品質檢查
- 測試覆蓋率 ≥ 80%
- 功能測試通過率 ≥ 95%
- 效能測試通過率 = 100%
- 安全測試通過率 = 100%
- 關鍵路徑測試通過率 = 100%

### 發布條件
只有當所有品質閘門通過時，才能進行正式發布：
1. 所有 P1 和 P2 缺陷已修復
2. 測試覆蓋率達標
3. 效能指標符合需求
4. 安全掃描無高風險漏洞
5. BA/SA 已審核並簽核測試報表