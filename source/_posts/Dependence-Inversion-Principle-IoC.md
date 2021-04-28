---
title: Dependence Inversion Principle (DIP, DI, IoC)
date: 2019-01-20 13:50:07
tags: DIP
categories: Design Pattern
---

For the software design, there are five principles called S.O.L.I.D, which stands for

1. Single Responsibility Principle
2. Open-closed Principle
3. The Liskov Substitution Principle
4. Interface Segregation Principle
5. Dependency Inversion Principle

In Object-Oriented programming, those principles play an important roles and give us the guideline to design loosely coupled classes. Along with dependency inversion principle(DIP), there are some terms called Inversion of Control(IoC) and Dependency Injection(DI). These are very abstract concepts.

### Dependence Inversion Principle (DIP)

First, let's see the definition of the DIP:
> 1.High-level modules should not depend on low-level modules. Both should depend on abstractions.
> 2.Abstractions should not depend upon details. Details should depend upon abstractions.

It is really a abstract concept. Let's first figure out what is dependency. The dependency is kind of a requirment. For example, a person needs a fork to eat that is the penson depends on the fork. When we coding, we can write like this

```java
class Fork {
    public void use() {
        System.out.println("Using fork to eat");
    }
}

class Person {
    Fork fork;
    public Person() {
        fork = new Fork();
    }
    public void eat() {
        fork.use();
    }
}

class Test {
    public static void main(String[] args) {
        Person person = new Person();
        person.eat();
    }
}
```

_Person_ holds a _Fork_ variable. What if we want to use a spoon to eat, we probably will write code like following

```java
class Spoon {
    public void use() {
        System.out.println("Using fork to eat");
    }
}

class Person {
    // Fork fork;
    Spoon spoon;
    public Person() {
        // fork = new Fork();
        spoon = new Spoon();
    }
    public void eat() {
        // fork.use();
        spoon.use();
    }
}
```
You can see that we need to change lots of places to let this person to use spoon to eat. If we need to use chopsticks, we also need to change chunk of codes. Here is the dependency hierarchy.
```
person  ->      spoon
        ->      fork
        ->      chopsticks
```
This hierarchy violates the first point of DIP

>High-level modules should not depend on low-level modules. Both should depend on abstractions.

Let's take a look of these cases. The person trys to call the _use_ function of these tools, which is the only care. Person does not need to care about how the _use_ function implemented that is the job of the tool class. So this _use_ function is the abstract behavior of the tools, while what exactly the tool is should be the details. Here comes to the second point.

>Abstractions should not depend upon details. Details should depend upon abstractions.

Depend on these two points. We use an interface for _use_ function and all tools should implemented this function.
```java
public interface Tool {
    public void use()
}

class Fork implements Tool {
    @Override
    public void use() {
        System.out.println("Using fork to eat");
    }
}

class Spoon implements Tool {
    @Override
    public void use() {
        System.out.println("Using spoon to eat");
    }
}

class Person {
    private Tool tool;
    public Person() {
        tool = new Spoon();
    }
    public void eat() {
        tool.use();
    }
}
```
Now the _Person_ does not depend on the _Fork_ or other low-level detail while it depends on the _Tool_ interface. Both high-level and low-level modules depend on interface. It looses coupling between modules. The hierarchy should be like this.
```
person  ->  tool        <-  spoon
            (abstract)  <-  fork
                        <-  chopsticks
```
Next the person wants to use other tool to eat, we can simple implemented this interface. This is detail depends on abstraction.

### Inversion of Control (IoC)

The _Person_ class still has the control to instantiate the tool. So everytime we need to change a tool to eat, we have to change the code inside the _Person_ constractor. For the Inversion of Control, _Person_ should move the instantiation of the tool outside its class.

```java
class Person {
    private Tool tool;
        public Person(Tool tool) {
        this.tool = tool;
    }
    public void eat() {
        tool.use();
    }
}

class Test {
    public static void main(String[] args) {
        Person person = new Person(new Fork());
        person.eat();
    }
}
```

Now this _Person_ class will not need to be modified whenever the tools changed (It does not care). It gives the authority to the _Test_ which is the container. This is the IoC. Let's change the _Person_ class a little bit.

```java
class Person {
    protected Tool tool;
    public Person(Tool tool) {
        this.tool = tool;
    }
    abstract void eat();
}

class Boy extends Person {
    public Boy(Tool tool) {
        super(tool);
    }
    void eat() {
        System.out.print("I am a boy and I'm ");
        tool.use();
    }
}

class Girl extends Person {
    public Girl(Tool tool) {
        super(tool);
    }
    void eat() {
        System.out.print("I am a girl and I'm ");
        tool.use();
    }
}

class Test {
    public static void main(String[] args) {
        Person person1 = new Boy(new Fork());
        Person person2 = new Girl(new Spoon());
        person1.eat();
        person2.eat()
    }
}
```
_Person_ and _Tool_ are loosely coupled and do not depend on each other. Any person can use any tool to eat. In Java Spring/Dagger, we set up the xml or use java refection to instantiate/inject (usually by annotation) in the IoC container.

### Dependence Injection (DI)

IoC is our purpose and dependence injection is how we do it. In the code above we used contractor injection. If we use a setter, it is called setter injection.
```java
class Person {
    private Tool tool;
        public Person() {
    }
    public void eat() {
        tool.use();
    }
    public void setter(Tool tool) {
        this.tool = tool;
    }
}
```
Also we can also use interface injection. It is very similar with setter. With interface injection, we can only inject the tool to the person want to eat.

Inversion of Control is the design pattern followed the Denpendency Inversion Principle and the Dependency Injection is the way to do it.

