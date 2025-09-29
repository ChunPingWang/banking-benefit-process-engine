# 設計文件 - 銀行客戶優惠規則引擎

## 概述

銀行客戶優惠規則引擎採用六角形架構設計，結合狀態模式、策略模式和命令模式，實現高度可配置和可擴展的規則評估系統。系統核心採用反射技術動態載入判斷因子，透過資料庫配置實現靈活的規則組合和執行順序控制。

## 架構

### 六角形架構分層

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
│  │   Entities      │  │   Value         │  │   Domain    │ │
│  │   Aggregates    │  │   Objects       │  │   Services  │ │
│  └─────────────────┘  └─────────────────┘  └─────────────┘ │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────┐ │
│  │   Rule Engine   │  │   State         │  │   Command   │ │
│  │   Core          │  │   Machine       │  │   Handlers  │ │
│  └─────────────────┘  └─────────────────┘  └─────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

## 元件和介面

### 核心領域介面

```java
// 命令模式 - 判斷因子介面
public interface JudgmentFactor {
    String getFactorName();
    boolean evaluate(CustomerContext context);
    int getPriority();
}

// 狀態模式 - 規則路徑介面
public interface RulePath {
    String getPathId();
    boolean canTransition(CustomerContext context);
    PromotionResult execute(CustomerContext context);
    List<JudgmentFactor> getJudgmentFactors();
}

// 策略模式 - 規則評估策略
public interface RuleEvaluationStrategy {
    List<PromotionResult> evaluate(String customerId, CustomerContext context);
}

// 規則引擎核心介面
public interface PromotionRuleEngine {
    List<PromotionResult> findApplicablePromotions(String customerId, Map<String, Object> additionalContext);
    void refreshRuleConfiguration();
}
```

### 應用層服務介面

```java
// 優惠查詢服務
public interface PromotionQueryService {
    PromotionQueryResponse queryCustomerPromotions(PromotionQueryRequest request);
}

// 規則管理服務
public interface RuleManagementService {
    void createRule(CreateRuleRequest request);
    void updateRule(UpdateRuleRequest request);
    void deleteRule(String ruleId);
    RuleConfigurationResponse getRuleConfiguration(String ruleId);
}

// 客戶關聯管理服務
public interface CustomerPromotionService {
    void addCustomerToPromotion(String customerId, String promotionId);
    void removeCustomerFromPromotion(String customerId, String promotionId);
    List<CustomerPromotionMapping> getCustomerPromotions(String customerId);
}
```

### 基礎設施層介面

```java
// 規則配置儲存庫
public interface RuleConfigurationRepository {
    List<RuleConfiguration> findActiveRules();
    Optional<RuleConfiguration> findById(String ruleId);
    void save(RuleConfiguration ruleConfiguration);
}

// 判斷因子載入器
public interface JudgmentFactorLoader {
    JudgmentFactor loadFactor(String className);
    List<JudgmentFactor> loadFactorsForRule(String ruleId);
}
```

## 資料模型

### 核心實體

```java
// 優惠規則聚合根
@Entity
public class PromotionRule {
    private String ruleId;
    private String ruleName;
    private String description;
    private RuleStatus status;
    private LocalDateTime effectiveDate;
    private LocalDateTime expiryDate;
    private List<RulePathConfiguration> rulePaths;
    private RuleMetadata metadata;
}

// 規則路徑配置
@Entity
public class RulePathConfiguration {
    private String pathId;
    private String pathName;
    private int executionOrder;
    private List<JudgmentFactorConfiguration> judgmentFactors;
    private String nextPathId;
    private PathCondition transitionCondition;
}

// 判斷因子配置
@Entity
public class JudgmentFactorConfiguration {
    private String factorId;
    private String factorClassName;
    private int priority;
    private Map<String, Object> parameters;
    private boolean isActive;
}

// 客戶優惠關聯
@Entity
public class CustomerPromotionMapping {
    private String customerId;
    private String promotionId;
    private LocalDateTime assignedDate;
    private LocalDateTime lastEvaluatedDate;
    private MappingStatus status;
    private Map<String, Object> customerAttributes;
}
```

### 值物件

```java
// 客戶上下文
public class CustomerContext {
    private final String customerId;
    private final CustomerProfile profile;
    private final Map<String, Object> additionalAttributes;
    private final LocalDateTime evaluationTime;
}

// 優惠結果
public class PromotionResult {
    private final String promotionId;
    private final String promotionName;
    private final PromotionType type;
    private final Map<String, Object> promotionDetails;
    private final List<String> appliedRulePaths;
}
```

## 設計模式實作

### 命令模式 - 判斷因子

```java
// 抽象判斷因子
public abstract class AbstractJudgmentFactor implements JudgmentFactor {
    protected final String factorName;
    protected final int priority;
    protected final Map<String, Object> parameters;
    
    @Override
    public final boolean evaluate(CustomerContext context) {
        try {
            return doEvaluate(context);
        } catch (Exception e) {
            // 記錄錯誤並回傳預設值
            return getDefaultResult();
        }
    }
    
    protected abstract boolean doEvaluate(CustomerContext context);
    protected abstract boolean getDefaultResult();
}

// 具體判斷因子範例
@Component
public class AgeRangeJudgmentFactor extends AbstractJudgmentFactor {
    @Override
    protected boolean doEvaluate(CustomerContext context) {
        int customerAge = context.getProfile().getAge();
        int minAge = (Integer) parameters.get("minAge");
        int maxAge = (Integer) parameters.get("maxAge");
        return customerAge >= minAge && customerAge <= maxAge;
    }
}
```

### 狀態模式 - 規則路徑

```java
// 抽象規則路徑
public abstract class AbstractRulePath implements RulePath {
    protected final String pathId;
    protected final List<JudgmentFactor> judgmentFactors;
    protected final RulePathConfiguration configuration;
    
    @Override
    public final boolean canTransition(CustomerContext context) {
        return judgmentFactors.stream()
            .sorted(Comparator.comparing(JudgmentFactor::getPriority))
            .allMatch(factor -> factor.evaluate(context));
    }
    
    @Override
    public final PromotionResult execute(CustomerContext context) {
        if (!canTransition(context)) {
            return PromotionResult.empty();
        }
        return doExecute(context);
    }
    
    protected abstract PromotionResult doExecute(CustomerContext context);
}
```

### 策略模式 - 規則評估

```java
// 預設規則評估策略
@Component
public class DefaultRuleEvaluationStrategy implements RuleEvaluationStrategy {
    
    private final RuleConfigurationRepository ruleRepository;
    private final JudgmentFactorLoader factorLoader;
    
    @Override
    public List<PromotionResult> evaluate(String customerId, CustomerContext context) {
        List<RuleConfiguration> activeRules = ruleRepository.findActiveRules();
        
        return activeRules.stream()
            .map(rule -> evaluateRule(rule, context))
            .filter(Objects::nonNull)
            .collect(Collectors.toList());
    }
    
    private PromotionResult evaluateRule(RuleConfiguration rule, CustomerContext context) {
        List<RulePath> rulePaths = loadRulePaths(rule);
        
        return rulePaths.stream()
            .sorted(Comparator.comparing(RulePath::getExecutionOrder))
            .filter(path -> path.canTransition(context))
            .findFirst()
            .map(path -> path.execute(context))
            .orElse(null);
    }
}
```

## 反射技術實作

### 動態判斷因子載入器

```java
@Component
public class ReflectionJudgmentFactorLoader implements JudgmentFactorLoader {
    
    private final ApplicationContext applicationContext;
    private final Map<String, JudgmentFactor> factorCache = new ConcurrentHashMap<>();
    
    @Override
    public JudgmentFactor loadFactor(String className) {
        return factorCache.computeIfAbsent(className, this::createFactorInstance);
    }
    
    private JudgmentFactor createFactorInstance(String className) {
        try {
            Class<?> factorClass = Class.forName(className);
            
            // 優先從 Spring 容器取得
            if (applicationContext.containsBean(factorClass)) {
                return (JudgmentFactor) applicationContext.getBean(factorClass);
            }
            
            // 否則使用反射建立實例
            Constructor<?> constructor = factorClass.getDeclaredConstructor();
            return (JudgmentFactor) constructor.newInstance();
            
        } catch (Exception e) {
            throw new RuleEngineException("Failed to load judgment factor: " + className, e);
        }
    }
}
```

## 錯誤處理

### 異常處理策略

```java
// 規則引擎異常
public class RuleEngineException extends RuntimeException {
    private final String errorCode;
    private final Map<String, Object> context;
}

// 全域異常處理器
@ControllerAdvice
public class GlobalExceptionHandler {
    
    @ExceptionHandler(RuleEngineException.class)
    public ResponseEntity<ErrorResponse> handleRuleEngineException(RuleEngineException e) {
        ErrorResponse response = ErrorResponse.builder()
            .errorCode(e.getErrorCode())
            .message("規則引擎處理異常")
            .timestamp(LocalDateTime.now())
            .build();
        
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(response);
    }
}
```

## 測試策略

### 單元測試

```java
// 判斷因子測試
@ExtendWith(MockitoExtension.class)
class AgeRangeJudgmentFactorTest {
    
    @Test
    void shouldReturnTrueWhenAgeInRange() {
        // Given
        CustomerContext context = createCustomerContext(25);
        AgeRangeJudgmentFactor factor = new AgeRangeJudgmentFactor(18, 30);
        
        // When
        boolean result = factor.evaluate(context);
        
        // Then
        assertThat(result).isTrue();
    }
}
```

### 整合測試

```java
// 規則引擎整合測試
@SpringBootTest
@TestPropertySource(properties = "spring.datasource.url=jdbc:h2:mem:testdb")
class PromotionRuleEngineIntegrationTest {
    
    @Test
    void shouldFindApplicablePromotions() {
        // Given
        String customerId = "CUST001";
        Map<String, Object> context = Map.of("accountType", "PREMIUM");
        
        // When
        List<PromotionResult> results = ruleEngine.findApplicablePromotions(customerId, context);
        
        // Then
        assertThat(results).isNotEmpty();
    }
}
```

### 資料庫橋接模式測試

```java
// 資料庫測試橋接
public interface DatabaseTestBridge {
    void setupTestData();
    void cleanupTestData();
    void verifyDataState();
}

@Component
@Profile("test")
public class H2DatabaseTestBridge implements DatabaseTestBridge {
    // H2 特定的測試實作
}

@Component
@Profile("integration")
public class PostgreSQLDatabaseTestBridge implements DatabaseTestBridge {
    // PostgreSQL 特定的測試實作
}
```

## OpenAPI 規格

### API 端點定義

```java
@RestController
@RequestMapping("/api/v1/promotions")
@Tag(name = "優惠查詢", description = "客戶優惠查詢相關 API")
public class PromotionQueryController {
    
    @PostMapping("/query")
    @Operation(summary = "查詢客戶適用優惠", description = "根據客戶編號和條件查詢適用的優惠方案")
    @ApiResponses({
        @ApiResponse(responseCode = "200", description = "查詢成功"),
        @ApiResponse(responseCode = "400", description = "請求參數錯誤"),
        @ApiResponse(responseCode = "500", description = "系統內部錯誤")
    })
    public ResponseEntity<PromotionQueryResponse> queryPromotions(
        @Valid @RequestBody PromotionQueryRequest request) {
        
        PromotionQueryResponse response = promotionQueryService.queryCustomerPromotions(request);
        return ResponseEntity.ok(response);
    }
}
```

## 效能考量

### 快取策略

```java
@Service
public class CachedRuleEvaluationService {
    
    @Cacheable(value = "ruleConfigurations", key = "#ruleId")
    public RuleConfiguration getRuleConfiguration(String ruleId) {
        return ruleRepository.findById(ruleId);
    }
    
    @CacheEvict(value = "ruleConfigurations", allEntries = true)
    public void refreshRuleCache() {
        // 清除快取，強制重新載入規則
    }
}
```

### 非同步處理

```java
@Service
public class AsyncPromotionEvaluationService {
    
    @Async("promotionExecutor")
    public CompletableFuture<List<PromotionResult>> evaluatePromotionsAsync(
        String customerId, CustomerContext context) {
        
        List<PromotionResult> results = ruleEngine.findApplicablePromotions(customerId, context);
        return CompletableFuture.completedFuture(results);
    }
}
```