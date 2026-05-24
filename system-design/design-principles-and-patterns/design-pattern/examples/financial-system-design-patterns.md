# Financial System Design Patterns

## About

This page is a consolidated reference for understanding and revisiting software design patterns through real-world financial system examples. The focus is not only on theoretical definitions, but also on how these patterns are practically applied in enterprise backend systems such as payment platforms, banking applications, and microservices architectures.

## Creational

### Singleton

Ensure a class has **only one instance** and provide a **global access point**.

Without Singleton:

* Multiple DB connection managers → inconsistent state
* Multiple cache instances → memory issues
* Multiple config loaders → bugs



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

_“Where did you use Singleton in your project?”_

> "In Spring Boot, most services are Singleton by default. For example, we used a singleton cache for exchange rates and feature configurations to ensure consistency across all payment flows and avoid repeated external API calls."

How to Make Singleton Thread-Safe?

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



