# Java OOP Concepts — A Guided Tour with One Running Example

This README walks through core Java concepts, all built around a single, growing example: a small **vehicle rental system**. Each section adds a new idea on top of the previous code, so by the end you'll have seen how these concepts fit together in a real class hierarchy — finishing with lambda expressions.

---

## Table of Contents

1. [The `final` Keyword](#1-the-final-keyword)
2. [Objects (OOP Basics)](#2-objects-oop-basics)
3. [Constructors](#3-constructors)
4. [Variable Scope](#4-variable-scope)
5. [Overloaded Constructors](#5-overloaded-constructors)
6. [The `toString()` Method](#6-the-tostring-method)
7. [Arrays of Objects](#7-arrays-of-objects)
8. [Object Passing](#8-object-passing)
9. [The `static` Keyword](#9-the-static-keyword)
10. [Inheritance](#10-inheritance)
11. [Method Overriding](#11-method-overriding)
12. [The `super` Keyword](#12-the-super-keyword)
13. [Abstraction](#13-abstraction)
14. [Access Modifiers](#14-access-modifiers)
15. [Encapsulation](#15-encapsulation)
16. [Copying Objects](#16-copying-objects)
17. [Interfaces](#17-interfaces)
18. [Polymorphism](#18-polymorphism)
19. [Dynamic Polymorphism](#19-dynamic-polymorphism)
20. [Lambda Expressions](#20-lambda-expressions)

---

## 1. The `final` Keyword

`final` marks something as unchangeable once set:
- a **final variable** can't be reassigned after initialization
- a **final method** can't be overridden by a subclass
- a **final class** can't be subclassed at all

```java
public class Vehicle {
    // A constant — shared rule for every vehicle, never changes
    public static final int MAX_PASSENGERS = 8;
}
```

We'll use `MAX_PASSENGERS` later as a fixed business rule.

---

## 2. Objects (OOP Basics)

Object-Oriented Programming organizes code around **objects** — bundles of *state* (fields) and *behavior* (methods) modeled from a **class** (the blueprint).

```java
public class Vehicle {
    String make;
    String model;
    int year;
}
```

```java
Vehicle car = new Vehicle();   // an object — an instance of the Vehicle class
car.make = "Toyota";
car.model = "Corolla";
car.year = 2022;
```

Each `Vehicle` object has its own copy of `make`, `model`, and `year` — that's what makes objects distinct from one another even though they share a blueprint.

---

## 3. Constructors

A **constructor** is a special method, matching the class name, that runs when an object is created — used to set up initial state instead of assigning fields one by one.

```java
public class Vehicle {
    String make;
    String model;
    int year;

    // Constructor
    public Vehicle(String make, String model, int year) {
        this.make = make;
        this.model = model;
        this.year = year;
    }
}
```

```java
Vehicle car = new Vehicle("Toyota", "Corolla", 2022);
```

`this.make` refers to the object's field; `make` (the parameter) is a local variable — `this` disambiguates between them.

---

## 4. Variable Scope

**Scope** determines where a variable is visible and how long it lives:

| Type | Declared | Lives in |
|---|---|---|
| Instance variable | inside the class, outside any method | as long as the object exists |
| Local variable | inside a method/constructor | only during that method call |
| Static variable | with `static`, inside the class | as long as the class is loaded (shared across all objects) |

```java
public class Vehicle {
    String make;                     // instance variable — one per object

    public void describe() {
        String summary = make + " vehicle"; // local variable — exists only inside describe()
        System.out.println(summary);
    }
}
```

`summary` disappears once `describe()` finishes; `make` sticks around for the object's whole life.

---

## 5. Overloaded Constructors

A class can have **multiple constructors** with different parameter lists — this is **constructor overloading**, giving callers flexible ways to build an object.

```java
public class Vehicle {
    String make;
    String model;
    int year;

    public Vehicle(String make, String model, int year) {
        this.make = make;
        this.model = model;
        this.year = year;
    }

    // Overloaded constructor — defaults the year to the current year
    public Vehicle(String make, String model) {
        this(make, model, 2026);   // calls the constructor above
    }

    // Overloaded constructor — a totally unspecified vehicle
    public Vehicle() {
        this("Unknown", "Unknown", 2026);
    }
}
```

```java
Vehicle v1 = new Vehicle("Honda", "Civic", 2021);
Vehicle v2 = new Vehicle("Ford", "Focus");   // year defaults to 2026
Vehicle v3 = new Vehicle();                  // fully default
```

---

## 6. The `toString()` Method

Every Java object inherits a `toString()` method from `Object`, but its default output (like `Vehicle@1b6d3586`) isn't useful. **Overriding** it lets you control how an object prints.

```java
@Override
public String toString() {
    return year + " " + make + " " + model;
}
```

```java
System.out.println(v1);   // prints: 2021 Honda Civic  (instead of a memory address)
```

---

## 7. Array of Objects

Just like arrays of primitives, you can have an array where each slot holds an **object reference**.

```java
Vehicle[] fleet = new Vehicle[3];
fleet[0] = new Vehicle("Toyota", "Corolla", 2022);
fleet[1] = new Vehicle("Honda", "Civic", 2021);
fleet[2] = new Vehicle("Tesla", "Model 3", 2023);

for (Vehicle v : fleet) {
    System.out.println(v);   // uses our toString() automatically
}
```

This is how a real "fleet" or "inventory" is usually modeled — a collection of objects, not a collection of raw values.

---

## 8. Object Passing

In Java, object variables hold a **reference** to the object, not the object itself. When you pass an object to a method, you pass a copy of that reference — both the caller and the method point to the *same* object, so changes to its fields are visible outside the method.

```java
public static void repaint(Vehicle v, String newModelSuffix) {
    v.model = v.model + " " + newModelSuffix;   // mutates the shared object
}
```

```java
Vehicle car = new Vehicle("Toyota", "Corolla", 2022);
repaint(car, "SE");
System.out.println(car.model);   // "Corolla SE" — the original object changed
```

Note: reassigning the parameter itself (`v = new Vehicle(...)`) inside the method would *not* affect the caller's variable — only field mutations on the shared object are visible.

---

## 9. The `static` Keyword

`static` members belong to the **class itself**, not to any one object — every object shares the same copy.

```java
public class Vehicle {
    static int totalVehiclesCreated = 0;   // shared counter

    public Vehicle(String make, String model, int year) {
        this.make = make;
        this.model = model;
        this.year = year;
        totalVehiclesCreated++;            // increments the shared counter
    }
}
```

```java
new Vehicle("Kia", "Rio", 2020);
new Vehicle("Mazda", "3", 2019);
System.out.println(Vehicle.totalVehiclesCreated);   // 2 — accessed via the class, not an instance
```

---

## 10. Inheritance

Inheritance lets one class (**subclass**) reuse and extend fields/methods from another (**superclass**), modeling an "is-a" relationship.

```java
public class Car extends Vehicle {
    int numberOfDoors;

    public Car(String make, String model, int year, int numberOfDoors) {
        super(make, model, year);   // build the Vehicle part first
        this.numberOfDoors = numberOfDoors;
    }
}
```

`Car` automatically has `make`, `model`, `year`, and every `Vehicle` method — plus its own `numberOfDoors`.

---

## 11. Method Overriding

A subclass can **override** a superclass method to provide its own version, as long as the signature matches.

```java
public class Car extends Vehicle {
    // ... fields/constructor from above ...

    @Override
    public String toString() {
        return super.toString() + " (" + numberOfDoors + "-door)";
    }
}
```

```java
Car myCar = new Car("Toyota", "Corolla", 2022, 4);
System.out.println(myCar);   // "2022 Toyota Corolla (4-door)"
```

---

## 12. The `super` Keyword

`super` refers to the immediate **superclass**. It's used to:
- call the superclass constructor: `super(make, model, year);`
- call a superclass method that's been overridden: `super.toString()`
- access a superclass field, if not hidden by a subclass field of the same name

You already saw both uses above — `super(...)` in `Car`'s constructor, and `super.toString()` inside the overridden `toString()`.

---

## 13. Abstraction

Abstraction means exposing *what* something does while hiding *how*. In Java, an **abstract class** can declare methods with no body (`abstract` methods), forcing subclasses to provide the implementation.

```java
public abstract class Vehicle {
    String make;
    String model;
    int year;

    public Vehicle(String make, String model, int year) {
        this.make = make;
        this.model = model;
        this.year = year;
    }

    // No body — every concrete vehicle type must define its own rental cost
    public abstract double calculateDailyRentalCost();

    @Override
    public String toString() {
        return year + " " + make + " " + model;
    }
}
```

You can no longer write `new Vehicle(...)` directly — only concrete subclasses like `Car` can be instantiated, and they *must* implement `calculateDailyRentalCost()`.

---

## 14. Access Modifiers

Access modifiers control **visibility** of classes, fields, and methods:

| Modifier | Same class | Same package | Subclass (other package) | Everywhere |
|---|:---:|:---:|:---:|:---:|
| `private` | ✅ | ❌ | ❌ | ❌ |
| *(default/package)* | ✅ | ✅ | ❌ | ❌ |
| `protected` | ✅ | ✅ | ✅ | ❌ |
| `public` | ✅ | ✅ | ✅ | ✅ |

```java
public class Vehicle {
    private String make;     // only Vehicle itself can touch this directly
    protected int year;      // Vehicle and its subclasses can touch this
    public String model;     // anyone can touch this
}
```

---

## 15. Encapsulation

Encapsulation combines access modifiers with **getters/setters** to protect an object's internal state — fields stay `private`, and controlled access goes through public methods, which can validate input.

```java
public class Vehicle {
    private int year;

    public int getYear() {
        return year;
    }

    public void setYear(int year) {
        if (year < 1886) {   // the first automobile was built in 1886
            throw new IllegalArgumentException("Year is not valid");
        }
        this.year = year;
    }
}
```

```java
car.setYear(1800);   // throws IllegalArgumentException — invalid state is rejected
```

Without encapsulation, `car.year = 1800;` would have silently corrupted the object.

---

## 16. Copying Objects

Because object variables are references (see [Object Passing](#8-object-passing)), `Vehicle copy = original;` doesn't create a new object — it just gives you a second reference to the same one. To truly copy, you need a **copy constructor** or `clone()`.

```java
public class Vehicle {
    String make, model;
    int year;

    // Copy constructor — builds a new, independent object from an existing one
    public Vehicle(Vehicle other) {
        this.make = other.make;
        this.model = other.model;
        this.year = other.year;
    }
}
```

```java
Vehicle original = new Vehicle("Toyota", "Corolla", 2022);
Vehicle copy = new Vehicle(original);   // a separate object with the same data

copy.model = "Camry";
System.out.println(original.model);   // still "Corolla" — unaffected
```

---

## 17. Interfaces

An **interface** defines a contract — a set of methods a class promises to implement — without any implementation details of its own (fields in interfaces are implicitly `public static final`). Unlike abstract classes, a class can implement *multiple* interfaces.

```java
public interface Rentable {
    double calculateDailyRentalCost();
    boolean isAvailable();
}

public interface Drivable {
    void drive();
}
```

```java
public class Car extends Vehicle implements Rentable, Drivable {
    private boolean available = true;

    @Override
    public double calculateDailyRentalCost() {
        return 45.0;
    }

    @Override
    public boolean isAvailable() {
        return available;
    }

    @Override
    public void drive() {
        System.out.println(model + " is driving.");
    }
}
```

---

## 18. Polymorphism

Polymorphism ("many forms") means the same operation behaves differently depending on the object it's acting on. Method overloading (multiple methods, same name, different parameters) and overriding (see [§11](#11-method-overriding)) are both forms of it.

```java
public class RentalOffice {
    public double quote(Car car) {
        return car.calculateDailyRentalCost();
    }

    // Overloaded: same method name, different parameter type
    public double quote(Car car, int days) {
        return car.calculateDailyRentalCost() * days;
    }
}
```

The compiler picks the right `quote(...)` version based on the arguments — that's **compile-time (static) polymorphism**.

---

## 19. Dynamic Polymorphism

**Dynamic (runtime) polymorphism** happens when you refer to an object through a *superclass or interface* type, but the *actual* object's overridden method runs — decided at runtime, not compile time.

```java
public class ElectricCar extends Car {
    @Override
    public double calculateDailyRentalCost() {
        return 60.0;   // electric cars rent for more
    }
}
```

```java
Rentable[] fleet = {
    new Car("Toyota", "Corolla", 2022, 4),
    new ElectricCar("Tesla", "Model 3", 2023, 4)
};

for (Rentable r : fleet) {
    // The same call, r.calculateDailyRentalCost(), runs a DIFFERENT
    // method body depending on the actual object's runtime type.
    System.out.println(r.calculateDailyRentalCost());
}
// Output: 45.0
//         60.0
```

Each element is declared as `Rentable`, but Java looks at the *real* object at runtime to decide which `calculateDailyRentalCost()` to run. This is the mechanism behind flexible, extensible designs — you can add `HybridCar`, `Truck`, etc., and existing code that loops over `Rentable[]` keeps working unchanged.

---

## 20. Lambda Expressions

Everything above builds toward this: a **lambda expression** is a compact way to write an implementation of a **functional interface** (an interface with exactly one abstract method) inline, without writing a full named class.

### The old way (anonymous class)

Suppose we want to sort our fleet by rental cost, using `Comparator` (a functional interface with one method, `compare`):

```java
List<Rentable> fleet = List.of(
    new Car("Toyota", "Corolla", 2022, 4),
    new ElectricCar("Tesla", "Model 3", 2023, 4)
);

Collections.sort(fleet, new Comparator<Rentable>() {
    @Override
    public int compare(Rentable a, Rentable b) {
        return Double.compare(a.calculateDailyRentalCost(), b.calculateDailyRentalCost());
    }
});
```

### The same thing, as a lambda

```java
Collections.sort(fleet, (a, b) ->
    Double.compare(a.calculateDailyRentalCost(), b.calculateDailyRentalCost())
);
```

The lambda `(a, b) -> Double.compare(...)` *is* the implementation of `compare` — Java infers which interface and method it's implementing from context, so all the boilerplate (`new Comparator<Rentable>() { ... }`) disappears.

### Lambdas with our own functional interface

You can define your own functional interfaces and use lambdas the same way:

```java
@FunctionalInterface
public interface RentalDiscount {
    double apply(double baseCost);
}
```

```java
// A 10% discount, expressed as a lambda
RentalDiscount tenPercentOff = baseCost -> baseCost * 0.9;

double finalPrice = tenPercentOff.apply(myCar.calculateDailyRentalCost());
System.out.println(finalPrice);   // 40.5
```

### Lambdas + streams, tying it together

```java
double averageCost = fleet.stream()
    .mapToDouble(Rentable::calculateDailyRentalCost)  // method reference, a lambda shorthand
    .average()
    .orElse(0);

List<String> availableModels = fleet.stream()
    .filter(Rentable::isAvailable)
    .map(Object::toString)
    .toList();
```

**Why it matters:** lambdas rely on everything covered above — they're only possible *because* Java has interfaces with a single abstract method to implement, and they lean on polymorphism (a lambda passed as a `Comparator` is used polymorphically, just like `Rentable` was in [§19](#19-dynamic-polymorphism)). They don't replace OOP — they're a concise notation for a very common OOP pattern: implementing a single-method interface on the fly.

---

## Summary

| Concept | What it solves |
|---|---|
| `final`, `static` | Constants and class-wide (shared) state |
| Objects, constructors, overloaded constructors | Creating and initializing instances flexibly |
| Variable scope | Controlling where data lives and is visible |
| `toString()` | Human-readable object output |
| Arrays of objects, object passing | Working with collections and references |
| Inheritance, `super`, method overriding | Reusing and specializing behavior |
| Abstraction, interfaces | Defining contracts without dictating implementation |
| Access modifiers, encapsulation | Protecting and validating internal state |
| Copying objects | Avoiding accidental shared-reference bugs |
| Polymorphism, dynamic polymorphism | One interface, many runtime behaviors |
| Lambda expressions | Concise implementations of single-method interfaces |