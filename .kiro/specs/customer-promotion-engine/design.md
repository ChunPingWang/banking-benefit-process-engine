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

## Mapper 轉換層設計

### 六角形架構邊界轉換原則

所有外層物件（REST DTO、JPA Entity、外部系統物件）都必須透過 Mapper 轉換為領域物件，嚴格禁止外層物件直接進入核心層。

### 轉換層架構

```
┌─────────────────────────────────────────────────────────────┐
│                Infrastructure Layer                         │
│  REST DTOs, JPA Entities, External System Objects          │
└─────────────────────┬───────────────────────────────────────┘
                      │ Mapper 轉換
┌─────────────────────▼───────────────────────────────────────┐
│                Application Layer                            │
│  Application Services (使用純領域物件)                      │
└─────────────────────┬───────────────────────────────────────┘
                      │ 直接使用
┌─────────────────────▼───────────────────────────────────────┐
│                  Domain Layer                               │
│  純領域物件 (無任何外層依賴)                                │
└─────────────────────────────────────────────────────────────┘
```

### Mapper 介面定義

```java
// 通用 Mapper 介面
public interface DomainMapper<D, E> {
    D toDomain(E external);
    E toExternal(D domain);
    List<D> toDomainList(List<E> externalList);
    List<E> toExternalList(List<D> domainList);
}

// 優惠查詢 Mapper
public interface PromotionQueryMapper {
    CustomerContext toCustomerContext(PromotionQueryRequest request, CustomerProfileEntity customerEntity);
    PromotionQueryResponse toResponse(List<PromotionResult> domainResults);
}

// 規則配置 Mapper
public interface RuleConfigurationMapper extends DomainMapper<PromotionRule, RuleConfigurationEntity> {
    PromotionRule toDomain(RuleConfigurationEntity entity);
    RuleConfigurationEntity toEntity(PromotionRule domain);
    
    // 特殊轉換方法
    List<JudgmentFactor> toJudgmentFactors(List<JudgmentFactorEntity> entities);
    List<RulePath> toRulePaths(List<RulePathEntity> entities);
}

// 客戶資料 Mapper
public interface CustomerMapper extends DomainMapper<CustomerProfile, CustomerEntity> {
    CustomerProfile toDomain(CustomerEntity entity);
    CustomerPromotionMapping toPromotionMapping(CustomerPromotionEntity entity);
}
```

### Mapper 實作範例

```java
// 優惠查詢 Mapper 實作
public class PromotionQueryMapperImpl implements PromotionQueryMapper {
    
    @Override
    public CustomerContext toCustomerContext(PromotionQueryRequest request, CustomerProfileEntity customerEntity) {
        // 將外層 DTO 和 Entity 轉換為純領域物件
        CustomerProfile profile = CustomerProfile.builder()
            .customerId(customerEntity.getCustomerId())
            .customerName(customerEntity.getCustomerName())
            .age(customerEntity.getAge())
            .accountType(AccountType.valueOf(customerEntity.getAccountType()))
            .customerStatus(CustomerStatus.valueOf(customerEntity.getStatus()))
            .registrationDate(customerEntity.getRegistrationDate())
            .build();
        
        Map<String, Object> additionalAttributes = new HashMap<>();
        if (request.getAdditionalContext() != null) {
            additionalAttributes.putAll(request.getAdditionalContext());
        }
        
        return new CustomerContext(
            request.getCustomerId(),
            profile,
            additionalAttributes,
            LocalDateTime.now()
        );
    }
    
    @Override
    public PromotionQueryResponse toResponse(List<PromotionResult> domainResults) {
        // 將領域物件轉換為外層 DTO
        List<PromotionDto> promotionDtos = domainResults.stream()
            .map(this::toPromotionDto)
            .collect(Collectors.toList());
        
        return PromotionQueryResponse.builder()
            .applicablePromotions(promotionDtos)
            .totalCount(promotionDtos.size())
            .queryTimestamp(LocalDateTime.now())
            .build();
    }
    
    private PromotionDto toPromotionDto(PromotionResult domainResult) {
        return PromotionDto.builder()
            .promotionId(domainResult.getPromotionId())
            .promotionName(domainResult.getPromotionName())
            .promotionType(domainResult.getType().name())
            .promotionDetails(domainResult.getPromotionDetails())
            .appliedRulePaths(domainResult.getAppliedRulePaths())
            .build();
    }
}

// 規則配置 Mapper 實作
public class RuleConfigurationMapperImpl implements RuleConfigurationMapper {
    
    private final JudgmentFactorLoader factorLoader;
    
    public RuleConfigurationMapperImpl(JudgmentFactorLoader factorLoader) {
        this.factorLoader = factorLoader;
    }
    
    @Override
    public PromotionRule toDomain(RuleConfigurationEntity entity) {
        // 將 JPA Entity 轉換為純領域物件
        PromotionRule rule = new PromotionRule(
            entity.getRuleId(),
            entity.getRuleName(),
            entity.getDescription()
        );
        
        rule.setStatus(RuleStatus.valueOf(entity.getStatus()));
        rule.setEffectiveDate(entity.getEffectiveDate());
        rule.setExpiryDate(entity.getExpiryDate());
        
        // 轉換規則路徑
        List<RulePath> rulePaths = toRulePaths(entity.getRulePathEntities());
        rule.setRulePaths(rulePaths);
        
        return rule;
    }
    
    @Override
    public RuleConfigurationEntity toEntity(PromotionRule domain) {
        // 將領域物件轉換為 JPA Entity
        RuleConfigurationEntity entity = new RuleConfigurationEntity();
        entity.setRuleId(domain.getRuleId());
        entity.setRuleName(domain.getRuleName());
        entity.setDescription(domain.getDescription());
        entity.setStatus(domain.getStatus().name());
        entity.setEffectiveDate(domain.getEffectiveDate());
        entity.setExpiryDate(domain.getExpiryDate());
        
        // 轉換規則路徑實體
        List<RulePathEntity> pathEntities = toRulePathEntities(domain.getRulePaths());
        entity.setRulePathEntities(pathEntities);
        
        return entity;
    }
    
    @Override
    public List<JudgmentFactor> toJudgmentFactors(List<JudgmentFactorEntity> entities) {
        return entities.stream()
            .map(this::toJudgmentFactor)
            .collect(Collectors.toList());
    }
    
    private JudgmentFactor toJudgmentFactor(JudgmentFactorEntity entity) {
        try {
            // 使用反射載入判斷因子，但配置透過 Mapper 轉換
            Map<String, Object> domainConfiguration = convertEntityParametersToDomainConfiguration(
                entity.getParameters()
            );
            
            return factorLoader.loadFactor(entity.getFactorClassName(), domainConfiguration);
        } catch (Exception e) {
            throw new MappingException("Failed to map JudgmentFactor: " + entity.getFactorClassName(), e);
        }
    }
    
    private Map<String, Object> convertEntityParametersToDomainConfiguration(String jsonParameters) {
        // 將 JSON 字串轉換為領域配置物件
        // 這裡可以使用純 Java 的 JSON 解析，不依賴框架
        try {
            return JsonParser.parseToMap(jsonParameters);
        } catch (Exception e) {
            return new HashMap<>();
        }
    }
    
    private List<RulePathEntity> toRulePathEntities(List<RulePath> domainPaths) {
        return domainPaths.stream()
            .map(this::toRulePathEntity)
            .collect(Collectors.toList());
    }
    
    private RulePathEntity toRulePathEntity(RulePath domainPath) {
        RulePathEntity entity = new RulePathEntity();
        entity.setPathId(domainPath.getPathId());
        entity.setPathName(domainPath.getPathName());
        
        // 將判斷因子轉換為實體
        List<JudgmentFactorEntity> factorEntities = domainPath.getJudgmentFactors().stream()
            .map(this::toJudgmentFactorEntity)
            .collect(Collectors.toList());
        entity.setJudgmentFactorEntities(factorEntities);
        
        return entity;
    }
    
    private JudgmentFactorEntity toJudgmentFactorEntity(JudgmentFactor domainFactor) {
        JudgmentFactorEntity entity = new JudgmentFactorEntity();
        entity.setFactorId(UUID.randomUUID().toString());
        entity.setFactorClassName(domainFactor.getClass().getName());
        entity.setPriority(domainFactor.getPriority());
        entity.setIsActive(true);
        
        // 將領域配置轉換為 JSON 字串
        String jsonParameters = JsonSerializer.toJson(domainFactor.getConfiguration());
        entity.setParameters(jsonParameters);
        
        return entity;
    }
}
```

### 邊界控制實作

```java
// REST 控制器 - 嚴格使用 Mapper 轉換
@RestController
@RequestMapping("/api/v1/promotions")
public class PromotionQueryController {
    
    private final PromotionQueryService promotionQueryService;
    private final PromotionQueryMapper promotionQueryMapper;
    private final CustomerRepository customerRepository;
    
    @PostMapping("/query")
    public ResponseEntity<PromotionQueryResponse> queryPromotions(
            @Valid @RequestBody PromotionQueryRequest request) {
        
        // 1. 從外層獲取資料
        CustomerProfileEntity customerEntity = customerRepository.findEntityById(request.getCustomerId())
            .orElseThrow(() -> new CustomerNotFoundException(request.getCustomerId()));
        
        // 2. 透過 Mapper 轉換為領域物件
        CustomerContext customerContext = promotionQueryMapper.toCustomerContext(request, customerEntity);
        
        // 3. 呼叫應用服務 (只傳入領域物件)
        List<PromotionResult> results = promotionQueryService.queryCustomerPromotions(
            request.getCustomerId(), customerContext);
        
        // 4. 透過 Mapper 轉換回外層 DTO
        PromotionQueryResponse response = promotionQueryMapper.toResponse(results);
        
        return ResponseEntity.ok(response);
    }
}

// JPA Repository 實作 - 嚴格使用 Mapper 轉換
@Repository
public class JpaRuleConfigurationRepository implements RuleConfigurationRepository {
    
    private final RuleConfigurationJpaRepository jpaRepository;
    private final RuleConfigurationMapper mapper;
    
    @Override
    public List<PromotionRule> findActiveRules() {
        // 1. 從 JPA 獲取實體
        List<RuleConfigurationEntity> entities = jpaRepository.findByStatusAndIsActiveTrue("ACTIVE");
        
        // 2. 透過 Mapper 轉換為領域物件
        return entities.stream()
            .map(mapper::toDomain)
            .collect(Collectors.toList());
    }
    
    @Override
    public Optional<PromotionRule> findById(String ruleId) {
        // 1. 從 JPA 獲取實體
        Optional<RuleConfigurationEntity> entityOpt = jpaRepository.findById(ruleId);
        
        // 2. 透過 Mapper 轉換為領域物件
        return entityOpt.map(mapper::toDomain);
    }
    
    @Override
    public void save(PromotionRule rule) {
        // 1. 透過 Mapper 轉換為 JPA 實體
        RuleConfigurationEntity entity = mapper.toEntity(rule);
        
        // 2. 儲存到資料庫
        jpaRepository.save(entity);
    }
}
```

### 轉換驗證機制

```java
// Mapper 驗證器
public class MapperValidator {
    
    public static void validateDomainObject(Object domainObject) {
        if (domainObject == null) {
            throw new MappingException("Domain object cannot be null");
        }
        
        // 檢查是否包含外層框架依賴
        Class<?> clazz = domainObject.getClass();
        if (hasFrameworkAnnotations(clazz)) {
            throw new MappingException(
                "Domain object contains framework annotations: " + clazz.getName());
        }
        
        if (hasFrameworkDependencies(clazz)) {
            throw new MappingException(
                "Domain object has framework dependencies: " + clazz.getName());
        }
    }
    
    private static boolean hasFrameworkAnnotations(Class<?> clazz) {
        return Arrays.stream(clazz.getAnnotations())
            .anyMatch(annotation -> {
                String packageName = annotation.annotationType().getPackage().getName();
                return packageName.startsWith("org.springframework") ||
                       packageName.startsWith("javax.persistence") ||
                       packageName.startsWith("jakarta.persistence");
            });
    }
    
    private static boolean hasFrameworkDependencies(Class<?> clazz) {
        return Arrays.stream(clazz.getDeclaredFields())
            .anyMatch(field -> {
                String typeName = field.getType().getPackage().getName();
                return typeName.startsWith("org.springframework") ||
                       typeName.startsWith("javax.persistence") ||
                       typeName.startsWith("jakarta.persistence");
            });
    }
}
```

## 元件和介面

### 核心領域介面 (純 Java，無框架依賴)

```java
// 命令模式 - 判斷因子介面 (領域核心，純 Java)
public interface JudgmentFactor {
    String getFactorName();
    boolean evaluate(CustomerContext context);
    int getPriority();
    Map<String, Object> getConfiguration();
    void configure(Map<String, Object> parameters);
}

// 狀態模式 - 規則路徑介面 (領域核心，純 Java)
public interface RulePath {
    String getPathId();
    String getPathName();
    boolean canTransition(CustomerContext context);
    PromotionResult execute(CustomerContext context);
    List<JudgmentFactor> getJudgmentFactors();
    RulePath getNextPath();
    void addJudgmentFactor(JudgmentFactor factor);
    void configure(RulePathConfiguration configuration);
}

// 策略模式 - 規則評估策略 (領域核心，純 Java)
public interface RuleEvaluationStrategy {
    List<PromotionResult> evaluate(String customerId, CustomerContext context);
    String getStrategyName();
    void configure(Map<String, Object> configuration);
}

// 規則引擎核心介面 (領域核心，純 Java)
public interface PromotionRuleEngine {
    List<PromotionResult> findApplicablePromotions(String customerId, Map<String, Object> additionalContext);
    void refreshRuleConfiguration();
    void registerJudgmentFactor(String className, Map<String, Object> configuration);
    void registerRulePath(String className, Map<String, Object> configuration);
}
```

### 應用層服務介面 (使用領域物件，不依賴外層)

```java
// 優惠查詢服務 - 使用純領域物件
public interface PromotionQueryService {
    List<PromotionResult> queryCustomerPromotions(String customerId, CustomerContext context);
}

// 規則管理服務 - 使用純領域物件
public interface RuleManagementService {
    void createRule(PromotionRule rule);
    void updateRule(PromotionRule rule);
    void deleteRule(String ruleId);
    Optional<PromotionRule> getRuleById(String ruleId);
    List<PromotionRule> getAllActiveRules();
}

// 客戶關聯管理服務 - 使用純領域物件
public interface CustomerPromotionService {
    void addCustomerToPromotion(CustomerPromotionMapping mapping);
    void removeCustomerFromPromotion(String customerId, String promotionId);
    List<CustomerPromotionMapping> getCustomerPromotions(String customerId);
    boolean isCustomerEligible(String customerId, String promotionId);
}
```

### 基礎設施層介面 (Port 介面，使用領域物件)

```java
// 規則配置儲存庫 Port - 只使用領域物件
public interface RuleConfigurationRepository {
    List<PromotionRule> findActiveRules();
    Optional<PromotionRule> findById(String ruleId);
    void save(PromotionRule rule);
    void delete(String ruleId);
}

// 客戶資料儲存庫 Port - 只使用領域物件
public interface CustomerRepository {
    Optional<CustomerProfile> findById(String customerId);
    List<CustomerPromotionMapping> findCustomerPromotions(String customerId);
    void saveCustomerPromotion(CustomerPromotionMapping mapping);
}

// 判斷因子載入器 Port - 只使用領域物件
public interface JudgmentFactorLoader {
    JudgmentFactor loadFactor(String className, Map<String, Object> configuration);
    List<JudgmentFactor> loadFactorsForRule(String ruleId);
}

// 外部系統整合 Port - 只使用領域物件
public interface ExternalSystemPort {
    CustomerProfile getCustomerProfile(String customerId);
    void notifyPromotionAssigned(String customerId, PromotionResult promotion);
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

### 命令模式 - 判斷因子 (充血模式，純 Java 實作)

```java
// 抽象判斷因子 - 充血模式，包含業務邏輯和狀態
public abstract class AbstractJudgmentFactor implements JudgmentFactor {
    private final String factorName;
    private int priority;
    private Map<String, Object> configuration;
    private boolean isActive;
    private LocalDateTime lastModified;
    
    protected AbstractJudgmentFactor(String factorName) {
        this.factorName = factorName;
        this.configuration = new HashMap<>();
        this.isActive = true;
        this.lastModified = LocalDateTime.now();
    }
    
    @Override
    public final boolean evaluate(CustomerContext context) {
        if (!isActive) {
            return getDefaultResult();
        }
        
        try {
            validateConfiguration();
            return doEvaluate(context);
        } catch (Exception e) {
            logEvaluationError(e, context);
            return getDefaultResult();
        }
    }
    
    @Override
    public void configure(Map<String, Object> parameters) {
        this.configuration = new HashMap<>(parameters);
        this.lastModified = LocalDateTime.now();
        validateConfiguration();
    }
    
    @Override
    public Map<String, Object> getConfiguration() {
        return new HashMap<>(configuration);
    }
    
    // 業務邏輯方法 - 充血模式
    public void activate() {
        this.isActive = true;
        this.lastModified = LocalDateTime.now();
    }
    
    public void deactivate() {
        this.isActive = false;
        this.lastModified = LocalDateTime.now();
    }
    
    public boolean isConfigurationValid() {
        try {
            validateConfiguration();
            return true;
        } catch (Exception e) {
            return false;
        }
    }
    
    public void updatePriority(int newPriority) {
        this.priority = newPriority;
        this.lastModified = LocalDateTime.now();
    }
    
    // 抽象方法供子類實作
    protected abstract boolean doEvaluate(CustomerContext context);
    protected abstract boolean getDefaultResult();
    protected abstract void validateConfiguration();
    
    // 內部方法
    private void logEvaluationError(Exception e, CustomerContext context) {
        // 純 Java 日誌記錄，不依賴框架
        System.err.println(String.format(
            "JudgmentFactor [%s] evaluation failed for customer [%s]: %s",
            factorName, context.getCustomerId(), e.getMessage()
        ));
    }
    
    // Getters
    @Override
    public String getFactorName() { return factorName; }
    
    @Override
    public int getPriority() { return priority; }
    
    public boolean isActive() { return isActive; }
    
    public LocalDateTime getLastModified() { return lastModified; }
}

// 具體判斷因子範例 - 純 Java，無框架註解
public class AgeRangeJudgmentFactor extends AbstractJudgmentFactor {
    
    public AgeRangeJudgmentFactor() {
        super("AgeRangeJudgmentFactor");
    }
    
    @Override
    protected boolean doEvaluate(CustomerContext context) {
        int customerAge = context.getProfile().getAge();
        int minAge = getConfigurationValue("minAge", Integer.class);
        int maxAge = getConfigurationValue("maxAge", Integer.class);
        
        return customerAge >= minAge && customerAge <= maxAge;
    }
    
    @Override
    protected boolean getDefaultResult() {
        return false; // 預設不符合條件
    }
    
    @Override
    protected void validateConfiguration() {
        requireConfigurationKey("minAge", Integer.class);
        requireConfigurationKey("maxAge", Integer.class);
        
        int minAge = getConfigurationValue("minAge", Integer.class);
        int maxAge = getConfigurationValue("maxAge", Integer.class);
        
        if (minAge < 0 || maxAge < 0 || minAge > maxAge) {
            throw new IllegalArgumentException("Invalid age range configuration");
        }
    }
    
    // 輔助方法
    private <T> T getConfigurationValue(String key, Class<T> type) {
        Object value = getConfiguration().get(key);
        if (value == null) {
            throw new IllegalArgumentException("Missing configuration key: " + key);
        }
        return type.cast(value);
    }
    
    private void requireConfigurationKey(String key, Class<?> type) {
        Object value = getConfiguration().get(key);
        if (value == null || !type.isInstance(value)) {
            throw new IllegalArgumentException(
                String.format("Configuration key '%s' must be of type %s", key, type.getSimpleName())
            );
        }
    }
}
```

### 狀態模式 - 規則路徑 (充血模式，純 Java 實作)

```java
// 抽象規則路徑 - 充血模式，包含完整的業務邏輯和狀態管理
public abstract class AbstractRulePath implements RulePath {
    private final String pathId;
    private String pathName;
    private final List<JudgmentFactor> judgmentFactors;
    private RulePath nextPath;
    private boolean isActive;
    private int executionOrder;
    private LocalDateTime lastExecuted;
    private long executionCount;
    private Map<String, Object> pathConfiguration;
    private List<String> executionHistory;
    
    protected AbstractRulePath(String pathId, String pathName) {
        this.pathId = pathId;
        this.pathName = pathName;
        this.judgmentFactors = new ArrayList<>();
        this.isActive = true;
        this.executionCount = 0;
        this.pathConfiguration = new HashMap<>();
        this.executionHistory = new ArrayList<>();
    }
    
    @Override
    public final boolean canTransition(CustomerContext context) {
        if (!isActive) {
            return false;
        }
        
        // 記錄評估歷史
        String evaluationId = generateEvaluationId(context);
        
        try {
            boolean result = evaluateAllFactors(context);
            recordEvaluationResult(evaluationId, result, context);
            return result;
        } catch (Exception e) {
            recordEvaluationError(evaluationId, e, context);
            return false;
        }
    }
    
    @Override
    public final PromotionResult execute(CustomerContext context) {
        if (!canTransition(context)) {
            return PromotionResult.empty();
        }
        
        try {
            incrementExecutionCount();
            PromotionResult result = doExecute(context);
            recordSuccessfulExecution(context, result);
            return result;
        } catch (Exception e) {
            recordExecutionError(context, e);
            return PromotionResult.empty();
        }
    }
    
    @Override
    public void addJudgmentFactor(JudgmentFactor factor) {
        if (factor == null) {
            throw new IllegalArgumentException("JudgmentFactor cannot be null");
        }
        
        // 檢查是否已存在相同因子
        boolean exists = judgmentFactors.stream()
            .anyMatch(existing -> existing.getFactorName().equals(factor.getFactorName()));
        
        if (!exists) {
            judgmentFactors.add(factor);
            sortFactorsByPriority();
        }
    }
    
    @Override
    public void configure(RulePathConfiguration configuration) {
        if (configuration == null) {
            throw new IllegalArgumentException("RulePathConfiguration cannot be null");
        }
        
        this.pathName = configuration.getPathName();
        this.executionOrder = configuration.getExecutionOrder();
        this.pathConfiguration = new HashMap<>(configuration.getParameters());
        
        // 載入判斷因子
        loadJudgmentFactors(configuration.getJudgmentFactorConfigurations());
        
        validateConfiguration();
    }
    
    // 業務邏輯方法 - 充血模式
    public void activate() {
        this.isActive = true;
    }
    
    public void deactivate() {
        this.isActive = false;
    }
    
    public void updateExecutionOrder(int newOrder) {
        this.executionOrder = newOrder;
    }
    
    public void clearExecutionHistory() {
        this.executionHistory.clear();
        this.executionCount = 0;
    }
    
    public List<String> getRecentExecutionHistory(int limit) {
        int size = executionHistory.size();
        int fromIndex = Math.max(0, size - limit);
        return new ArrayList<>(executionHistory.subList(fromIndex, size));
    }
    
    public double getExecutionSuccessRate() {
        if (executionCount == 0) return 0.0;
        
        long successCount = executionHistory.stream()
            .mapToLong(history -> history.contains("SUCCESS") ? 1 : 0)
            .sum();
        
        return (double) successCount / executionCount;
    }
    
    public boolean isHealthy() {
        return isActive && getExecutionSuccessRate() >= 0.95; // 95% 成功率閾值
    }
    
    // 抽象方法供子類實作
    protected abstract PromotionResult doExecute(CustomerContext context);
    protected abstract void validateConfiguration();
    
    // 內部方法
    private boolean evaluateAllFactors(CustomerContext context) {
        return judgmentFactors.stream()
            .sorted(Comparator.comparing(JudgmentFactor::getPriority))
            .allMatch(factor -> factor.evaluate(context));
    }
    
    private void sortFactorsByPriority() {
        judgmentFactors.sort(Comparator.comparing(JudgmentFactor::getPriority));
    }
    
    private void incrementExecutionCount() {
        this.executionCount++;
        this.lastExecuted = LocalDateTime.now();
    }
    
    private String generateEvaluationId(CustomerContext context) {
        return String.format("%s_%s_%d", 
            pathId, context.getCustomerId(), System.currentTimeMillis());
    }
    
    private void recordEvaluationResult(String evaluationId, boolean result, CustomerContext context) {
        String record = String.format("[%s] EVALUATION %s - Customer: %s, Result: %s",
            LocalDateTime.now(), evaluationId, context.getCustomerId(), result ? "PASS" : "FAIL");
        executionHistory.add(record);
    }
    
    private void recordEvaluationError(String evaluationId, Exception e, CustomerContext context) {
        String record = String.format("[%s] EVALUATION_ERROR %s - Customer: %s, Error: %s",
            LocalDateTime.now(), evaluationId, context.getCustomerId(), e.getMessage());
        executionHistory.add(record);
    }
    
    private void recordSuccessfulExecution(CustomerContext context, PromotionResult result) {
        String record = String.format("[%s] EXECUTION_SUCCESS - Customer: %s, Promotion: %s",
            LocalDateTime.now(), context.getCustomerId(), result.getPromotionId());
        executionHistory.add(record);
    }
    
    private void recordExecutionError(CustomerContext context, Exception e) {
        String record = String.format("[%s] EXECUTION_ERROR - Customer: %s, Error: %s",
            LocalDateTime.now(), context.getCustomerId(), e.getMessage());
        executionHistory.add(record);
    }
    
    private void loadJudgmentFactors(List<JudgmentFactorConfiguration> factorConfigs) {
        judgmentFactors.clear();
        
        for (JudgmentFactorConfiguration config : factorConfigs) {
            try {
                JudgmentFactor factor = createJudgmentFactorByReflection(config);
                addJudgmentFactor(factor);
            } catch (Exception e) {
                System.err.println("Failed to load judgment factor: " + config.getFactorClassName());
            }
        }
    }
    
    private JudgmentFactor createJudgmentFactorByReflection(JudgmentFactorConfiguration config) 
            throws Exception {
        Class<?> factorClass = Class.forName(config.getFactorClassName());
        JudgmentFactor factor = (JudgmentFactor) factorClass.getDeclaredConstructor().newInstance();
        factor.configure(config.getParameters());
        return factor;
    }
    
    // Getters
    @Override
    public String getPathId() { return pathId; }
    
    @Override
    public String getPathName() { return pathName; }
    
    @Override
    public List<JudgmentFactor> getJudgmentFactors() { 
        return new ArrayList<>(judgmentFactors); 
    }
    
    @Override
    public RulePath getNextPath() { return nextPath; }
    
    public boolean isActive() { return isActive; }
    
    public int getExecutionOrder() { return executionOrder; }
    
    public LocalDateTime getLastExecuted() { return lastExecuted; }
    
    public long getExecutionCount() { return executionCount; }
}

// 具體規則路徑範例 - 純 Java 實作
public class PremiumCustomerRulePath extends AbstractRulePath {
    
    public PremiumCustomerRulePath() {
        super("PREMIUM_CUSTOMER_PATH", "高級客戶規則路徑");
    }
    
    @Override
    protected PromotionResult doExecute(CustomerContext context) {
        // 純業務邏輯，無框架依賴
        String promotionId = generatePromotionId(context);
        String promotionName = "高級客戶專屬優惠";
        
        Map<String, Object> promotionDetails = new HashMap<>();
        promotionDetails.put("discountRate", 0.15); // 15% 折扣
        promotionDetails.put("validUntil", LocalDateTime.now().plusDays(30));
        promotionDetails.put("customerTier", "PREMIUM");
        
        return new PromotionResult(promotionId, promotionName, 
            PromotionType.PREMIUM_EXCLUSIVE, promotionDetails, 
            List.of(getPathId()));
    }
    
    @Override
    protected void validateConfiguration() {
        // 驗證路徑特定的配置
        if (getJudgmentFactors().isEmpty()) {
            throw new IllegalStateException("Premium customer path must have at least one judgment factor");
        }
    }
    
    private String generatePromotionId(CustomerContext context) {
        return String.format("PREM_%s_%d", 
            context.getCustomerId(), System.currentTimeMillis());
    }
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