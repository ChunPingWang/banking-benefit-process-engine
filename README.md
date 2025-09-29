# 銀行客戶優惠規則引擎

一個基於六角形架構設計的高效能銀行客戶優惠規則引擎，支援動態規則配置和複雜業務邏輯評估。

## 🏗️ 架構特色

- **六角形架構 (Hexagonal Architecture)** - 清晰的分層設計，核心業務邏輯與基礎設施完全分離
- **設計模式組合** - 整合命令模式、狀態模式、策略模式，實現高度可擴展的規則引擎
- **動態規則載入** - 使用 Java 反射技術，支援資料庫配置的動態判斷因子載入
- **SOLID 原則** - 遵循物件導向設計原則，確保程式碼的可維護性和擴展性

## 🛠️ 技術棧

- **Java 21** - 最新 LTS 版本，支援現代 Java 特性
- **Spring Boot 3** - 企業級應用框架
- **Spring Data JPA** - 資料持久化層
- **OpenAPI 3 / Swagger** - API 文件和測試介面
- **JUnit 5 & Mockito** - 單元測試和模擬框架
- **H2 Database** - 開發和測試環境
- **PostgreSQL** - 正式環境資料庫

## 📋 功能特色

### 核心功能
- 🔍 **優惠查詢服務** - 根據客戶編號和條件快速查詢適用優惠
- ⚙️ **規則管理** - 靈活的優惠規則配置和管理
- 👥 **客戶關聯管理** - 精確控制客戶與優惠的關聯關係
- 🧠 **智慧規則引擎** - 支援複雜條件邏輯的動態評估

### 技術特色
- 🚀 **高效能** - 單一查詢 < 500ms，支援 1000+ TPS
- 🔄 **動態配置** - 無需重啟即可更新規則配置
- 📊 **完整監控** - 詳細的效能指標和健康檢查
- 🛡️ **安全可靠** - 完整的錯誤處理和安全機制

## 🏛️ 系統架構

```
┌─────────────────────────────────────────────────────────────┐
│                    Infrastructure Layer                     │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────┐ │
│  │   REST API      │  │   Database      │  │  External   │ │
│  │   Controller    │  │   Repository    │  │  Services   │ │
│  └─────────────────┘  └─────────────────┘  └─────────────┘ │
└─────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────────┐
│                    Application Layer                        │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────┐ │
│  │   Use Cases     │  │   Application   │  │   DTO       │ │
│  │   Services      │  │   Services      │  │   Mappers   │ │
│  └─────────────────┘  └─────────────────┘  └─────────────┘ │
└─────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────────┐
│                      Domain Layer                           │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────┐ │
│  │   Rule Engine   │  │   State         │  │   Command   │ │
│  │   Core          │  │   Machine       │  │   Handlers  │ │
│  └─────────────────┘  └─────────────────┘  └─────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

## 🚀 快速開始

### 環境需求

- Java 21+
- Maven 3.8+
- Docker (可選，用於資料庫)

### 安裝步驟

1. **複製專案**
   ```bash
   git clone <repository-url>
   cd customer-promotion-engine
   ```

2. **編譯專案**
   ```bash
   mvn clean compile
   ```

3. **執行測試**
   ```bash
   mvn test
   ```

4. **啟動應用程式**
   ```bash
   # 開發環境 (使用 H2)
   mvn spring-boot:run -Dspring-boot.run.profiles=dev
   
   # 正式環境 (使用 PostgreSQL)
   mvn spring-boot:run -Dspring-boot.run.profiles=prod
   ```

### Docker 部署

```bash
# 建立映像檔
docker build -t promotion-engine .

# 執行容器
docker run -p 8080:8080 promotion-engine
```

## 📖 API 文件

啟動應用程式後，可透過以下網址存取 API 文件：

- **Swagger UI**: http://localhost:8080/swagger-ui.html
- **OpenAPI JSON**: http://localhost:8080/v3/api-docs

### 主要 API 端點

```http
POST /api/v1/promotions/query
Content-Type: application/json

{
  "customerId": "CUST001",
  "additionalContext": {
    "accountType": "PREMIUM",
    "transactionAmount": 10000
  }
}
```

## 🧪 測試策略

### 測試層級

- **單元測試** - 測試個別元件和業務邏輯
- **整合測試** - 測試元件間的互動
- **端對端測試** - 測試完整的業務流程

### 執行測試

```bash
# 執行所有測試
mvn test

# 執行特定測試類別
mvn test -Dtest=PromotionRuleEngineTest

# 執行整合測試
mvn test -Dtest=*IntegrationTest

# 產生測試報告
mvn jacoco:report
```

### 測試環境

- **開發測試**: H2 記憶體資料庫
- **整合測試**: PostgreSQL 測試容器
- **橋接模式**: 支援不同資料庫環境的無縫切換

## 📊 監控和維運

### 健康檢查

```bash
# 應用程式健康狀態
curl http://localhost:8080/actuator/health

# 詳細健康資訊
curl http://localhost:8080/actuator/health/detailed
```

### 效能指標

- **回應時間**: 單一查詢 < 500ms
- **吞吐量**: 支援 1000+ TPS
- **可用性**: 99.9% SLA

### 日誌監控

```bash
# 查看應用程式日誌
docker logs promotion-engine

# 即時監控日誌
docker logs -f promotion-engine
```

## 🔧 配置說明

### 應用程式配置

```yaml
# application.yml
spring:
  profiles:
    active: dev
  
promotion-engine:
  rule-cache:
    ttl: 300s
    max-size: 1000
  performance:
    query-timeout: 2s
    max-concurrent-requests: 100
```

### 資料庫配置

```yaml
# 開發環境 (H2)
spring:
  datasource:
    url: jdbc:h2:mem:promotiondb
    driver-class-name: org.h2.Driver
  jpa:
    database-platform: org.hibernate.dialect.H2Dialect

# 正式環境 (PostgreSQL)
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/promotiondb
    driver-class-name: org.postgresql.Driver
  jpa:
    database-platform: org.hibernate.dialect.PostgreSQLDialect
```

## 🏗️ 開發指南

### 新增判斷因子

1. 建立新的判斷因子類別：

```java
@Component
public class NewJudgmentFactor extends AbstractJudgmentFactor {
    @Override
    protected boolean doEvaluate(CustomerContext context) {
        // 實作評估邏輯
        return true;
    }
}
```

2. 在資料庫中配置判斷因子：

```sql
INSERT INTO judgment_factor_configuration 
(factor_id, factor_class_name, priority, is_active) 
VALUES ('NEW_FACTOR', 'com.bank.promotion.domain.factor.NewJudgmentFactor', 10, true);
```

### 新增規則路徑

1. 建立規則路徑類別：

```java
@Component
public class NewRulePath extends AbstractRulePath {
    @Override
    protected PromotionResult doExecute(CustomerContext context) {
        // 實作執行邏輯
        return PromotionResult.builder()
            .promotionId("PROMO001")
            .promotionName("新優惠方案")
            .build();
    }
}
```

## 📚 專案結構

```
src/
├── main/
│   ├── java/
│   │   └── com/bank/promotion/
│   │       ├── application/          # 應用層
│   │       ├── domain/              # 領域層
│   │       ├── infrastructure/      # 基礎設施層
│   │       └── PromotionEngineApplication.java
│   └── resources/
│       ├── application.yml
│       └── db/migration/           # 資料庫遷移腳本
└── test/
    ├── java/
    │   └── com/bank/promotion/
    │       ├── unit/               # 單元測試
    │       ├── integration/        # 整合測試
    │       └── e2e/               # 端對端測試
    └── resources/
        └── application-test.yml
```

## 🤝 貢獻指南

1. Fork 專案
2. 建立功能分支 (`git checkout -b feature/amazing-feature`)
3. 提交變更 (`git commit -m 'Add amazing feature'`)
4. 推送到分支 (`git push origin feature/amazing-feature`)
5. 開啟 Pull Request

## 📄 授權

本專案採用 MIT 授權條款 - 詳見 [LICENSE](LICENSE) 檔案

## 📞 聯絡資訊

- **專案維護者**: [Your Name]
- **Email**: [your.email@bank.com]
- **專案首頁**: [Project URL]

---

## 📋 Spec 文件

本專案採用 Spec Driven Development 方法論，完整的規格文件位於：

- 📋 [需求文件](.kiro/specs/customer-promotion-engine/requirements.md)
- 🏗️ [設計文件](.kiro/specs/customer-promotion-engine/design.md)
- ✅ [任務清單](.kiro/specs/customer-promotion-engine/tasks.md)

### 開發工作流程

1. **需求階段** - 定義用戶故事和驗收標準
2. **設計階段** - 制定技術架構和實作方案
3. **任務階段** - 分解為具體的開發任務
4. **執行階段** - 按照任務清單逐步實作

要開始開發，請參考 [任務清單](.kiro/specs/customer-promotion-engine/tasks.md) 並點擊 "Start task" 開始執行特定任務。