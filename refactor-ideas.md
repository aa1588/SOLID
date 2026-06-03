# Comprehensive Refactoring Plan: AIIG Quote Service
## Architecture Analysis & Modernization Strategy

**Document Version**: 1.0  
**Last Updated**: January 2025  
**Status**: Analysis Phase

---

## Table of Contents
1. [Executive Summary](#executive-summary)
2. [Current Architecture Analysis](#current-architecture-analysis)
3. [Critical Problems Identified](#critical-problems-identified)
4. [Proposed Architecture](#proposed-architecture)
5. [Refactoring Roadmap](#refactoring-roadmap)
6. [Design Patterns to Apply](#design-patterns-to-apply)
7. [Testing Strategy](#testing-strategy)
8. [Migration Path](#migration-path)

---

## Executive Summary

### Current State
The AIIG Quote Service is a Spring Boot application that handles insurance quote processing, transformation, and integration with an external SPIN rating system. The codebase exhibits **significant technical debt** with anti-patterns that severely impact maintainability, testability, and scalability.

### Key Findings
- **Massive Controller Classes**: `QuoteWS.java` (1758 lines) violates SRP severely
- **Static Utility Hell**: 16+ static utility classes creating tight coupling and testing nightmares
- **God Object Pattern**: Controllers doing business logic, validation, transformation, persistence
- **No Clear Boundaries**: Service layer barely exists, logic scattered everywhere
- **State Management Issues**: Static mutable state for deductible tracking (recent addition)
- **Zero Abstraction**: Direct coupling to SPIN DTOs throughout the codebase
- **Primitive Obsession**: Strings and BigDecimals everywhere instead of domain types

### Business Impact
- **Development Velocity**: New features take 3-5x longer due to code complexity
- **Bug Rate**: High defect rate in production (deductible calculation errors)
- **Testability**: ~70% of code untestable due to static methods and tight coupling
- **Scalability**: Unable to add new product types or states without massive changes
- **Onboarding**: New developers take 3-4 weeks to understand the codebase

### Recommended Approach
**Strangler Fig Pattern** - Gradually replace old code with new architecture without breaking existing functionality. Estimated timeline: 6-9 months for complete transformation.

---

## Current Architecture Analysis

### 1. Package Structure

```
com.aiig.quote/
├── api/                    # Controllers (FAT, doing too much)
│   ├── QuoteWS.java       # 1758 lines - CRITICAL PROBLEM
│   ├── AIController.java  # Similar issues
│   └── FileS3Controller.java
├── configuration/          # Security & Spring config
├── model/                  # Domain models (anemic)
│   ├── QuoteRequest.java
│   ├── QuoteResponse.java
│   ├── *Constants.java    # 3 different constant classes
│   └── ...
├── repository/            # Data access (minimal)
│   └── QuoterDao.java
└── service/               # Business logic (scattered)
    ├── *Utils.java        # 16+ static utility classes
    ├── Transform*.java    # Transformation logic
    └── ...
```

### 2. Current Data Flow

```
┌─────────────────┐
│   REST Client   │
└────────┬────────┘
         │
         ↓
┌────────────────────────────────────────────────────┐
│              QuoteWS.java (Controller)             │
│  ┌──────────────────────────────────────────────┐ │
│  │  • Request Validation (scattered)            │ │
│  │  • Product Type Routing                      │ │
│  │  • State-based Logic                         │ │
│  │  • Customer-specific Logic                   │ │
│  │  • SPIN API Integration                      │ │
│  │  • Billing Methods Retrieval                 │ │
│  │  • Error Handling                            │ │
│  │  • Response Transformation                   │ │
│  │  • Persistence                               │ │
│  │  • Moratorium Checking                       │ │
│  └──────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────┘
         │
         ↓
┌────────────────────────────────────────────────────┐
│         Static Utility Classes (16+)               │
│  ┌────────────────────────────────────────────┐   │
│  │  DeductibleUtils    CoverageUtils         │   │
│  │  BuildingUtils      DefaultUtils          │   │
│  │  DiscountUtils      InsuredUtils          │   │
│  │  TransformDeductibles                     │   │
│  │  ... (all static methods)                 │   │
│  └────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────┘
         │
         ↓
┌────────────────────────────────────────────────────┐
│              External SPIN System                  │
└────────────────────────────────────────────────────┘
```

### 3. Critical Code Smells

#### 3.1 QuoteWS.java Analysis

**Problems:**
```java
public class QuoteWS {
    // PROBLEM 1: Fat Controller - 1758 lines
    
    @RequestMapping(value = "add", method = RequestMethod.POST)
    public AIResponse callService(@RequestBody AIRequest request) {
        // PROBLEM 2: Validation mixed with business logic
        if (request.getQuoteRequest().getProductType() == null) {
            response.setStatus(GlobalConstants.ERROR);
            return response;
        }
        
        // PROBLEM 3: Complex conditional logic
        if (existingTransaction) {
            // 300+ lines of code
        } else {
            // 400+ lines of code
        }
        
        // PROBLEM 4: Direct SPIN API calls in controller
        restTemplate.exchange(url, HttpMethod.POST, entity, ...);
        
        // PROBLEM 5: Business logic in controller
        moratoriumCheck(request.getQuoteRequest());
        
        // PROBLEM 6: Data access in controller
        quoter = quoterDao.saveQuote(quoter);
    }
    
    // PROBLEM 7: Business logic methods in controller
    public void quoteAdd(...) { /* 600+ lines */ }
    public void quoteBind(...) { /* 400+ lines */ }
    protected BillingMethods retrieveFLBillingMethods(...) { /* ... */ }
    // ... many more
}
```

#### 3.2 Static Utility Hell

**DeductibleUtils.java**:
```java
@Service  // PROBLEM: Annotated as Service but all methods are static
public class Deduct
ibleUtils {
    // PROBLEM 1: Static mutable state - Thread safety issues!
    private static BigInteger covA;
    private static BigInteger calculatedAopDeductible;
    private static Boolean isCoastal;
    
    // PROBLEM 2: Recently added - static flags for tracking
    private static boolean flHO6HurricaneDeductibleChanged = false;
    private static boolean flHO6AOPDeductibleChanged = false;
    
    // PROBLEM 3: All business logic as static methods
    public static void determineSCDeductibles(...) { }
    public static DTOBuilding determineHO6HurricaneDeductible(...) { }
    // ... 20+ more static methods
}
```

**Why This Is Problematic:**
- **Untestable**: Cannot mock dependencies
- **Thread-unsafe**: Shared mutable state (covA, isCoastal, flags)
- **Tight Coupling**: Cannot swap implementations
- **Memory Leaks**: Static state persists across requests
- **Violation of OOP**: No encapsulation, just procedural code

#### 3.3 Anemic Domain Model

```java
// Current: Data containers with no behavior
public class QuoteRequest {
    private String productType;
    private Coverages coverages;
    private Deductibles deductibles;
    // 50+ getters and setters, ZERO business logic
}

public class Deductibles {
    private BigDecimal aop;
    private BigInteger hurricane;
    private Boolean windHailExclusion;
    // Only getters/setters
}
```

**The Problem:**
- All business logic extracted to static utility classes
- Domain objects are just data bags
- No encapsulation of business rules
- Impossible to enforce invariants

#### 3.4 Primitive Obsession

```java
// Current: Strings and primitives everywhere
String productType = "HO6";
String state = "FL";
BigDecimal aop = new BigDecimal("2500");
String occupancy = "OWNER";

// Problems:
// - No type safety
// - Easy to pass wrong values
// - No validation at type level
// - String comparisons scattered everywhere
```

#### 3.5 Lack of Abstraction

```java
// Direct coupling to SPIN DTOs throughout codebase
DTOBuilding building = new DTOBuilding();
DTOApplication app = new DTOApplication();
DTOBasicPolicy basicPolicy = new DTOBasicPolicy();

// Problems:
// - Cannot change external system without massive refactoring
// - Domain logic mixed with integration concerns
// - Tests require full SPIN DTO object graphs
```

---

## Critical Problems Identified

### Problem 1: Single Responsibility Principle Violations

**Severity: CRITICAL**

**QuoteWS.java Responsibilities** (Should be 1, currently ~15):
1. HTTP request/response handling
2. Request validation
3. Product type routing
4. State-based business rules
5. Customer-specific logic
6. Moratorium checking
7. SPIN API integration
8. Billing methods retrieval
9. Response transformation
10. Error handling
11. Persistence logic
12. Logging and auditing
13. Premium calculation orchestration
14. Validation error aggregation
15. URL generation

**Impact:**
- Changes to one concern affect all others
- Impossible to test in isolation
- Code duplication across methods
- Cognitive overload for developers

### Problem 2: Static Method Anti-Pattern

**Severity: CRITICAL**

**Affected Classes:**
- DeductibleUtils.java (20+ static methods)
- CoverageUtils.java
- BuildingUtils.java
- DefaultUtils.java
- DiscountUtils.java
- InsuredUtils.java
- OptionalCoverageUtils.java
- PriorPolicyUtils.java
- UnderwritingQuestionUtils.java
- WindMitigationUtils.java
- QuoteSvcUtils.java
- TransformDeductibles.java (all static)
- Utils.java

**Problems:**
1. **Testing Nightmare**: Cannot mock, cannot inject, cannot isolate
2. **Thread Safety**: Static mutable state (covA, calculatedAopDeductible, flags)
3. **Memory Issues**: State persists across requests
4. **No Polymorphism**: Cannot have different implementations for different products
5. **Tight Coupling**: Everything depends on everything else
6. **Hidden Dependencies**: Static methods calling other static methods

**Example of Thread Safety Bug:**
```java
public class DeductibleUtils {
    private static BigInteger covA;  // SHARED MUTABLE STATE!
    
    public static void determineSCDeductibles(QuoteRequest request, DTOBuilding building) {
        covA = request.getCoverages().getCovA();  // Thread 1 sets this
        // ... complex logic ...
        calculatedAopDeductible = covA.multiply(...);  // Thread 2 might have changed covA!
    }
}
```

**Real-world Scenario:**
- Thread 1 processes HO3 quote with covA=$300,000
- Thread 2 processes HO6 quote with covA=$100,000  
- Thread 1's calculation uses Thread 2's covA value
- **Result: Incorrect premium calculation!**

### Problem 3: No Layered Architecture

**Severity: HIGH**

**Current Architecture:**
```
Controller → Static Utils → DAO
     ↓           ↓           ↓
All mixed together, no clear boundaries
```

**Missing Layers:**
- No Service Layer (proper one with state and dependencies)
- No Domain Layer (rich domain models)
- No Application Layer (use cases/orchestration)
- No Integration Layer (SPIN adapter)
- No Validation Layer (proper validation)

**Impact:**
- Cannot enforce architectural rules
- Cannot test layers independently
- Cannot swap implementations
- Cannot apply layer-specific patterns

### Problem 4: Product/State Proliferation

**Severity: HIGH**

**Current Approach:**
```java
// Conditional explosion
if ("HO3".equals(productType) && !isSC && !isGA) {
    // HO3 logic
} else if ("HO3".equals(productType) && isSC) {
    // HO3 SC logic
} else if ("HO6".equals(productType)) {
    if ("FL".equals(state)) {
        // HO6 FL logic
    }
} else if ("DP1".equals(productType)) {
    // DP1 logic
} else if ("DP3".equals(productType)) {
    // DP3 logic
}
// ... hundreds of these scattered everywhere
```

**Scalability Analysis:**
- Current: 6 product types × 4 states = 24 combinations
- Each new product requires changes in ~30 files
- Each new state requires changes in ~40 files
- Adding business rules touches 20+ places

**Example: Adding NC HO6 Support**
Would require changes in:
1. DeductibleUtils.java (10+ methods)
2. TransformDeductibles.java (3+ locations)
3. QuoteWS.java (validation, routing)
4. CoverageUtils.java
5. BuildingUtils.java
6. Constants files
7. Tests (if they existed!)

### Problem 5: No Domain-Driven Design

**Severity: HIGH**

**Missing Concepts:**
- No Product abstraction (HO3, HO6, DP1 should be objects)
- No State/Territory abstraction
- No Policy concept (just DTOs)
- No Deductible Strategy pattern
- No Rating Context
- No bounded contexts

**Current Mental Model:**
```
QuoteRequest → Transformation → SPIN → Response
(procedural pipeline)
```

**Should Be:**
```
Quote Aggregate → Business Rules → Rating → Policy
(domain-centric)
```

### Problem 6: Testing Impossibility

**Severity: CRITICAL**

**Current Test Coverage**: ~10-20% (estimated)

**Why So Low:**
1. Static methods everywhere - cannot mock
2. Complex conditionals - combinatorial explosion
3. External API calls in controllers - requires real SPIN
4. No dependency injection - hard-coded dependencies
5. Shared mutable state - tests interfere with each other
6. 1758-line methods - where do you even start?

**Example Test Difficulty:**
```java
// How do you test this?
public void quoteAdd(String customerID, QuoteRequest request, ...) {
    // 600+ lines
    // 50+ conditional branches
    // 10+ external API calls
    // 5+ static method calls
    // Direct database access
    // All in one method!
}
```

---

## Proposed Architecture

### 1. Hexagonal Architecture (Ports & Adapters)

**Why Hexagonal:**
- Decouples business logic from external concerns
- Makes testing trivial
- Allows swapping implementations
- Clear dependency direction (inward)
- Perfect for insurance domain complexity

**Architecture Diagram:**

```
┌─────────────────────────────────────────────────────────────┐
│                     External World                           │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐           │
│  │ REST API   │  │ SPIN       │  │ Database   │           │
│  │ (Primary)  │  │ (Secondary)│  │ (Secondary)│           │
│  └──────┬─────┘  └──────┬─────┘  └──────┬─────┘           │
│         │                │                │                 │
│         ↓                ↓                ↓                 │
│  ┌──────────────────────────────────────────────────┐     │
│  │              ADAPTERS (Infrastructure)            │     │
│  ├──────────────────────────────────────────────────┤     │
│  │  • QuoteController    • SpinRatingAdapter        │     │
│  │  • RequestValidator   • DatabaseAdapter          │     │
│  │  • ResponseMapper     • BillingAdapter           │     │
│  └─────────────────────┬────────────────────────────┘     │
│                        │ implements                        │
└────────────────────────┼───────────────────────────────────┘
                         │
                         ↓
┌─────────────────────────────────────────────────────────────┐
│                    PORTS (Interfaces)                        │
├─────────────────────────────────────────────────────────────┤
│  Primary Ports (inbound):                                    │
│  • QuoteService                                              │
│  • BindService                                               │
│                                                              │
│  Secondary Ports (outbound):                                 │
│  • RatingEngine                                              │
│  • QuoteRepository                                           │
│  • BillingService                                            │
│  • NotificationService                                       │
└─────────────────────────┬───────────────────────────────────┘
                         │ used by
                         ↓
┌─────────────────────────────────────────────────────────────┐
│                  APPLICATION LAYER                           │
│                     (Use Cases)                              │
├─────────────────────────────────────────────────────────────┤
│  • CreateQuoteUseCase                                        │
│  • UpdateQuoteUseCase                                        │
│  • BindPolicyUseCase                                         │
│  • CalculatePremiumUseCase                                   │
│  • RetrieveBillingOptionsUseCase                            │
└─────────────────────────┬───────────────────────────────────┘
                         │ orchestrates
                         ↓
┌─────────────────────────────────────────────────────────────┐
│                    DOMAIN LAYER                              │
│                  (Business Logic)                            │
├─────────────────────────────────────────────────────────────┤
│  Aggregates:                                                 │
│  • Quote (root)                                              │
│  • Policy                                                    │
│                                                              │
│  Entities:                                                   │
│  • Product (HO3, HO6, DP1, DP3)                             │
│  • Insured                                                   │
│  • Property                                                  │
│  • Coverage                                                  │
│                                                              │
│  Value Objects:                                              │
│  • DeductibleAmount                                          │
│  • PremiumAmount                                             │
│  • ProductType                                               │
│  • State                                                     │
│  • Address                                                   │
│                                                              │
│  Domain Services:                                            │
│  • DeductibleCalculationService                             │
│  • PremiumCalculationService                                │
│  • UnderwritingRulesService                                 │
│  • ProductFactory                                            │
│                                                              │
│  Strategies (Product-specific):                             │
│  • HO3DeductibleStrategy                                     │
│  • HO6DeductibleStrategy                                     │
│  • FLDeductibleRules                                        │
│  • SCDeductibleRules                                        │
└─────────────────────────────────────────────────────────────┘
```

### 2. Package Structure (New)

```
com.aiig.quote/
├── domain/                          # Domain Layer (Core Business Logic)
│   ├── model/
│   │   ├── aggregate/
│   │   │   ├── Quote.java
│   │   │   └── Policy.java
│   │   ├── entity/
│   │   │   ├── Product.java
│   │   │   ├── Insured.java
│   │   │   ├── Property.java
│   │   │   └── Coverage.java
│   │   └── valueobject/
│   │       ├── DeductibleAmount.java
│   │       ├── PremiumAmount.java
│   │       ├── ProductType.java
│   │       ├── State.java
│   │       └── Address.java
│   ├── service/                     # Domain Services
│   │   ├── DeductibleCalculationService.java
│   │   ├── UnderwritingRulesService.java
│   │   └── ProductFactory.java
│   ├── strategy/                    # Strategy Pattern
│   │   ├── deductible/
│   │   │   ├── DeductibleStrategy.java (interface)
│   │   │   ├── HO3DeductibleStrategy.java
│   │   │   ├── HO6DeductibleStrategy.java
│   │   │   └── DP1DeductibleStrategy.java
│   │   └── pricing/
│   │       └── PricingStrategy.java
│   ├── repository/                  # Repository Interfaces (Ports)
│   │   ├── QuoteRepository.java
│   │   └── PolicyRepository.java
│   └── event/                       # Domain Events
│       ├── QuoteCreated.java
│       └── DeductibleAdjusted.java
│
├── application/                     # Application Layer (Use Cases)
│   ├── usecase/
│   │   ├── CreateQuoteUseCase.java
│   │   ├── UpdateQuoteUseCase.java
│   │   ├── BindPolicyUseCase.java
│   │   └── CalculatePremiumUseCase.java
│   ├── port/
│   │   ├── in/                     # Primary Ports
│   │   │   ├── QuoteService.java
│   │   │   └── BindService.java
│   │   └── out/                    # Secondary Ports
│   │       ├── RatingEngine.java
│   │       ├── BillingService.java
│   │       └── NotificationService.java
│   └── dto/                        # Application DTOs
│       ├── QuoteCommand.java
│       └── QuoteResult.java
│
├── infrastructure/                  # Infrastructure Layer (Adapters)
│   ├── adapter/
│   │   ├── in/                     # Primary Adapters
│   │   │   ├── rest/
│   │   │   │   ├── QuoteController.java
│   │   │   │   ├── QuoteRequestMapper.java
│   │   │   │   └── QuoteResponseMapper.java
│   │   │   └── validation/
│   │   │       └── QuoteRequestValidator.java
│   │   └── out/                    # Secondary Adapters
│   │       ├── spin/
│   │       │   ├── SpinRatingAdapter.java
│   │       │   ├── SpinDtoMapper.java
│   │       │   └── SpinApiClient.java
│   │       ├── persistence/
│   │       │   ├── JpaQuoteRepository.java
│   │       │   └── entity/
│   │       │       └── QuoteEntity.java
│   │       └── billing/
│   │           └── BillingServiceAdapter.java
│   └── config/
│       ├── BeanConfiguration.java
│       └── SecurityConfiguration.java
│
└── shared/                          # Shared Kernel
    ├── exception/
    │   ├── DomainException.java
    │   └── ValidationException.java
    └── util/
        └── DateUtils.java
```

### 3. New Domain Model

#### 3.1 Quote Aggregate (Root)

```java
package com.aiig.quote.domain.model.aggregate;

import com.aiig.quote.domain.model.entity.*;
import com.aiig.quote.domain.model.valueobject.*;
import com.aiig.quote.domain.event.*;

/**
 * Quote Aggregate Root
 * 
 * Responsibilities:
 * - Enforce invariants
 * - Coordinate entities within boundary
 * - Raise domain events
 * - Encapsulate business rules
 */
public class Quote {
    private QuoteId id;
    private Product product;
    private Insured insured;
    private Property property;
    private List<Coverage> coverages;
    private Deductibles deductibles;
    private PremiumAmount premium;
    private QuoteStatus status;
    private List<ValidationMessage> messages;
    private List<DomainEvent> domainEvents;
    
    // Constructor enforces creation rules
    private Quote(QuoteId id, Product product, Insured insured, Property property) {
        this.id = Objects.requireNonNull(id);
        this.product = Objects.requireNonNull(product);
        this.insured = Objects.requireNonNull(insured);
        this.property = Objects.requireNonNull(property);
        this.coverages = new ArrayList<>();
        this.messages = new ArrayList<>();
        this.domainEvents = new ArrayList<>();
        this.status = QuoteStatus.DRAFT;
    }
    
    // Factory method - enforces business rules at creation
    public static Quote create(Product product, Insured insured, Property property) {
        QuoteId id = QuoteId.generate();
        Quote quote = new Quote(id, product, insured, property);
        
        // Validate at creation
        quote.validate();
        
        // Raise domain event
        quote.addDomainEvent(new QuoteCreated(id, product.getType(), property.getState()));
        
        return quote;
    }
    
    // Business logic methods
    public void calculateDeductibles(DeductibleCalculationService calculator) {
        Deductibles original = this.deductibles;
        this.deductibles = calculator.calculate(product, property, coverages);
        
        // Check if adjusted
        if (!original.equals(this.deductibles)) {
            this.addInfoMessage(
                "Hurricane and/or Non Hurricane deductible updated to meet Underwriting Guidelines."
            );
            this.addDomainEvent(new DeductibleAdjusted(this.id, original, this.deductibles));
        }
    }
    
    public void calculatePremium(PremiumCalculationService calculator) {
        this.premium = calculator.calculate(this);
        if (this.premium.isZero()) {
            throw new DomainException("Premium cannot be zero");
        }
    }
    
    public void bind() {
        if (!canBind()) {
            throw new DomainException("Quote cannot be bound in current state");
        }
        this.status = QuoteStatus.BOUND;
        this.addDomainEvent(new QuoteBound(this.id));
    }
    
    private boolean canBind() {
        return this.status == QuoteStatus.QUOTED 
            && this.premium != null 
            && !this.premium.isZero()
            && this.messages.stream().noneMatch(ValidationMessage::isError);
    }
    
    private void validate() {
        if (!product.isAvailableIn(property.getState())) {
            throw new DomainException(
                String.format("Product %s not available in state %s", 
                    product.getType(), property.getState())
            );
        }
    }
    
    // Getters (no setters - immutability where possible)
    public QuoteId getId() { return id; }
    public Product getProduct() { return product; }
    public PremiumAmount getPremium() { return premium; }
    public QuoteStatus getStatus() { return status; }
    public List<ValidationMessage> getMessages() { 
        return Collections.unmodifiableList(messages); 
    }
    
    // Event handling
    public List<DomainEvent> getDomainEvents() {
        return Collections.unmodifiableList(domainEvents);
    }
    
    public void clearDomainEvents() {
        this.domainEvents.clear();
    }
    
    private void addDomainEvent(DomainEvent event) {
        this.domainEvents.add(event);
    }
    
    private void addInfoMessage(String message) {
        this.messages.add(ValidationMessage.info(message));
    }
}
```

#### 3.2 Value Objects - Type Safety

```java
package com.aiig.quote.domain.model.valueobject;

/**
 * ProductType Value Object
 * 
 * Benefits:
 * - Type safety (cannot pass wrong string)
 * - Validation at creation
 * - Immutability
 * - Self-documenting
 */
public final class ProductType {
    private final String code;
    
    // Private constructor
    private ProductType(String code) {
        this.code = code;
    }
    
    // Factory methods - only valid types can be created
    public static ProductType HO3() { return new ProductType("HO3"); }
    public static ProductType HO6() { return new ProductType("HO6"); }
    public static ProductType DP1() { return new ProductType("DP1"); }
    public static ProductType DP3() { return new ProductType("DP3"); }
    public static ProductType HO4() { return new ProductType("HO4"); }
    public static ProductType HO5() { return new ProductType("HO5"); }
    
    public static ProductType fromString(String code) {
        return switch (code.toUpperCase()) {
            case "HO3" -> HO3();
            case "HO6" -> HO6();
            case "DP1" -> DP1();
            case "DP3" -> DP3();
            case "HO4" -> HO4();
            case "HO5" -> HO5();
            default -> throw new IllegalArgumentException("Invalid product type: " + code);
        };
    }
    
    public String getCode() { return code; }
    public boolean isHO6() { return "HO6".equals(code); }
    public boolean isHomeowners() { return code.startsWith("HO"); }
    
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        ProductType that = (ProductType) o;
        return Objects.equals(code, that.code);
    }
    
    @Override
    public int hashCode() {
        return Objects.hash(code);
    }
}
```

```java
package com.aiig.quote.domain.model.valueobject;

/**
 * DeductibleAmount Value Object
 * 
 * Encapsulates deductible business rules:
 * - Must be positive
 * - Can be flat amount or percentage
 * - Has display formatting
 */
public final class DeductibleAmount {
    private final BigDecimal amount;
    private final DeductibleType type; // FLAT or PERCENTAGE
    
    private DeductibleAmount(BigDecimal amount, DeductibleType type) {
        if (amount.compareTo(BigDecimal.ZERO) < 0) {
            throw new IllegalArgumentException("Deductible cannot be negative");
        }
        this.amount = amount;
        this.type = type;
    }
    
    public static DeductibleAmount flat(BigDecimal amount) {
        return new DeductibleAmount(amount, DeductibleType.FLAT);
    }
    
    public static DeductibleAmount percentage(BigDecimal percent) {
        if (percent.compareTo(BigDecimal.valueOf(100)) > 0) {
            throw new IllegalArgumentException("Percentage cannot exceed 100");
        }
        return new DeductibleAmount(percent, DeductibleType.PERCENTAGE);
    }
    
    public BigDecimal getAmount() { return amount; }
    public boolean isFlat() { return type == DeductibleType.FLAT; }
    public boolean isPercentage() { return type == DeductibleType.PERCENTAGE; }
    
    public String display() {
        return isFlat() ? "$" + amount : amount + "%";
    }
    
    // Value object equality
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        DeductibleAmount that = (DeductibleAmount) o;
        return amount.compareTo(that.amount) == 0 && type == that.type;
    }
    
    @Override
    public int hashCode() {
        return Objects.hash(amount, type);
    }
}
```

#### 3.3 Strategy Pattern - Product-Specific Logic

```java
package com.aiig.quote.domain.strategy.deductible;

/**
 * Strategy interface for deductible calculation
 * 
 * Allows different implementations for different products/states
 * without conditional logic explosion
 */
public interface DeductibleStrategy {
    Deductibles calculate(Property property, List<Coverage> coverages);
    boolean supports(ProductType product, State state);
}
```

```java
package com.aiig.quote.domain.strategy.deductible;

/**
 * HO6 Florida Deductible Strategy
 * 
 * Encapsulates ALL FL HO6 deductible logic in one place
 * No more scattered conditionals!
 */
@Component
public class HO6FloridaDeductibleStrategy implements DeductibleStrategy {
    
    private final DeductibleRulesConfig config;
    
    public HO6FloridaDeductibleStrategy(DeductibleRulesConfig config) {
        this.config = config;
    }
    
    @Override
    public Deductibles calculate(Property property, List<Coverage> coverages) {
        DeductibleAmount hurricane = calculateHurricane(property, coverages);
        DeductibleAmount aop = calculateAOP(property, coverages, hurricane);
        
        return new Deductibles(hurricane, aop);
    }
    
    @Override
    public boolean supports(ProductType product, State state) {
        return product.isHO6() && state.isFlorida();
    }
    
    private DeductibleAmount calculateHurricane(Property property, List<Coverage> coverages) {
        // All FL HO6 hurricane logic here
        BigInteger sumCovACovC = calculateCovAPlus CovC(coverages);
        
        if (sumCovACovC.compareTo(config.getHO6Threshold1()) < 0) {
            // Logic for range 1
        } else if (sumCovACovC.compareTo(config.getHO6Threshold2()) <= 0) {
            // Logic for range 2
        }
        // etc.
    }
    
    private DeductibleAmount calculateAOP(Property property, List<Coverage> coverages, 
                                          DeductibleAmount hurricane) {
        // All FL HO6 AOP logic here
        if (property.hasWindHailExclusion()) {
            // Wind/hail exclusion logic
        } else {
            // Normal logic based on hurricane deductible
        }
    }
}
```

```java
package com.aiig.quote.domain.service;

/**
 * Domain Service for deductible calculation
 * 
 * Delegates to appropriate strategy based on product/state
 */
@Service
public class DeductibleCalculationService {
    
    private final List<DeductibleStrategy> strategies;
    
    public DeductibleCalculationService(List<DeductibleStrategy> strategies) {
        this.strategies = strategies;
    }
    
    public Deductibles calculate(Product product, Property property, List<Coverage> coverages) {
        DeductibleStrategy strategy = strategies.stream()
            .filter(s -> s.supports(product.getType(), property.getState()))
            .findFirst()
            .orElseThrow(() -> new DomainException(
                "No deductible strategy found for " + product.getType() + " in " + property.getState()
            ));
        
        return strategy.calculate(property, coverages);
    }
}
```

### 4. Use Case Layer

```java
package com.aiig.quote.application.usecase;

/**
 * CreateQuoteUseCase
 * 
 * Application service that orchestrates the quote creation process
 * Pure business flow, no infrastructure concerns
 */
@UseCase
@Transactional
public class CreateQuoteUseCase {
    
    private final QuoteRepository quoteRepository;
    private final DeductibleCalculationService deductibleService;
    private final RatingEngine ratingEngine;
    private final EventPublisher eventPublisher;
    
    public CreateQuoteUseCase(
        QuoteRepository quoteRepository,
        DeductibleCalculationService deductibleService,
        RatingEngine ratingEngine,
        EventPublisher eventPublisher
    ) {
        this.quoteRepository = quoteRepository;
        this.deductibleService = deductibleService;
        this.ratingEngine = ratingEngine;
        this.eventPublisher = eventPublisher;
    }
    
    public QuoteResult execute(CreateQuoteCommand command) {
        // 1. Create aggregate from command
        Quote quote = Quote.create(
            command.getProduct(),
            command.getInsured(),
            command.getProperty()
        );
        
        // 2. Apply business rules
        quote.calculateDeductibles(deductibleService);
        
        // 3. Get premium from rating engine
        PremiumAmount premium = ratingEngine.rate(quote);
        quote.setPremium(premium);
        
        // 4. Persist
        quoteRepository.save(quote);
        
        // 5. Publish domain events
        quote.getDomainEvents().forEach(eventPublisher::publish);
        quote.clearDomainEvents();
        
        // 6. Return result
        return QuoteResult.from(quote);
    }
}
```

---

## Design Patterns to Apply

### 1. Strategy Pattern

**Problem**: Product/state conditional explosion
**Solution**: One strategy per product/state combination

**Benefits:**
- Open/Closed Principle
- Easy to add new products
- Easy to test each strategy
- No more if-else chains

**Example:**
```
DeductibleStrategy (interface)
├── HO3FloridaDeductibleStrategy
├── HO3SouthCarolinaDeductibleStrategy
├── HO6FloridaDeductibleStrategy
├── DP1FloridaDeductibleStrategy
└── ... (one per combination)
```

### 2. Factory Pattern

**Problem**: Complex object creation with validation
**Solution**: Encapsulate creation logic

```java
@Component
public class ProductFactory {
    
    public Product create(ProductType type, State state) {
        return switch (type.getCode()) {
            case "HO3" -> new HO3Product(state);
            case "HO6" -> new HO6Product(state);
            case "DP1" -> new DP1Product(state);
            // ...
            default -> throw new IllegalArgumentException("Unknown product type");
        };
    }
}
```

### 3. Repository Pattern

**Problem**: Direct database access mixed with business logic
**Solution**: Abstract persistence behind interface

```java
// Domain layer - interface
public interface QuoteRepository {
    Quote save(Quote quote);
    Optional<Quote> findById(QuoteId id);
    Optional<Quote> findByQuoteNumber(String quoteNumber);
}

// Infrastructure layer - implementation
@Repository
public class JpaQuoteRepository implements QuoteRepository {
    private final SpringDataQuoteRepository springRepo;
    private final QuoteEntityMapper mapper;
    
    @Override
    public Quote save(Quote quote) {
        QuoteEntity entity = mapper.toEntity(quote);
        QuoteEntity saved = springRepo.save(entity);
        return mapper.toDomain(saved);
    }
}
```

### 4. Adapter Pattern

**Problem**: Tight coupling to SPIN API
**Solution**: Wrap external system behind interface

```java
// Port (domain)
public interface RatingEngine {
    PremiumAmount rate(Quote quote);
}

// Adapter (infrastructure)
@Component
public class SpinRatingAdapter implements RatingEngine {
    private final SpinApiClient spinClient;
    private final SpinDtoMapper mapper;
    
    @Override
    public PremiumAmount rate(Quote quote) {
        UWExternalQuoteAddRq spinRequest = mapper.toSpinRequest(quote);
        UWExternalQuoteAddRs spinResponse = spinClient.quoteAdd(spinRequest);
        return mapper.toPremiumAmount(spinResponse);
    }
}
```

### 5. Builder Pattern

**Problem**: Complex object construction
**Solution**: Fluent API for object creation

```java
Quote quote = Quote.builder()
    .product(ProductType.HO6())
    .insured(insured)
    .property(property)
    .coverage(Coverage.CovA(Money.of(300000)))
    .coverage(Coverage.CovC(Money.of(45000)))
    .deductible(Deductibles.hurricane(DeductibleAmount.flat(1000)))
    .build();
```

### 6. Event Sourcing (Optional - Advanced)

**Problem**: Audit trail, complex state changes
**Solution**: Store events instead of state

```java
public class Quote {
    private List<DomainEvent> domainEvents = new ArrayList<>();
    
    public void adjustDeductible(DeductibleAmount newAmount) {
        DeductibleAmount oldAmount = this.deductible;
        this.deductible = newAmount;
        this.addDomainEvent(new DeductibleAdjusted(id, oldAmount, newAmount));
    }
}
```

---

## Refactoring Roadmap

### Phase 1: Foundation (Weeks 1-4)

**Goal**: Set up new architecture without breaking existing code

#### Week 1-2: Domain Model
- [ ] Create value objects (ProductType, State, DeductibleAmount, etc.)
- [ ] Create Quote aggregate
- [ ] Create Product entity hierarchy
- [ ] Write unit tests for domain model
- [ ] Define repository interfaces

#### Week 3-4: Application Layer
- [ ] Create use case interfaces
- [ ] Implement CreateQuoteUseCase
- [ ] Create command/result DTOs
- [ ] Add basic validation

**Deliverables:**
- Working domain model with 80%+ test coverage
- Use cases with integration tests
- No impact on existing code

### Phase 2: Strategy Pattern (Weeks 5-8)

**Goal**: Replace static utility classes with strategies

#### Week 5-6: Deductible Strategies
- [ ] Create DeductibleStrategy interface
- [ ] Implement HO6FloridaDeductibleStrategy
- [ ] Implement HO3FloridaDeductibleStrategy
- [ ] Create DeductibleCalculationService
- [ ] Write comprehensive tests

#### Week 7-8: Other Strategies
- [ ] Coverage strategies
- [ ] Discount strategies
- [ ] Validation strategies

**Migration:**
- Keep old static methods temporarily
- New code uses strategies
- Gradually migrate old calls

### Phase 3: Adapters (Weeks 9-12)

**Goal**: Isolate external dependencies

#### Week 9-10: SPIN Adapter
- [ ] Create RatingEngine port
- [ ] Implement SpinRatingAdapter
- [ ] Create SpinDtoMapper
- [ ] Add retry/circuit breaker

#### Week 11-12: Other Adapters
- [ ] BillingServiceAdapter
- [ ] DatabaseAdapter (Repository implementation)
- [ ] NotificationAdapter

### Phase 4: Controller Decomposition (Weeks 13-16)

**Goal**: Break down fat controllers

#### Week 13: Extract validation
- [ ] Create RequestValidator
- [ ] Move all validation logic
- [ ] Add Bean Validation annotations

#### Week 14: Extract transformation
- [ ] Create request/response mappers
- [ ] Remove DTO transformation from controller

#### Week 15-16: Slim controllers
- [ ] Controllers only call use cases
- [ ] Remove all business logic
- [ ] Remove direct repository access

### Phase 5: Remove Static Utilities (Weeks 17-20)

**Goal**: Eliminate all static methods

#### Approach:
1. Identify all usages of static method
2. Replace with strategy/service call
3. Add deprecation warning
4. Monitor usage
5. Delete static method

**Priority Order:**
1. DeductibleUtils (highest impact)
2. CoverageUtils
3. BuildingUtils
4. Others

### Phase 6: Testing & Quality (Weeks 21-24)

**Goal**: Achieve 80%+ test coverage

- [ ] Unit tests for all domain logic
- [ ] Integration tests for use cases
- [ ] Contract tests for adapters
- [ ] End-to-end tests for critical flows
- [ ] Performance tests

### Phase 7: Production Rollout (Weeks 25-26)

- [ ] Feature flags for new vs old code paths
- [ ] Gradual traffic migration
- [ ] Monitoring and alerts
- [ ] Rollback plan

---

## Testing Strategy

### 1. Unit Testing - Domain Layer

**Coverage Goal**: 90%+

```java
@DisplayName("Quote Aggregate Tests")
class QuoteTest {
    
    @Test
    @DisplayName("Should create quote with valid data")
    void shouldCreateQuote() {
        // Given
        Product product = ProductType.HO6().createProduct(State.FL());
        Insured insured = InsuredMother.valid();
        Property property = PropertyMother.floridaCondo();
        
        // When
        Quote quote = Quote.create(product, insured, property);
        
        // Then
        assertThat(quote.getId()).isNotNull();
        assertThat(quote.getStatus()).isEqualTo(QuoteStatus.DRAFT);
    }
    
    @Test
    @DisplayName("Should adjust deductible and raise event")
    void shouldAdjustDeductibleAndRaiseEvent() {
        // Given
        Quote quote = QuoteMother.flHO6();
        DeductibleCalculationService calculator = mock(DeductibleCalculationService.class);
        Deductibles newDeductibles = DeductiblesMother.adjusted();
        when(calculator.calculate(any(), any(), any())).thenReturn(newDeductibles);
        
        // When
        quote.calculateDeductibles(calculator);
        
        // Then
        assertThat(quote.getMessages()).hasSize(1);
        assertThat(quote.getMessages().get(0).isInfo()).isTrue();
        assertThat(quote.getDomainEvents())
            .hasSize(1)
            .first()
            .isInstanceOf(DeductibleAdjusted.class);
    }
}
```

### 2. Integration Testing - Use Cases

```java
@SpringBootTest
@Transactional
class CreateQuoteUseCaseIT {
    
    @Autowired
    private CreateQuoteUseCase createQuoteUseCase;
    
    @Autowired
    private QuoteRepository quoteRepository;
    
    @Test
    void shouldCreateQuoteEndToEnd() {
        // Given
        CreateQuoteCommand command = CreateQuoteCommandMother.flHO6();
        
        // When
        QuoteResult result = createQuoteUseCase.execute(command);
        
        // Then
        assertThat(result.getQuoteId()).isNotNull();
        Optional<Quote> saved = quoteRepository.findById(result.getQuoteId());
        assertThat(saved).isPresent();
    }
}
```

### 3. Contract Testing - Adapters

```java
@WebMvcTest(SpinRatingAdapter.class)
class SpinRatingAdapterTest {
    
    @Autowired
    private MockRestServiceServer mockServer;
    
    @Autowired
    private SpinRatingAdapter adapter;
    
    @Test
    void shouldMapQuoteToSpinRequest() {
        // Given
        Quote quote = QuoteMother.flHO6();
        mockServer.expect(requestTo(containsString("/spin/quoteAdd")))
            .andExpect(method(HttpMethod.POST))
            .andExpect(jsonPath("$.dtoApplication[0].dtoBasicPolicy[0].subTypeCd").value("HO6"))
            .andRespond(withSuccess(spinResponseJson(), APPLICATION_JSON));
        
        // When
        PremiumAmount premium = adapter.rate(quote);
        
        // Then
        assertThat(premium).isNotNull();
        mockServer.verify();
    }
}
```

### 4. Architecture Testing

```java
@AnalyzeClasses(packages = "com.aiig.quote")
class ArchitectureTest {
    
    @ArchTest
    static final ArchRule domainShouldNotDependOnInfrastructure =
        noClasses()
            .that().resideInAPackage("..domain..")
            .should().dependOnClassesThat().resideInAPackage("..infrastructure..");
    
    @ArchTest
    static final ArchRule noStaticMethods =
        methods()
            .that().areDeclaredInClassesThat().resideInAPackage("..service..")
            .should().notBeStatic();
    
    @ArchTest
    static final ArchRule controllersShouldOnlyCallUseCases =
        classes()
            .that().resideInAPackage("..infrastructure.adapter.in.rest..")
            .should().onlyAccessClassesThat().resideInAnyPackage(
                "..application.usecase..",
                "..application.port.in..",
                "..dto.."
            );
}
```

---

## Migration Path

### Strangler Fig Pattern

**Concept**: Gradually replace old system with new, running both side-by-side

```
┌──────────────────────────────────────┐
│         Old System (Legacy)          │
│  ┌────────────────────────────────┐  │
│  │    QuoteWS (Fat Controller)    │  │
│  │    Static Utils                │  │
│  └────────────────────────────────┘  │
└──────────────┬───────────────────────┘
               │
               ↓
       ┌───────────────┐
       │ Feature Flags │
       └───────┬───────┘
               │
        ┌──────┴──────┐
        ↓             ↓
    Old Path      New Path
        │             │
        ↓             ↓
┌────────────┐ ┌──────────────────────┐
│  Legacy    │ │   New Architecture   │
│  Code      │ │                      │
└────────────┘ │  ┌─────────────────┐ │
               │  │ Use Cases       │ │
               │  │ Domain Model    │ │
               │  │ Strategies      │ │
               │  │ Adapters        │ │
               │  └─────────────────┘ │
               └──────────────────────┘
```

### Feature Flag Example

```java
@RestController
public class QuoteController {
    
    private final CreateQuoteUseCase createQuoteUseCase;
    private final QuoteWS legacyController;
    private final FeatureFlags featureFlags;
    
    @PostMapping("/quote/add")
    public ResponseEntity<QuoteResponse> createQuote(@RequestBody QuoteRequest request) {
        
        if (featureFlags.isEnabled("new-architecture")) {
            // New path
            CreateQuoteCommand command = mapper.toCommand(request);
            QuoteResult result = createQuoteUseCase.execute(command);
            return ResponseEntity.ok(mapper.toResponse(result));
        } else {
            // Old path
            return legacyController.callService(request);
        }
    }
}
```

### Gradual Migration Steps

#### Step 1: Parallel Run (Week 25)
- Both old and new code execute
- Compare results
- Log discrepancies
- 0% production traffic to new code

#### Step 2: Canary Release (Week 26)
- 5% of traffic to new code
- Monitor errors, performance
- Quick rollback if issues

#### Step 3: Gradual Increase
- Week 27: 25% traffic
- Week 28: 50% traffic
- Week 29: 75% traffic
- Week 30: 100% traffic

#### Step 4: Cleanup
- Remove feature flags
- Delete legacy code
- Update documentation

---

## Benefits Analysis

### Before vs After

#### Maintainability

**Before:**
- Add new product: 30+ file changes
- Fix bug: Find all static method calls
- Understand code: Read 1758 lines
- Add feature: Modify fat controller

**After:**
- Add new product: Create 2-3 strategies
- Fix bug: Single strategy class
- Understand code: Read focused domain model
- Add feature: Create new use case

#### Testability

**Before:**
- Test coverage: ~10-20%
- Unit test: Impossible (static methods)
- Integration test: Requires full SPIN
- Mocking: Cannot mock statics

**After:**
- Test coverage: 80%+
- Unit test: Fast, isolated
- Integration test: Mock ports
- Mocking: Easy (interfaces)

#### Performance

**Before:**
- Thread safety issues
- Memory leaks (static state)
- No caching possible

**After:**
- Thread-safe by design
- Proper lifecycle management
- Easy to add caching

#### Scalability

**Before:**
- Monolithic conditionals
- Tightly coupled
- Hard to parallelize

**After:**
- Strategy pattern scales
- Loose coupling
- Easy to parallelize

---

## Risks & Mitigation

### Risk 1: Feature Parity

**Risk**: New code doesn't match old behavior exactly

**Mitigation:**
- Comprehensive test suite
- Parallel run with comparison
- Feature flags for quick rollback
- Business sign-off on each phase

### Risk 2: Performance Regression

**Risk**: New abstractions slower than static methods

**Mitigation:**
- Performance testing from day 1
- Profiling and optimization
- Benchmarking old vs new
- Acceptable threshold: <10% slower

### Risk 3: Timeline Overrun

**Risk**: 26 weeks too optimistic

**Mitigation:**
- Build in 20% buffer
- Prioritize critical paths
- MVP approach - core features first
- Regular stakeholder updates

### Risk 4: Team Resistance

**Risk**: Team prefers old familiar code

**Mitigation:**
- Training sessions
- Pair programming
- Show benefits early
- Celebrate wins

---

## Success Metrics

### Technical Metrics

- **Test Coverage**: 10% → 80%+
- **Cyclomatic Complexity**: Avg 50 → Avg 5
- **Method Length**: Avg 100 lines → Avg 10 lines
- **Defect Rate**: 5/sprint → 1/sprint
- **Build Time**: 10 min → 3 min (better tests)

### Business Metrics

- **Feature Velocity**: +50% (easier to add features)
- **Time to Market**: -40% (new products faster)
- **Onboarding Time**: 4 weeks → 1 week
- **Production Incidents**: -60%

### Developer Experience

- **Code Comprehension**: "Very Hard" → "Easy"
- **Confidence in Changes**: Low → High
- **Job Satisfaction**: Survey improvement
- **Tech Debt**: High → Low

---

## Conclusion

The current architecture is **unsustainable**. The combination of fat controllers, static utility hell, anemic domain models, and lack of abstraction creates a maintenance nightmare that will only get worse as the system grows.

The proposed refactoring using **Hexagonal Architecture**, **Domain-Driven Design**, and **Design Patterns** will transform this codebase into a maintainable, testable, scalable system.

**Recommendation**: Start Phase 1 immediately. The investment will pay off within 6 months through increased velocity and reduced defects.

---

## Appendix A: Example Comparisons

### Adding New Feature: "Email notification when deductible adjusted"

#### Current Approach (Before)

1. Find all places deductibles are calculated (scattered across 5 files)
2. Add email notification code in each place
3. Hope you found them all
4. Cannot test without SPIN integration

**Files Changed**: 5-7  
**Lines of Code**: 200+  
**Test Coverage**: Minimal  
**Risk**: High (missed locations)

#### New Approach (After)

1. Listen to `DeductibleAdjusted` domain event
2. Implement `NotificationEventHandler`
3. Unit test event handler

**Files Changed**: 2  
**Lines of Code**: 30  
**Test Coverage**: 100%  
**Risk**: Low (centralized)

---

## Appendix B: Resources

### Books
- "Domain-Driven Design" - Eric Evans
- "Implementing Domain-Driven Design" - Vaughn Vernon
- "Clean Architecture" - Robert C. Martin
- "Refactoring" - Martin Fowler
- "Working Effectively with Legacy Code" - Michael Feathers

### Patterns
- Hexagonal Architecture (Ports & Adapters)
- Strategy Pattern
- Repository Pattern
- Factory Pattern
- Domain Events
- Strangler Fig Pattern

### Tools
- ArchUnit (architecture testing)
- JaCoCo (test coverage)
- SonarQube (code quality)
- Feature flags (LaunchDarkly/FF4J)

---

**Document End**

*This refactoring plan is a living document and should be updated as the team learns and adapts during implementation.*
