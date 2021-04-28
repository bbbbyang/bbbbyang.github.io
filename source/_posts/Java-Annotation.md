---
title: Java Annotation
date: 2019-05-09 23:05:49
tags: Annotation
categories: Java
---

Recently, I am learning Spring framework. Spring framework is  based on annotation-driven development. I used to use annotation not often, just like _override_. For better understanding spring framework, I think I need to understand annotation development first.

Annotation is a note for the function or varibale. If you search on Google about annotation, there are many tutorials about the definition of annotation in Java. Easy words, hard to understand how the annotation works. Here I will explain annotation development with an example in Android development of [NetworkListener][1] ( switch to _Annotation-Network-Listener_ branch).

In our application, we want to catch the changes in network, network switching between wifi and mobile, or between connected and disconnected. The first thing to do is to define an enum type NetType

### Define an Annotation

```java
public enum NetType {
    // Connected With Wifi/GPRS
    AUTO,
    WIFI,
    MOBILE,
    // No Internet
    NONE
}
```
There are 4 states in network, WIFI, MOBILE and NONE are very straightforward. Why we have this AUTO? NetType is not only used for network type for the network connection, also used for which network type changes you want to listen. So AUTO means to listen all types of network change.

Next we need to define the annotation.
```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Network {
    NetType netType() default NetType.AUTO;
}
```
There are two annotations for our annotation. This is called meta-annotation. Meta-annotation is like a key word in Java, such as int, string, enum. It is how you define your annotation.

- _@Target(ElementType.METHOD)_ : our annotation should be defined on methods.
- _@Retention(RetentionPolicy.RUNTIME)_ : annotations are to be recorded in the class file by the compiler and retained by the VM at run time, so they may be read reflectively.

I am not going to explain all the meta-annotation here. From the definition, it uses _@interface_ to define an annotation, so we can see that annotation is kind of interface, which extends `java.lang.annotation.Annotation`. `netType()` is called attribute in annotation, while you can give it a default value. Java will generate an instance that implements this interface by dynamic proxy and then assigns the value to the attribute.

### Use the Annotation
After we defined the annotation _Network_, now we need to use it. In our application, the _MainActivity_ want to listen to the network changes. So we use annotation like this
```java
@Network(netType = NetType.AUTO)
public void network(NetType netType) {
    switch (netType) {
        case WIFI:
            Log.e(Constants.LOG_TAG, "WIFI");
            break;
        case MOBILE:
            Log.e(Constants.LOG_TAG, "MOBILE");
            break;
        case NONE:
            Log.e(Constants.LOG_TAG, "Network Disconnected");
            break;
    }
}
```
There are two _netType_ here.
- `public void network(NetType netType)` _netType_ is what kind of network connection type is for the device.
- `@Network(netType = NetType.AUTO)` _netType_ is what kind of network changes you want to listen. If you change it to NetType.WIFI which means this function only listeners to WIFI changes.

### Track the Annotation

For network changes, we will define a broadcast receiver to catch the network changed broadcasting. Also Receiver needs to know which function in which activity want to know about this. First we need to pass the _MainActivity_ to the receiver. We have a registerObserver function to handle this.

```java
public void registerObserver(Object register) {
    List<MethodManager> methodList = networkList.get(register);
    if( methodList == null ) {
        methodList = findAnnotationMethod(register);
        networkList.put(register, methodList);
        Log.e(Constants.LOG_TAG, register.getClass().getName() + " Registered");
    }
}
```
`networkList` is a map which key is the actiivty and value is a list of methods that has the annotation. `findAnnotationMethod` is the function to get all satisfied function that has the _network_ annotation.

```java
private List<MethodManager> findAnnotationMethod( Object register ) {
    List<MethodManager> methodList = new ArrayList<>();
    Class<?> clazz = register.getClass();
    Method[] methods = clazz.getMethods();
    for( Method method : methods ) {
        Network network = method.getAnnotation(Network.class);
        if( network == null ) {
            continue;
        }
        Type returnType = method.getGenericReturnType();
        if( !"void".equals( returnType.toString() ) ) {
            throw new RuntimeException( method.getName() + " return type should be void" );
        }
        Class<?>[] parameterTypes = method.getParameterTypes();
        if( parameterTypes.length != 1 ) {
            throw new RuntimeException( method.getName() + " should have only one parameter" );
        }
        MethodManager methodManager = new MethodManager( parameterTypes[0], network.netType(), method );
        methodList.add(methodManager);
    }
    return methodList;
}
```
This is the most improtant part in our application of annotation. We use Java refection to do this.
1. Get all function in that activity/class and filter the function has the _network_ annotation.
2. Filter the voide return type function
3. Filter the one parameter function
4. Store the function in the list. ParameterType(netType in the parameter, current network connection type), network.netType(netType in our annotation, what kind of network changes we want to catch), method

Now after we registered all the satisfied methods in our application, how to trigger it? In receiver, when the network changes, the receiver should receive the broadcasting message. Then trigger a _post_ function with the current network connection type
```java
private void post(NetType netType) {
    Set<Object> set = networkList.keySet();
    for ( final Object getter : set ) {
        List<MethodManager> methodeList = networkList.get(getter);
        if( methodeList != null ) {
            for( final MethodManager method : methodeList ) {
                if( method.getType().isAssignableFrom(netType.getClass() ) ) {
                    switch ( method.getNetType() ) {
                        case AUTO:
                            invoke( method, getter, netType );
                            break;
                        case WIFI:
                            if( netType == NetType.WIFI || netType == NetType.NONE ) {
                                invoke( method, getter, netType );
                            }
                            break;
                        case MOBILE:
                            if( netType == NetType.MOBILE || netType == NetType.NONE ) {
                                invoke( method, getter, netType );
                            }
                            break;
                        default:
                            break;
                    }
                }
            }
        }
    }
}
```
We give the _network_ attribute _netType_ as NetType.AUTO. so `method.getNetType()` will give what kind of network changes we want to catch. In our case, it is AUTO, so it will invoke the function with the annotation as long as the network changes. If it WIFI, it will only invoke the function when WIFI changes. This is how we use annotation. If you are interested in the source code, you can check my [NetworkListener][1] repo with Annnotation-Network-Listener branch.


[1]:https://github.com/bbbbyang/NetworkListener/tree/Annotation-Network-Listener