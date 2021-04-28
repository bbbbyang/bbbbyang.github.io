---
title: Java Dynamic Proxy and AOP
date: 2019-07-10 20:38:12
tags: AOP
categories: Design Pattern
---
AOP and IOC are the two important concepts in Spring developement. AOP is Aspect Oriented Programming. I will start with proxy design pattern.

When we play online game, we can pay for someone else to help us to play to get experience or money. This _someone else_ is a proxy-like role. Or more likely, vendors do not want to sell clothes or shoes directly with customers, they will ask a proxy, more like a store, to communicate with customers. Do something like, deal with invoice or a return. Proxy is like a middle man stuff. There are two good points for proxy pattern.

- Proxy can hide the original object. For a store, customer will not contact vendor.
- Proxy can decouple the original object with the clients, so proxy can do more stuff, like heal with the invoice or a return in the example.

#### Static Proxy
If the proxy class exists already before run time. It is called static proxy. We define an interface called `Product`, it has two functions _sell_ and _service_.
```java
public interface Product() {
    void service();
    void sell();
}
```
For a vendor, which is considered as a real subject, implements this product interface.
```java
public class Vendor implements Product {
    @Override
    public void service() {
        System.out.println("product service");
    }
    @Override
    public void sell() {
        System.out.println("product sell") ;
    }
}
```
Customers are so annoying and the vendor does not want to deal with the sell/service process with customers. He wants to ask a proxy for help. This proxy agent can decouple real subject and client.
```java
public ProxyAgent implements Product {
    Product product = new Vendor(); // We usually use setter DI to inject
    @Override
    public void service() {
        System.out.println("Proxy service");
        product.service();
    }
    @Override
    public void sell() {
        System.out.println("Proxy sell") ;
        product.sell();
    }
}
```
Client calls the `ProxyAgent` to ask for a service or sell.
```java
public class Client {
    Product product = new ProxyAgent();
    product.service();
    product.sell();
}
```
So now, if the process of service or sell need to be changed, you can just fix the functions inside ProxyAgent. Like this product is only for student, you can simply do this
```java
public ProxyAgent implements Product {
    @Override
    public void sell() {
        System.out.println("Proxy sell") ;
        if( isStudent ) {
            vendor.sell();
        }
    }
}
```
Now you can see why we want this proxy. Proxy class can decouple the real subject with the client. But we need to define the proxy class before running the code. However, if we want to use this pattern, we need to define a proxy class for every class.

#### Dynamic Proxy
There are so many classes in our code and we definitely do not want to create proxy class for every single class. How can we do not create a proxy class but get the feature of it?

When we create an instance, the process should be like
```
Vendor.java  -->  Vendor.class  -->  Class<Vendor>  --> vendor object
```
So the most important part is to get `Class Object` and use it to create an instance. We use java refection to get this class object during run time. Interface is the common info shared by proxy and real subject. However, interface can not create an instance. JDK offers us with java.lang.reflect.InvocationHandler interface and java.lang.reflect.Proxy class.

Proxy.getProxyClass or Proxy.getNewInstance can dynamic create an proxy class instance for the real subject. We can write it this way

```java
public class DynamicProxy implements InvocationHandler { 
    private Object obj; 
    public Object getNewInstance(Object obj) {
        this.obj = obj;
        return Proxy.newProxyInstance(object.getClass().getClassLoader(), object.getClass().getInterfaces(), this);
    } 
 
    @Override 
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable { 
        System.out.println("before"); 
        Object result = method.invoke(obj, args); 
        System.out.println("after"); 
        return result; 
    }
} 
```
We pass the real subject into this dynamic proxy, it will get the classloader and interface info. In the invoke function, we can enhance the code in the way we want.

```java
public class DynamicProxyDemo {
    public static void main(String[] args){
        DynamicProxy proxySubject = new DynamicProxy();
        Product product = (Product)proxySubject.getNewInstance( new Vendor() );
        product.sell();
    }
}
```
Here, the _product_ is an instance of `Proxy` class which implements `product` interface.

- `Product` is an interface which does not have constructor and can not generate instance.
- Helped by `Proxy` class, it can disassemble the info of `Product` class and generate an proxy object instance
- Handler.invoke proxy the real subject function.

Proxy class object will generate a proxy object which implements interface. The essential of proxy object is that it implements the interface which is the same implemented in real subject.








