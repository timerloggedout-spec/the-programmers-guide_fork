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

