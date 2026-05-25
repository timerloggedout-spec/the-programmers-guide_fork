# Financial System Design Patterns Interview Notes

## About

This page is a consolidated reference for understanding and revisiting software design patterns through real-world financial system examples. The focus is not only on theoretical definitions, but also on how these patterns are practically applied in enterprise backend systems such as payment platforms, banking applications, and microservices architectures.

## Creational

### Singleton

Singleton ensures a single instance with global access. In Java, thread safety is achieved using double-checked locking with volatile or Bill Pugh pattern. However, in modern Spring Boot applications, Singleton is managed by the Spring container using default bean scope. We used it for caching exchange rates and configuration to maintain consistency across payment flows. One must be careful as Singleton can be broken via reflection or serialization and can hurt testability if not used with dependency injection

Without Singleton:

* Multiple DB connection managers → inconsistent state
* Multiple cache instances → memory issues
* Multiple config loaders → bugs

{% hint style="success" %}
Spring Boot Reality

You usually **DON’T implement Singleton manually**

👉 Spring beans are Singleton by default:

```
@Service
public class ExchangeRateService {}
```

Spring ensures:

* Only one instance exists
* Thread-safe lifecycle
{% endhint %}

Use Case: **Central Configuration Manager**

In banking/payment system:

* Exchange rate config
* Feature flags (IMPS enabled, UPI limits)
* AML rules

Must be **single source of truth**

```
public class ConfigManager {

    private static volatile ConfigManager instance;

    private ConfigManager() {
        // load configs (DB / file / remote service)
    }

    public static ConfigManager getInstance() {
        if (instance == null) {
            synchronized (ConfigManager.class) {
                if (instance == null) {
                    instance = new ConfigManager();
                }
            }
        }
        return instance;
    }

    public String getValue(String key) {
        return "value"; // fetch from cache/map
    }
}
```

{% hint style="info" %}
In the Double-Checked Locking singleton pattern, `volatile` is extremely important.

Without `volatile`, the code is broken:

```
private static ConfigManager instance;
```

It may work most of the time, but under concurrency another thread can get a partially initialized object.



Why?

This line:

```
instance = new ConfigManager();
```

is not actually a single atomic operation.

Internally JVM may do:

1. Allocate memory
2. Assign memory reference to `instance`
3. Execute constructor

Due to JVM/compiler/CPU optimizations, steps 2 and 3 can be reordered.

So actual execution may become:

1. Allocate memory
2. Assign reference to `instance`
3. Constructor executes

Now imagine:

Thread A:

```
instance = new ConfigManager();
```

Thread B:

```
if (instance != null)
```

Since reference assignment already happened, Thread B sees non-null `instance` and returns it.

But constructor may not have completed yet.

So Thread B gets a partially initialized object.

`volatile` provides:

1. Visibility guarantee
   * One thread's updates are immediately visible to others.
2. Prevents instruction reordering
   * JVM cannot reorder constructor completion after reference assignment.

So JVM must guarantee:

1. Allocate memory
2. Run constructor fully
3. Assign reference to `instance`

Only after complete construction can another thread see the object.
{% endhint %}

**Question**

_Where did you use Singleton in your project?_

> "In Spring Boot, most services are Singleton by default. For example, we used a singleton cache for exchange rates and feature configurations to ensure consistency across all payment flows and avoid repeated external API calls."

_How to Make Singleton Thread-Safe?_

```
// Basic Singleton breaks in multithreading
if (instance == null) {
    instance = new Singleton(); // race condition
}

// Synchronized Method (simple but slow)
public static synchronized Singleton getInstance() {
    if (instance == null) {
        instance = new Singleton();
    }
    return instance;
}

// Double-Checked Locking (best practical answer)
public class Singleton {
    private static volatile Singleton instance;

    private Singleton() {}

    public static Singleton getInstance() {
        if (instance == null) {
            synchronized (Singleton.class) {
                if (instance == null) {
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}

// Bill Pugh (Best + Clean)
public class Singleton {

    private Singleton() {}

    private static class Holder {
        private static final Singleton INSTANCE = new Singleton();
    }

    public static Singleton getInstance() {
        return Holder.INSTANCE;
    }
}

// Enum Singleton (Most robust)
public enum Singleton {
    INSTANCE;

    public void doSomething() {}
}

```

How Can Singleton Be Broken?

```
// Reflection Attack
Constructor<Singleton> constructor = Singleton.class.getDeclaredConstructor();
constructor.setAccessible(true);
Singleton newInstance = constructor.newInstance();

// Fix
private Singleton() {
    if (instance != null) {
        throw new RuntimeException("Use getInstance()");
    }
}

// Serialization Break
ObjectOutputStream.writeObject(instance);
ObjectInputStream.readObject(); // creates new instance

// Fix
protected Object readResolve() {
    return instance;
}

// Cloning
singleton.clone(); // new instance

// Fix
@Override
protected Object clone() throws CloneNotSupportedException {
    throw new CloneNotSupportedException();
}
```

Spring Singleton vs GoF Singleton

| Aspect      | GoF Singleton | Spring Singleton  |
| ----------- | ------------- | ----------------- |
| Scope       | JVM-wide      | Spring container  |
| Creation    | Manual        | Framework-managed |
| Testability | Hard          | Easy              |
| Flexibility | Low           | High              |

### Factory Method Pattern

Factory pattern encapsulates object creation logic and removes tight coupling from client code. In our payment system, we used a factory to route transactions dynamically to UPI, IMPS, or NEFT processors. With Spring Boot, we leveraged map-based injection of beans to make it extensible without modifying existing code.

Define an interface for creating an object, but let subclasses decide **which class to instantiate**.



Problem It Solves ?

Without Factory:

{% code overflow="wrap" %}
```
PaymentProcessor processor;
if (type.equals("UPI")) {    
    processor = new UpiProcessor();
} else if (type.equals("IMPS")) {
    processor = new ImpsProcessor();
}
```
{% endcode %}

❌ Issues:

* Tight coupling
* Violates Open/Closed Principle
* Hard to extend (add NEFT → modify code everywhere)

✅ With Factory Pattern

```
PaymentProcessor processor = factory.getProcessor(type);
```

&#x20;Client doesn’t care **which implementation**



Use Case

💳 Payment Routing System

A bank supports:

* UPI
* IMPS
* NEFT
* RTGS

Each has:

* different APIs
* different validations
* different SLAs

Factory decides **which processor to use**

🧱 Structure

```
Product (interface)
   ↑
ConcreteProduct (UPI / IMPS / NEFT)

Creator (Factory)
   ↑
ConcreteCreator (optional in Java, often single class)
```

Example

<pre data-overflow="wrap"><code>// Step 1: Product Interface

public interface PaymentProcessor {
    void processPayment(PaymentRequest request);
}

// Step 2: Implementations

@Component
public class UpiPaymentProcessor implements PaymentProcessor {
    public void processPayment(PaymentRequest request) {
        System.out.println("Processing UPI payment");
    }
}

@Component
public class ImpsPaymentProcessor implements PaymentProcessor {
    public void processPayment(PaymentRequest request) {
        System.out.println("Processing IMPS payment");
    }
}

<strong>// Step 3: Factory
</strong>@Component
public class PaymentProcessorFactory {

    private final Map&#x3C;String, PaymentProcessor> processors;

    public PaymentProcessorFactory(List&#x3C;PaymentProcessor> processorList) {
        this.processors = processorList.stream()
                .collect(Collectors.toMap(
                    p -> p.getClass().getSimpleName(), 
                    Function.identity()
                ));
    }

    public PaymentProcessor getProcessor(String type) {
        return switch (type) {
            case "UPI" -> processors.get("UpiPaymentProcessor");
            case "IMPS" -> processors.get("ImpsPaymentProcessor");
            default -> throw new IllegalArgumentException("Unsupported type");
        };
    }
}

// Step 4: Usage
@Service
public class PaymentService {

    private final PaymentProcessorFactory factory;

    public PaymentService(PaymentProcessorFactory factory) {
        this.factory = factory;
    }

    public void process(PaymentRequest request) {
        PaymentProcessor processor = factory.getProcessor(request.getType());
        processor.processPayment(request);
    }
}
<strong>
</strong></code></pre>

{% code overflow="wrap" %}
```
// Spring already helps implement Factory pattern implicitly

@Component("UPI")
public class UpiPaymentProcessor implements PaymentProcessor {}

@Component("IMPS")
public class ImpsPaymentProcessor implements PaymentProcessor {}

// Cleaner Version Using Map Injection
@Component
public class PaymentProcessorFactory {

    private final Map<String, PaymentProcessor> processorMap;

    public PaymentProcessorFactory(Map<String, PaymentProcessor> processorMap) {
        this.processorMap = processorMap;
    }

    public PaymentProcessor getProcessor(String type) {
        return processorMap.get(type);
    }
}

```
{% endcode %}

**Question**

Factory vs Strategy (VERY COMMON)

| Factory                | Strategy          |
| ---------------------- | ----------------- |
| Creates object         | Uses object       |
| Focus on instantiation | Focus on behavior |
| Happens once           | Used repeatedly   |

Together in real systems:

* Factory → chooses processor
* Strategy → processor executes logic

{% code overflow="wrap" %}
```
public void process(PaymentRequest request) {
    PaymentProcessor processor = factory.getProcessor(request.getType()); // Factory
    processor.processPayment(request); // Strategy
}
```
{% endcode %}

### Abstract Factory Pattern

Abstract Factory is used to create families of related objects. In our system, we used it for multi-country payment processing where each country had its own processor, validator, and fee calculator. This ensured consistency and prevented mixing incompatible components while keeping the system extensible.

Problem It Solves ?

Let’s say we’re building a **multi-country payment system**:

Each country has:

* Payment Processor
* Validator
* Fee Calculator

Without Abstract Factory:

{% code overflow="wrap" %}
```
if (country.equals("INDIA")) {
    processor = new IndiaPaymentProcessor();
    validator = new IndiaValidator();
    feeCalculator = new IndiaFeeCalculator();
} else if (country.equals("USA")) {
    processor = new UsPaymentProcessor();
    validator = new UsValidator();
    feeCalculator = new UsFeeCalculator();
}
```
{% endcode %}

#### ❌ Problem:

* Risk of mixing objects (India processor + US validator = bug)
* Tight coupling
* Hard to scale

Solution: Abstract Factory

👉 Ensure **consistent object family creation**

🧱 Structure

```
AbstractFactory
   ├── createProcessor()
   ├── createValidator()
   ├── createFeeCalculator()

ConcreteFactory (IndiaFactory / USFactory)
   ├── returns country-specific objects
```

Example

<pre data-overflow="wrap"><code><strong>// Step 1: Product Interfaces
</strong>public interface PaymentProcessor {
    void process();
}

public interface PaymentValidator {
    void validate();
}

public interface FeeCalculator {
    double calculateFee();
}

// Step 2: Concrete Implementations (India)
public class IndiaPaymentProcessor implements PaymentProcessor {
    public void process() {
        System.out.println("Processing India payment");
    }
}

public class IndiaValidator implements PaymentValidator {
    public void validate() {
        System.out.println("Validating India rules");
    }
}

public class IndiaFeeCalculator implements FeeCalculator {
    public double calculateFee() {
        return 5.0;
    }
}

// Step 3: Concrete Implementations (US)
public class UsPaymentProcessor implements PaymentProcessor {
    public void process() {
        System.out.println("Processing US payment");
    }
}

// Step 4: Abstract Factory
public interface PaymentFactory {
    PaymentProcessor createProcessor();
    PaymentValidator createValidator();
    FeeCalculator createFeeCalculator();
}

// Step 5: Concrete Factories
@Component("INDIA")
public class IndiaPaymentFactory implements PaymentFactory {

    public PaymentProcessor createProcessor() {
        return new IndiaPaymentProcessor();
    }

    public PaymentValidator createValidator() {
        return new IndiaValidator();
    }

    public FeeCalculator createFeeCalculator() {
        return new IndiaFeeCalculator();
    }
}

@Component("USA")
public class UsPaymentFactory implements PaymentFactory {

    public PaymentProcessor createProcessor() {
        return new UsPaymentProcessor();
    }

    public PaymentValidator createValidator() {
        return new UsValidator();
    }

    public FeeCalculator createFeeCalculator() {
        return new UsFeeCalculator();
    }
}

// Step 6: Usage
@Service
public class PaymentService {

    private final Map&#x3C;String, PaymentFactory> factoryMap;

    public PaymentService(Map&#x3C;String, PaymentFactory> factoryMap) {
        this.factoryMap = factoryMap;
    }

    public void process(String country) {
        PaymentFactory factory = factoryMap.get(country);

        PaymentProcessor processor = factory.createProcessor();
        PaymentValidator validator = factory.createValidator();
        FeeCalculator feeCalculator = factory.createFeeCalculator();

        validator.validate();
        processor.process();
        System.out.println(feeCalculator.calculateFee());
    }
}
</code></pre>

Factory vs Abstract Factory&#x20;

| Feature | Factory          | Abstract Factory            |
| ------- | ---------------- | --------------------------- |
| Creates | One object       | Multiple related objects    |
| Example | PaymentProcessor | Processor + Validator + Fee |
| Scope   | Single product   | Product family              |

### Builder Pattern

Builder pattern is used to construct complex objects step-by-step, especially when there are many optional parameters. In our payment system, we used Builder for creating request objects to ensure immutability, readability, and validation before object creation.

👉 In simple terms:

> “Build complex objects step-by-step instead of using huge constructors.”

👉 Builder is commonly used to create immutable objects since fields are `final`

Problem It Solves ?

❌ Telescoping Constructor Problem

```
new PaymentRequest("A", "B", 1000, "INR", "UPI", null, null, true);
```

Issues:

* Hard to read
* Order matters (bug-prone)
* Optional fields messy
* Not maintainable

✅ Builder Solution

```
PaymentRequest request = PaymentRequest.builder()
    .fromAccount("A")
    .toAccount("B")
    .amount(1000)
    .currency("INR")
    .paymentType("UPI")
    .priority(true)
    .build();
```

👉 Clean, readable, safe

Structure

```
Product (PaymentRequest)
   ↑
Builder (inner/static class)
   ↑
build() → final object
```

Example

{% code overflow="wrap" %}
```
public class PaymentRequest {

    private final String fromAccount;
    private final String toAccount;
    private final double amount;
    private final String currency;
    private final String paymentType;
    private final boolean priority;

    private PaymentRequest(Builder builder) {
        this.fromAccount = builder.fromAccount;
        this.toAccount = builder.toAccount;
        this.amount = builder.amount;
        this.currency = builder.currency;
        this.paymentType = builder.paymentType;
        this.priority = builder.priority;
    }

    public static class Builder {
        private String fromAccount;
        private String toAccount;
        private double amount;
        private String currency;
        private String paymentType;
        private boolean priority;

        public Builder fromAccount(String fromAccount) {
            this.fromAccount = fromAccount;
            return this;
        }

        public Builder toAccount(String toAccount) {
            this.toAccount = toAccount;
            return this;
        }

        public Builder amount(double amount) {
            this.amount = amount;
            return this;
        }

        public Builder currency(String currency) {
            this.currency = currency;
            return this;
        }

        public Builder paymentType(String paymentType) {
            this.paymentType = paymentType;
            return this;
        }

        public Builder priority(boolean priority) {
            this.priority = priority;
            return this;
        }

        public PaymentRequest build() {
            return new PaymentRequest(this);
        }
    }

    public static Builder builder() {
        return new Builder();
    }
}
```
{% endcode %}



Builder vs Constructor

| Constructor              | Builder      |
| ------------------------ | ------------ |
| Hard to read             | Readable     |
| Order matters            | Named fields |
| Poor for optional fields | Excellent    |

Builder vs Factory

| Builder                            | Factory                        |
| ---------------------------------- | ------------------------------ |
| Builds complex object step-by-step | Decides which object to create |
| Same class                         | Multiple classes               |

### Prototype Pattern

Create new objects by **copying (cloning)** an existing object instead of creating from scratch.

👉 In simple terms:

> “Duplicate an existing object instead of building a new one.”

Problem It Solves ?

❌ Expensive Object Creation

Imagine:

* DB-heavy object initialization
* complex config loading
* large nested objects

```
new PaymentConfig(); // loads rules, limits, fees, etc.
```

👉 Creating repeatedly = costly

✅ Solution: Clone Existing Object

```
PaymentConfig copy = original.clone();
```

👉 Fast + efficient
