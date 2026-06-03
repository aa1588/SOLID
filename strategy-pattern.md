# Strategy Pattern for Deductible Calculation
## Solving the Product × State Combinatorial Explosion Problem

---

## Table of Contents
1. [The Problem: Current Architecture](#the-problem-current-architecture)
2. [The Concrete Challenge](#the-concrete-challenge)
3. [Strategy Pattern Solution](#strategy-pattern-solution)
4. [ProductType & State Modeling](#producttype--state-modeling)
5. [Complete Implementation](#complete-implementation)
6. [Benefits Analysis](#benefits-analysis)
7. [Migration Strategy](#migration-strategy)

---

## The Problem: Current Architecture

### Current Code Structure

In your existing codebase, deductible logic is scattered across:

```
TransformDeductibles.java (400+ lines)
├── processDeductibles() method
│   ├── if ("HO3".equals(...) && !isSC && !isGA) { ... }
│   ├── if ("HO3".equals(...) && isSC) { ... }
│   ├── if ("HO6".equals(...)) { ... }
│   ├── if ("DP1".equals(...)) { ... }
│   └── ... (massive nested conditionals)
│
DeductibleUtils.java (300+ lines)
├── determineSCDeductibles()
├── determineNCHO5Deductibles()
├── determineHO6HurricaneDeductible()
├── determineFLHO6AOPDeductible()
├── determineDP1AOPDeductible()
└── ... (20+ static methods)
```

### Visual Representation of Current Logic

```
                    ┌─────────────────┐
                    │ QuoteRequest    │
                    │ productType: ?  │
                    │ state: ?        │
                    └────────┬────────┘
                             │
                             ↓
        ┌────────────────────────────────────────┐
        │  TransformDeductibles.processDeductibles│
        └────────────────────┬───────────────────┘
                             │
        ┌────────────────────┴────────────────────┐
        │  Massive If-Else Chain                  │
        ├─────────────────────────────────────────┤
        │  if ("HO3" && !isSC && !isGA) {              │
        │    // FL HO3 logic (50 lines)               │
        │  } else if ("HO3" && isSC) {                │
        │    // SC HO3 logic (80 lines)               │
        │  } else if ("HO3" && isGA) {                │
        │    // GA HO3 logic (70 lines)               │
        │  } else if ("HO6") {                        │
        │    DeductibleUtils.determineHO6...()        │
        │    // Only FL HO6 exists today (100 lines)  │
        │  } else if ("DP1") {                        │
        │    DeductibleUtils.determineDP1...()        │
        │  } else if ("DP3") {                        │
        │    // DP3 logic (60 lines)                  │
        │  } else if ("HO4") {                        │
        │    // HO4 logic (40 lines)                  │
        │  } else if ("HO5" && (isSC || isGA)) {     │
        │    // HO5 logic (50 lines)                  │
        │  }                                          │
        └─────────────────────────────────────────────┘
```

---

## The Concrete Challenge

### Real Scenario: Adding NC HO6 Support

**Business Requirement**: "We need to offer HO6 (Condo) insurance in North Carolina"

**Current Implementation Required:**

```java
// In TransformDeductibles.java - Line ~88
} else if ("HO6".equals(request.getProductType())) {
    // CURRENT: Only handles FL
    DeductibleUtils.determineFLHO6AOPDeductibleWithTracking(request, building);
    
    // NEW: Need to add NC logic
    // But how? Another if-else?
    if ("FL".equals(state)) {
        DeductibleUtils.determineFLHO6AOPDeductibleWithTracking(request, building);
    } else if ("NC".equals(state)) {
        DeductibleUtils.determineNCHO6AOPDeductible(request, building);
    }
```

**Files You'd Have to Modify:**

1. **TransformDeductibles.java** - Add NC conditionals in 3+ places
2. **DeductibleUtils.java** - Add 4+ new static methods:
   - `determineNCHO6AOPDeductible()`
   - `determineNCHO6HurricaneDeductible()`
   - `determineNCHO6AOPDeductibleWithTracking()`
   - `determineNCHO6HurricaneDeductibleWithTracking()`
3. **GlobalConstants.java** - Add NC HO6 constants
4. **QuoteWS.java** - Potentially validation changes
5. **Tests** - If they existed, 10+ test files would need updates

**Estimated Effort**: 3-5 days of work, high risk of bugs

### The Scaling Problem

**Current Situation:**
- Products: HO3, HO4, HO5, HO6, DP1, DP3 = **6 products**
- States: FL, SC, GA, NC = **4 states**
- Not all products in all states

**Combinations Already Needed:**
```
HO3-FL, HO3-SC, HO3-GA, HO3-NC
HO4-FL
HO5-SC, HO5-GA, HO5-NC
HO6-FL (current), HO6-NC (new!)
DP1-FL
DP3-FL
= ~14 product-state combinations
```

**Each Combination Requires:**
- AOP Deductible logic
- Hurricane Deductible logic
- Wind/Hail logic
- Validation rules
- Tracking logic (for notifications)

**Result**: 400+ lines in TransformDeductibles + 300+ lines in DeductibleUtils = **700+ lines of nested conditionals**

---

## Strategy Pattern Solution

### What is the Strategy Pattern?

**Definition**: Define a family of algorithms, encapsulate each one, and make them interchangeable. Strategy lets the algorithm vary independently from clients that use it.

**In Simple Terms**: Instead of one big function with if-else for every case, create separate strategy classes where each class handles ONE specific case.

### Architecture Diagram

```
┌──────────────────────────────────────────────────────────┐
│                    Client Code                            │
│           (TransformDeductibles.processDeductibles)       │
└─────────────────────┬────────────────────────────────────┘
                      │
                      ↓
┌──────────────────────────────────────────────────────────┐
│           DeductibleCalculationService                    │
│  (selects appropriate strategy based on product/state)   │
└─────────────────────┬────────────────────────────────────┘
                      │ uses
                      ↓
         ┌────────────────────────┐
         │  DeductibleStrategy    │  ◄─── Interface
         │  (interface)           │
         └────────────────────────┘
                      △
                      │ implements
      ┌───────────────┴────────────────────────────┐
      │               │                │            │
      ↓               ↓                ↓            ↓
┌──────────┐   ┌──────────┐    ┌──────────┐  ┌──────────┐
│ HO6FL    │   │ HO6NC    │    │ HO3FL    │  │ HO3SC    │
│ Strategy │   │ Strategy │    │ Strategy │  │ Strategy │
└──────────┘   └──────────┘    └──────────┘  └──────────┘
  (existing)      (NEW!)        (existing)    (existing)

```

---

## ProductType & State Modeling

### Problem with Current Approach

**Current Code:**
```java
String productType = request.getProductType(); // "HO6"
Boolean isSC = false;
Boolean isGA = false;
String state = request.getInsuredProperty().getLocation().getState(); // "FL"

// Scattered string comparisons:
if ("HO6".equals(productType)) { ... }
if ("FL".equals(state)) { ... }
```

**Issues:**
1. **No Type Safety**: Can pass "HO7" or "XY" (invalid values)
2. **Scattered Logic**: State checks scattered across 20+ places
3. **Testing Difficulty**: Have to test all string combinations
4. **No IDE Support**: No autocomplete, easy typos
5. **No Validation**: "ho6" vs "HO6" vs "Ho6" all different

### Solution: Value Objects

Value Objects provide **type safety**, **validation**, and **immutability**.

#### 1. ProductType Value Object

```java
package com.aiig.quote.domain.valueobject;

import java.util.Objects;

/**
 * ProductType Value Object
 * 
 * Replaces: String productType
 * 
 * Benefits:
 * - Type safety (cannot pass invalid product)
 * - Validation at creation
 * - Behavior encapsulation
 * - IDE autocomplete support
 */
public final class ProductType {
    private final String code;
    
    // Private constructor - force use of factory methods
    private ProductType(String code) {
        this.code = Objects.requireNonNull(code, "Product code cannot be null");
    }
    
    // Factory methods - ONLY way to create valid instances
    public static ProductType HO3() { return new ProductType("HO3"); }
    public static ProductType HO4() { return new ProductType("HO4"); }
    public static ProductType HO5() { return new ProductType("HO5"); }
    public static ProductType HO6() { return new ProductType("HO6"); }
    public static ProductType DP1() { return new ProductType("DP1"); }
    public static ProductType DP3() { return new ProductType("DP3"); }
    
    /**
     * Create from string (for API requests)
     * Validates input!
     */
    public static ProductType fromString(String code) {
        if (code == null || code.trim().isEmpty()) {
            throw new IllegalArgumentException("Product code cannot be null or empty");
        }
        
        String normalized = code.trim().toUpperCase();
        
        return switch (normalized) {
            case "HO3" -> HO3();
            case "HO4" -> HO4();
            case "HO5" -> HO5();
            case "HO6" -> HO6();
            case "DP1" -> DP1();
            case "DP3" -> DP3();
            default -> throw new IllegalArgumentException(
                "Invalid product type: " + code + ". Valid values: HO3, HO4, HO5, HO6, DP1, DP3"
            );
        };
    }
    
    // Useful query methods
    public String getCode() { 
        return code; 
    }
    
    public boolean isHO6() { 
        return "HO6".equals(code); 
    }
    
    public boolean isHomeowners() { 
        return code.startsWith("HO"); 
    }
    
    public boolean isDwellingFire() { 
        return code.startsWith("DP"); 
    }
    
    public boolean isCondo() { 
        return "HO6".equals(code); 
    }
    
    // Value object equality (based on value, not reference)
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
    
    @Override
    public String toString() {
        return code;
    }
}
```

**Usage Example:**
```java
// OLD WAY (Current):
String productType = request.getProductType(); // "HO6"
if ("HO6".equals(productType)) { ... }  // Typo-prone, no validation

// NEW WAY (With Value Object):
ProductType productType = ProductType.fromString(request.getProductType());
if (productType.isHO6()) { ... }  // Type-safe, validated, IDE support

// Even better:
ProductType productType = ProductType.HO6();  // Compile-time safety!
```

#### 2. State Value Object

```java
package com.aiig.quote.domain.valueobject;

import java.util.Objects;
import java.util.Set;

/**
 * State Value Object
 * 
 * Replaces: String state, Boolean isSC, Boolean isGA, Boolean isNC
 * 
 * Benefits:
 * - Encapsulates state-specific behavior
 * - Type safety
 * - Eliminates Boolean flags
 */
public final class State {
    private final String code;
    
    // Coastal counties for each state (encapsulated!)
    private static final Set<String> FL_COASTAL_COUNTIES = Set.of();
    private static final Set<String> SC_COASTAL_COUNTIES = Set.of(
        "JASPER", "BEAUFORT", "COLLETON", "CHARLESTON", 
        "DORCHESTER", "BERKELEY", "GEORGETOWN", "HORRY", "WILLIAMSBURG"
    );
    private static final Set<String> GA_COASTAL_COUNTIES = Set.of(
        "CHATHAM", "BRYAN", "LIBERTY", "MCINTOSH", "GLYNN", "CAMDEN"
    );
    
    private State(String code) {
        this.code = Objects.requireNonNull(code);
    }
    
    // Factory methods
    public static State FL() { return new State("FL"); }
    public static State SC() { return new State("SC"); }
    public static State GA() { return new State("GA"); }
    public static State NC() { return new State("NC"); }
    
    public static State fromString(String code) {
        if (code == null || code.trim().isEmpty()) {
            throw new IllegalArgumentException("State code cannot be null or empty");
        }
        
        String normalized = code.trim().toUpperCase();
        
        return switch (normalized) {
            case "FL", "FLORIDA" -> FL();
            case "SC", "SOUTH CAROLINA" -> SC();
            case "GA", "GEORGIA" -> GA();
            case "NC", "NORTH CAROLINA" -> NC();
            default -> throw new IllegalArgumentException(
                "Invalid state: " + code + ". Valid values: FL, SC, GA, NC"
            );
        };
    }
    
    // Query methods - behavior lives in the object!
    public String getCode() { 
        return code; 
    }
    
    public boolean isFlorida() { 
        return "FL".equals(code); 
    }
    
    public boolean isSouthCarolina() { 
        return "SC".equals(code); 
    }
    
    public boolean isGeorgia() { 
        return "GA".equals(code); 
    }
    
    public boolean isNorthCarolina() { 
        return "NC".equals(code); 
    }
    
    /**
     * Check if county is coastal in this state
     * Encapsulates the coastal logic!
     */
    public boolean isCoastalCounty(String county) {
        if (county == null) return false;
        
        String normalizedCounty = county.trim().toUpperCase();
        
        return switch (code) {
            case "SC" -> SC_COASTAL_COUNTIES.contains(normalizedCounty);
            case "GA" -> GA_COASTAL_COUNTIES.contains(normalizedCounty);
            case "FL", "NC" -> false; // No coastal counties defined for FL/NC
            default -> false;
        };
    }
    
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        State state = (State) o;
        return Objects.equals(code, state.code);
    }
    
    @Override
    public int hashCode() {
        return Objects.hash(code);
    }
    
    @Override
    public String toString() {
        return code;
    }
}
```

**Benefits Demonstrated:**

```java
// OLD WAY:
Boolean isSC = "SC".equals(request.getInsuredProperty().getLocation().getState());
Boolean isGA = "GA".equals(request.getInsuredProperty().getLocation().getState());
String county = request.getInsuredProperty().getLocation().getCounty();
List<String> coastalCountiesSCList = Arrays.asList("JASPER", "BEAUFORT", ...);
boolean isCoastal = (isSC && coastalCountiesSCList.contains(county.toUpperCase()));

// NEW WAY:
State state = State.fromString(request.getInsuredProperty().getLocation().getState());
String county = request.getInsuredProperty().getLocation().getCounty();
boolean isCoastal = state.isCoastalCounty(county);  // One line!
```

---

## Complete Implementation

### Step 1: Define the Strategy Interface

```java
package com.aiig.quote.service.strategy;

import com.aiig.quote.domain.valueobject.ProductType;
import com.aiig.quote.domain.valueobject.State;
import com.aiig.quote.model.QuoteRequest;
import com.aiig.spin.model.DTOBuilding;

/**
 * Strategy interface for deductible calculation
 * 
 * Each product-state combination implements this interface
 * 
 * Replaces: Static methods in DeductibleUtils
 */
public interface DeductibleStrategy {
    
    /**
     * Calculate and set deductibles for this product/state combination
     * 
     * @param request The quote request
     * @param building The building DTO to populate
     * @return Updated building with deductibles set
     * @throws Exception if calculation fails
     */
    DTOBuilding calculateDeductibles(QuoteRequest request, DTOBuilding building) throws Exception;
    
    /**
     * Check if this strategy supports the given product and state
     * 
     * @param productType The product type
     * @param state The state
     * @return true if this strategy handles this combination
     */
    boolean supports(ProductType productType, State state);
    
    /**
     * Get strategy name for logging/debugging
     */
    default String getStrategyName() {
        return this.getClass().getSimpleName();
    }
}
```

### Step 2: Implement Concrete Strategies

#### Strategy for FL HO6 (Existing Logic)

```java
package com.aiig.quote.service.strategy.impl;

import org.springframework.stereotype.Component;
import com.aiig.quote.domain.valueobject.ProductType;
import com.aiig.quote.domain.valueobject.State;
import com.aiig.quote.model.GlobalConstants;
import com.aiig.quote.model.QuoteRequest;
import com.aiig.quote.service.CoverageUtils;
import com.aiig.quote.service.strategy.DeductibleStrategy;
import com.aiig.spin.model.DTOBuilding;
import java.math.BigInteger;

/**
 * HO6 Florida Deductible Strategy
 * 
 * Encapsulates ALL FL HO6 deductible logic
 * Extracted from DeductibleUtils.determineHO6HurricaneDeductible()
 * and DeductibleUtils.determineFLHO6AOPDeductible()
 */
@Component
public class HO6FloridaDeductibleStrategy implements DeductibleStrategy {
    
    @Override
    public DTOBuilding calculateDeductibles(QuoteRequest request, DTOBuilding building) throws Exception {
        // Calculate Hurricane deductible
        calculateHurricaneDeductible(request, building);
        
        // Calculate AOP deductible
        calculateAOPDeductible(request, building);
        
        return building;
    }
    
    @Override
    public boolean supports(ProductType productType, State state) {
        return productType.isHO6() && state.isFlorida();
    }
    
    /**
     * FL HO6 Hurricane Deductible Logic
     * Extracted from: DeductibleUtils.determineHO6HurricaneDeductible()
     */
    private void calculateHurricaneDeductible(QuoteRequest request, DTOBuilding building) {
        // Determine CovC (this is state/product specific)
        CoverageUtils.determineFLHO6CovC(request, building);
        
        BigInteger sum_covC_covA = request.getCoverages().getCovA()
            .add(request.getCoverages().getCovC());
        
        if (request.getDeductibles() != null) {
            if (!request.getDeductibles().getWindHailExclusion()) {
                calculateHurricaneWithoutWindHailExclusion(request, building, sum_covC_covA);
            } else {
                calculateHurricaneWithWindHailExclusion(request, building);
            }
        } else {
            building.setHurricaneDeductible(GlobalConstants.FL_HO6_HURRICANEDED_VALUE3);
        }
    }
    
    private void calculateHurricaneWithoutWindHailExclusion(
        QuoteRequest request, DTOBuilding building, BigInteger sum_covC_covA
    ) {
        if (request.getDeductibles().getHurricane() != null) {
            // Apply tiered logic based on CovA + CovC
            if (sum_covC_covA.compareTo(GlobalConstants.FL_HO6_COVA_PLUS_COVC_VALUE1) < 0) {
                if (request.getDeductibles().getHurricane().toString()
                    .matches(GlobalConstants.FL_HO6_HURR_DEDUCTIBLE_VALUES)) {
                    building.setHurricaneDeductible(
                        request.getDeductibles().getHurricane().toString()
                    );
                }
            } else if (sum_covC_covA.compareTo(GlobalConstants.FL_HO6_COVA_PLUS_COVC_VALUE1) >= 0 
                    && sum_covC_covA.compareTo(GlobalConstants.FL_HO6_COVA_PLUS_COVC_VALUE2) <= 0) {
                // Middle tier logic
                if (request.getDeductibles().getHurricane().toString()
                    .equals(GlobalConstants.FL_HO6_HURRICANEDED_VALUE2)) {
                    building.setHurricaneDeductible(GlobalConstants.FL_HO6_HURRICANEDED_VALUE2);
                } else {
                    building.setHurricaneDeductible(
                        request.getDeductibles().getHurricane().toString()
                    );
                }
            } else if (sum_covC_covA.compareTo(GlobalConstants.FL_HO6_COVA_PLUS_COVC_VALUE2) > 0 
                    && sum_covC_covA.compareTo(GlobalConstants.FL_HO6_COVA_PLUS_COVC_VALUE3) <= 0) {
                // Upper-middle tier
                if (request.getDeductibles().getHurricane().toString()
                    .matches(GlobalConstants.FL_HO6_HURRICANEDED_VALUE2)) {
                    building.setHurricaneDeductible(GlobalConstants.FL_HO6_HURRICANEDED_VALUE3);
                } else {
                    building.setHurricaneDeductible(
                        request.getDeductibles().getHurricane().toString()
                    );
                }
            } else if (sum_covC_covA.compareTo(GlobalConstants.FL_HO6_COVA_PLUS_COVC_VALUE3) > 0) {
                // Highest tier
                building.setHurricaneDeductible(GlobalConstants.FL_HO6_HURRICANEDED_VALUE4);
            }
        } else {
            // No hurricane deductible provided - use default based on coverage
            if (sum_covC_covA.compareTo(GlobalConstants.FL_HO6_COVA_PLUS_COVC_VALUE3) <= 0) {
                building.setHurricaneDeductible(GlobalConstants.FL_HO6_HURRICANEDED_VALUE3);
            } else {
                building.setHurricaneDeductible(GlobalConstants.FL_HO6_HURRICANEDED_VALUE4);
            }
        }
    }
    
    private void calculateHurricaneWithWindHailExclusion(
        QuoteRequest request, DTOBuilding building
    ) {
        if (request.getDeductibles().getHurricane() != null) {
            int hurricaneValue = request.getDeductibles().getHurricane().intValue();
            
            if (hurricaneValue <= 750) {
                building.setHurricaneDeductible(GlobalConstants.FL_HO6_HURRICANEDED_VALUE1);
            } else if (hurricaneValue <= 1750) {
                building.setHurricaneDeductible(GlobalConstants.FL_HO6_HURRICANEDED_VALUE2);
            } else if (hurricaneValue <= 3950) {
                building.setHurricaneDeductible(GlobalConstants.FL_HO6_HURRICANEDED_VALUE3);
            } else {
                building.setHurricaneDeductible(GlobalConstants.FL_HO6_HURRICANEDED_VALUE4);
            }
        } else {
            building.setHurricaneDeductible(GlobalConstants.FL_HO6_HURRICANEDED_VALUE3);
        }
    }
    
    /**
     * FL HO6 AOP Deductible Logic
     * Extracted from: DeductibleUtils.determineFLHO6AOPDeductible()
     */
    private void calculateAOPDeductible(QuoteRequest request, DTOBuilding building) throws Exception {
        if (request.getDeductibles() != null) {
            if (Boolean.TRUE.equals(request.getDeductibles().getWindHailExclusion())) {
                calculateAOPWithWindHailExclusion(request, building);
            } else {
                calculateAOPWithoutWindHailExclusion(request, building);
            }
        } else {
            building.setAllPerilDed(GlobalConstants.FL_HO6_AOPDED_VALUE3);
        }
    }
    
    private void calculateAOPWithWindHailExclusion(QuoteRequest request, DTOBuilding building) {
        if (request.getDeductibles().getAop() != null) {
            if (request.getDeductibles().getAop().toString()
                .matches(GlobalConstants.FL_HO6_AOP_DEDUCTIBLE_VALUES)) {
                building.setAllPerilDed(request.getDeductibles().getAop().toString());
            } else {
                building.setAllPerilDed(GlobalConstants.FL_HO6_AOPDED_VALUE3);
            }
        } else {
            building.setAllPerilDed(GlobalConstants.FL_HO6_AOPDED_VALUE3);
        }
    }
    
    private void calculateAOPWithoutWindHailExclusion(QuoteRequest request, DTOBuilding building) {
        String hurricaneValue = request.getDeductibles().getHurricane().toString();
        
        if (hurricaneValue.equals(GlobalConstants.FL_HO6_AOPDED_VALUE1)) {
            building.setAllPerilDed(GlobalConstants.FL_HO6_AOPDED_VALUE1);
        } else if (hurricaneValue.equals(GlobalConstants.FL_HO6_AOPDED_VALUE2)) {
            if (request.getDeductibles().getAop() != null) {
                if (request.getDeductibles().getAop().toString()
                    .equals(GlobalConstants.FL_HO6_AOPDED_VALUE1)) {
                    building.setAllPerilDed(GlobalConstants.FL_HO6_AOPDED_VALUE1);
                } else {
                    building.setAllPerilDed(GlobalConstants.FL_HO6_AOPDED_VALUE2);
                }
            } else {
                building.setAllPerilDed(GlobalConstants.FL_HO6_AOPDED_VALUE2);
            }
        } else if (hurricaneValue.equals(GlobalConstants.FL_HO6_AOPDED_VALUE3)) {
            if (request.getDeductibles().getAop() != null) {
                if (request.getDeductibles().getAop().toString()
                    .matches(GlobalConstants.FL_HO6_AOP_DEDUCTIBLE_VALUES2)) {
                    building.setAllPerilDed(request.getDeductibles().getAop().toString());
                } else {
                    building.setAllPerilDed(GlobalConstants.FL_HO6_AOPDED_VALUE3);
                }
            } else {
                building.setAllPerilDed(GlobalConstants.FL_HO6_AOPDED_VALUE3);
            }
        } else if (hurricaneValue.equals(GlobalConstants.FL_HO6_AOPDED_VALUE4)) {
            if (request.getDeductibles().getAop() != null) {
                if (request.getDeductibles().getAop().toString()
                    .matches(GlobalConstants.FL_HO6_AOP_DEDUCTIBLE_VALUES)) {
                    building.setAllPerilDed(request.getDeductibles().getAop().toString());
                } else {
                    building.setAllPerilDed(GlobalConstants.FL_HO6_AOPDED_VALUE4);
                }
            } else {
                building.setAllPerilDed(GlobalConstants.FL_HO6_AOPDED_VALUE4);
            }
        }
    }
}
```

#### Strategy for NC HO6 (NEW - Easy to Add!)

```java
package com.aiig.quote.service.strategy.impl;

import org.springframework.stereotype.Component;
import com.aiig.quote.domain.valueobject.ProductType;
import com.aiig.quote.domain.valueobject.State;
import com.aiig.quote.model.QuoteRequest;
import com.aiig.quote.service.strategy.DeductibleStrategy;
import com.aiig.spin.model.DTOBuilding;

/**
 * HO6 North Carolina Deductible Strategy
 * 
 * NEW PRODUCT-STATE COMBINATION!
 * 
 * Notice: Only ONE file to add, no changes to existing code!
 */
@Component
public class HO6NorthCarolinaDeductibleStrategy implements DeductibleStrategy {
    
    @Override
    public DTOBuilding calculateDeductibles(QuoteRequest request, DTOBuilding building) throws Exception {
        // NC HO6 specific logic here
        calculateNCHO6HurricaneDeductible(request, building);
        calculateNCHO6AOPDeductible(request, building);
        return building;
    }
    
    @Override
    public boolean supports(ProductType productType, State state) {
        return productType.isHO6() && state.isNorthCarolina();
    }
    
    private void calculateNCHO6HurricaneDeductible(QuoteRequest request, DTOBuilding building) {
        // NC-specific hurricane deductible rules
        // Example: NC might have different tiers, different defaults
        if (request.getDeductibles() != null && request.getDeductibles().getHurricane() != null) {
            // Validate against NC allowed values
            if (request.getDeductibles().getHurricane().toString().matches("1000|2500|5000")) {
                building.setHurricaneDeductible(request.getDeductibles().getHurricane().toString());
            } else {
                building.setHurricaneDeductible("2500"); // NC default
            }
        } else {
            building.setHurricaneDeductible("2500");
        }
    }
    
    private void calculateNCHO6AOPDeductible(QuoteRequest request, DTOBuilding building) {
        // NC-specific AOP rules
        if (request.getDeductibles() != null && request.getDeductibles().getAop() != null) {
            if (request.getDeductibles().getAop().toString().matches("500|1000|2500")) {
                building.setAllPerilDed(request.getDeductibles().getAop().toString());
            } else {
                building.setAllPerilDed("1000"); // NC default
            }
        } else {
            building.setAllPerilDed("1000");
        }
    }
}
```

### Step 3: Service to Select Strategy

```java
package com.aiig.quote.service;

import org.springframework.stereotype.Service;
import com.aiig.quote.domain.valueobject.ProductType;
import com.aiig.quote.domain.valueobject.State;
import com.aiig.quote.model.QuoteRequest;
import com.aiig.quote.service.strategy.DeductibleStrategy;
import com.aiig.spin.model.DTOBuilding;
import java.util.List;

/**
 * Service that selects and delegates to appropriate strategy
 * 
 * This replaces the massive if-else chains in TransformDeductibles
 */
@Service
public class DeductibleCalculationService {
    
    private final List<DeductibleStrategy> strategies;
    
    // Spring auto-injects all strategies
    public DeductibleCalculationService(List<DeductibleStrategy> strategies) {
        this.strategies = strategies;
    }
    
    /**
     * Calculate deductibles using appropriate strategy
     * 
     * @param request Quote request
     * @param building Building to populate
     * @return Updated building
     * @throws Exception if no strategy found or calculation fails
     */
    public DTOBuilding calculateDeductibles(QuoteRequest request, DTOBuilding building) throws Exception {
        // Parse product and state
        ProductType productType = ProductType.fromString(request.getProductType());
        State state = State.fromString(
            request.getInsuredProperty().getLocation().getState()
        );
        
        // Find appropriate strategy
        DeductibleStrategy strategy = strategies.stream()
            .filter(s -> s.supports(productType, state))
            .findFirst()
            .orElseThrow(() -> new IllegalArgumentException(
                String.format("No deductible strategy found for %s in %s", 
                    productType.getCode(), state.getCode())
            ));
        
        // Log which strategy is being used
        System.out.println("Using strategy: " + strategy.getStrategyName());
        
        // Delegate to strategy
        return strategy.calculateDeductibles(request, building);
    }
}
```

### Step 4: Update TransformDeductibles (Simplified!)

```java
package com.aiig.quote.service;

import com.aiig.quote.model.QuoteRequest;
import com.aiig.spin.model.DTOBuilding;
import org.springframework.stereotype.Service;

@Service
public class TransformDeductibles {
    
    private final DeductibleCalculationService deductibleService;
    
    public TransformDeductibles(DeductibleCalculationService deductibleService) {
        this.deductibleService = deductibleService;
    }
    
    public DTOBuilding processDeductibles(
        DTOBuilding building, 
        QuoteRequest request, 
        String productVersion
    ) throws Exception {
        
        // Reset tracking flags
        DeductibleUtils.resetFlHO6DeductibleFlags();
        
        // NEW: Single line replaces 400+ lines of conditionals!
        return deductibleService.calculateDeductibles(request, building);
    }
}
```

**THAT'S IT!** From 400+ lines to ~10 lines!

---

## Benefits Analysis

### Before: Adding NC HO6

**Files to Modify:**
1. `TransformDeductibles.java` - Add conditionals (3 places)
2. `DeductibleUtils.java` - Add 4 new methods
3. `GlobalConstants.java` - Add NC constants
4. `QuoteWS.java` - Validation changes
5. Tests - Update 10+ test files

**Lines of Code Changed:** 200-300 lines  
**Risk:** HIGH (touching shared code)  
**Time:** 3-5 days  
**Testing Effort:** Regression test everything

### After: Adding NC HO6 with Strategy

**Files to Add:**
1. `HO6NorthCarolinaDeductibleStrategy.java` - NEW file only!

**Files to Modify:**
0 (Zero!)

**Lines of Code:** 100 lines (in new file)  
**Risk:** LOW (isolated, no shared code touched)  
**Time:** 4-8 hours  
**Testing Effort:** Test only new strategy

### Comparison Table

| Aspect | Before (Current) | After (Strategy) |
|--------|-----------------|------------------|
| **Files Modified** | 5+ | 0 |
| **New Files** | 0 | 1 |
| **LOC Changed** | 200-300 | 100 (new) |
| **Risk Level** | HIGH | LOW |
| **Testing Scope** | Everything | New strategy only |
| **Development Time** | 3-5 days | 4-8 hours |
| **Bugs Introduced** | Likely | Unlikely |
| **Code Review** | Complex | Simple |

---

## Real-World Scenarios

### Scenario 1: Adding SC DP3

**Business Need**: Offer DP3 (Dwelling Fire) in South Carolina

**Current Approach:**
```java
// In TransformDeductibles.processDeductibles()
// Add another else-if branch (already 400 lines!)
} else if ("DP3".equals(productType) && isSC) {
    // 80+ lines of SC DP3 logic
    // Hope you don't break FL DP3!
}
```

**Strategy Approach:**
```java
// Create DP3SouthCarolinaDeductibleStrategy.java
@Component
public class DP3SouthCarolinaDeductibleStrategy implements DeductibleStrategy {
    @Override
    public boolean supports(ProductType productType, State state) {
        return productType == ProductType.DP3() && state.isSouthCarolina();
    }
    
    @Override
    public DTOBuilding calculateDeductibles(...) {
        // SC DP3 logic here
        // Isolated, doesn't affect anything else
    }
}
```
Done! No other files touched.

### Scenario 2: Changing FL HO6 Rules

**Business Need**: FL changes hurricane deductible tiers

**Current Approach:**
```java
// Find determineHO6HurricaneDeductible() in DeductibleUtils
// Modify nested if-else logic
// Cross fingers you didn't break something
// Run ALL tests (if they exist)
```

**Strategy Approach:**
```java
// Modify only HO6FloridaDeductibleStrategy
// Change tier thresholds
// Test only this strategy
// Other products/states unaffected
```

### Scenario 3: A/B Testing Different Rules

**Business Need**: Test different deductible rules for HO6 in FL

**Current Approach:**
Difficult! Would need feature flags sprinkled throughout static methods.

**Strategy Approach:**
```java
@Component
@Profile("experiment-a")
public class HO6FloridaDeductibleStrategyExperimentA implements DeductibleStrategy {
    // Variant A logic
}

@Component
@Profile("experiment-b")
public class HO6FloridaDeductibleStrategyExperimentB implements DeductibleStrategy {
    // Variant B logic
}
```

Switch with Spring profile! No code changes.

---

## Migration Strategy

### Phase 1: Add New Classes (No Breaking Changes)

**Week 1:**
```
1. Create value objects (ProductType, State)
2. Create DeductibleStrategy interface
3. Create DeductibleCalculationService
4. Create HO6FloridaDeductibleStrategy (extract existing logic)
5. Add comprehensive tests
```

**Result:** New code exists, old code still works

### Phase 2: Parallel Run (Validation)

**Week 2:**
```java
public DTOBuilding processDeductibles(...) {
    // OLD WAY (temporary)
    DTOBuilding oldResult = processDeductiblesOldWay(building, request);
    
    // NEW WAY
    DTOBuilding newResult = deductibleService.calculateDeductibles(request, building);
    
    // COMPARE
    if (!oldResult.equals(newResult)) {
        logger.warn("Mismatch detected! Old: {}, New: {}", oldResult, newResult);
    }
    
    // Return old for now (safe)
    return oldResult;
}
```

**Result:** Confidence that new code works identically

### Phase 3: Switch Over

**Week 3:**
```java
public DTOBuilding processDeductibles(...) {
    // NEW WAY only
    return deductibleService.calculateDeductibles(request, building);
}
```

**Result:** Using strategy pattern, old code dormant

### Phase 4: Cleanup

**Week 4:**
```
1. Delete old static methods
2. Remove old conditionals
3. Update documentation
4. Celebrate! 🎉
```

**Result:** Clean, maintainable code

---

## Testing Examples

### Testing a Strategy (Easy!)

```java
@Test
void testHO6FloridaStrategy() {
    // Given
    HO6FloridaDeductibleStrategy strategy = new HO6FloridaDeductibleStrategy();
    QuoteRequest request = createHO6Request(); // Test data
    DTOBuilding building = new DTOBuilding();
    
    // When
    DTOBuilding result = strategy.calculateDeductibles(request, building);
    
    // Then
    assertEquals("2500", result.getHurricaneDeductible());
    assertEquals("1000", result.getAllPerilDed());
}

@Test
void testStrategySupports() {
    HO6FloridaDeductibleStrategy strategy = new HO6FloridaDeductibleStrategy();
    
    // Should support HO6 + FL
    assertTrue(strategy.supports(ProductType.HO6(), State.FL()));
    
    // Should NOT support HO6 + NC
    assertFalse(strategy.supports(ProductType.HO6(), State.NC()));
    
    // Should NOT support HO3 + FL
    assertFalse(strategy.supports(ProductType.HO3(), State.FL()));
}
```

### Testing Strategy Selection

```java
@Test
void testServiceSelectsCorrectStrategy() {
    // Given
    List<DeductibleStrategy> strategies = List.of(
        new HO6FloridaDeductibleStrategy(),
        new HO6NorthCarolinaDeductibleStrategy(),
        new HO3FloridaDeductibleStrategy()
    );
    DeductibleCalculationService service = new DeductibleCalculationService(strategies);
    
    QuoteRequest request = new QuoteRequest();
    request.setProductType("HO6");
    request.getInsuredProperty().getLocation().setState("NC");
    
    // When
    DTOBuilding result = service.calculateDeductibles(request, new DTOBuilding());
    
    // Then
    // Should have used HO6NorthCarolinaDeductibleStrategy
    assertEquals("2500", result.getHurricaneDeductible()); // NC default
}
```

---

## Conclusion

### The Problem We Solved

**Before:**
- 400+ lines of nested if-else
- Product × State = combinatorial explosion
- Adding NC HO6 = 3-5 days, high risk
- Testing nightmare
- Code scattered everywhere

**After:**
- Clean strategy interface
- One class per product-state combination
- Adding NC HO6 = 4-8 hours, low risk
- Easy to test each strategy
- Code isolated and focused

### Key Takeaways

1. **Strategy Pattern** eliminates conditional explosion
2. **Value Objects** (ProductType, State) provide type safety
3. **Each strategy is independent** - add without fear
4. **Spring auto-wires strategies** - zero configuration
5. **Migration can be gradual** - no big bang rewrite

### Next Steps

1. Review this document with team
2. Create value objects (ProductType, State)
3. Extract one strategy (HO6 Florida) as proof of concept
4. Validate it works identically to current code
5. Add NC HO6 as second strategy
6. Measure time savings and risk reduction
7. Roll out to other product-state combinations

---

**This is not theoretical - it directly solves your existing codebase problems!**
