---
title: C++ Polymorphism
date: 2020-05-17 11:45:31
tag: Polymorphism
categories: C++
---
When I first time using C++ in my work, I was wondering why is the parent class desctructor need to be virtual. It is something related to polymorphism. About polymorphism, we know there are three principles:
1. inheritance
2. override
3. upcast, father pointer/reference pointing to a child object

You certainly can find lots of examples in Java, expecially used by the Java interface. Here I want to talk about the polymorphism in C++.

#### Static and Dynamic Polymorphism

The essential concept of polymorphism is one interface but multiple forms. There are two types polymorphism, static polymorphism and dynamic polymorphism. The difference is when to bind the function address.

>In the compile time, we can confirm the function called by which object. For example, overload and generic are static polymorphism. 

>When we using a virtual function, it can only bind the function address in the run time, which is a dynamic polymorphism. In this blog, I will discuss more on dynamic binding and virtual function in cpp.

#### C++ object model
For C, data and function are separate. It does not support the relation between these two. C++ use abstact data type ADT to bind data and function. C++ has two types of data member, static and non-static and it has three types of function member, static, nonstatic and virtual.

we can see a C++ object model here. Non-static variable members will be put in a data member table. For virtual function, C++ object model has a vptr in data member table which will point to a virtual table. The first slot of the virtual table is virtual type object. 

pic here

#### Polymorphism
Here is how C++ support polymorphism.
1. Base class pointer/reference pointing to a derived class
2. Function need to be virtual.

Here is a simple example (we don't consider destructor here).
```c++
class Animal
{
    public:
    std::string name;
    void sleep(){cout<<"animal sleep"<<endl;}
    void breathe(){cout<<"animal breathe"<<endl;}
};
class Fish:public animal
{
    public:
    void breathe(){cout<<"fish bubble"<<endl;}
};
int main()
{
    Fish fh;
    Animal *pAn = &fh;
    Fish *pFh = &fh;
    animal ani = fh;
    ani.breathe();
    pAn->breathe();
    pFh->breathe();
}

1000:|---------------------|                      --|
     |        Animal       |  --->  Animal memory   |
100x |---------------------|                        | -->Fish memory
     | Fish addtional part |                        |
     |---------------------|                      --|
```
The result is
```
animal breathe
animal breathe
fish bubble
```
So for the first _ani_ variable, it will cause an object slicing. It is easy to explain. `animal ani = fh;` It will call _animal_ copy constructor and assign fish object to an animal object. Animal object won't has any fish addtional infomation, that is object slicing.

We have two pointers, _pAn_ and _pfh_ pointing to the same address, the first byte of the _Fish_ object. However, there is an important difference between these two pointers. _pAn_ only contains the _Animal_ memory and _pFh_ has the whole fish object memory. This is why we have _animal_ porinter pointing to _fish_ but it still do not have polymorphism.

Let's make the _breathe_ as virtual function. _main_ function will be same.
```c++
class Animal
{
    public:
    std::string name;
    void sleep(){cout<<"animal sleep"<<endl;}
    void virtual breathe(){cout<<"animal breathe"<<endl;}
};
class Fish:public Animal
{
    public:
    void breathe() override {cout<<"fish bubble"<<endl;}
};
```
Based on what we talked about C++ object model, we will have a virtual table pointer pointing a virtual table
```
Animal
|---------------------|
|        Animal       |
|---------------------|            |-------|
|    __vptr_Animal    |  ------->  | slot1 |  ----> type_info for Animal
|---------------------|            |-------|
                                   | slot2 |  ----> Animal::breathe()
                                   |-------|
Fiah
          |-- |---------------------|
          |   |        Animal       |
Animal    |   |---------------------|       |-------|
subobject |   |    __vptr_Animal    |  -->  | slot1 |  --> type_info for Fish
          |-- |---------------------|       |-------|
              | Fish addtional part |       | slot2 |  --> Fish::breathe()
              |---------------------|       |-------|
```
Virtual functions address have been obtained during complie time. Virtual function table size and contains can not be modified during run time. In order to find the table, very class object has a _vptr_ pointer pointing to the virtual table. And in the virtual table, the virtual function will be indexed in the table slots. All this will be done in the compile time, while in the runtime it will activate the virtual function.

Activatation the virtual function has three possibility:
1. override the base virtual function, it will replace the base class virtual function
2. Inherent the base virtual function
3. Add new virtual function, enlager the virtual table.

(pure vitual function index put _pure_virtual_call() with the corresponding slot).
So when we change _breathe_ as virtual function in _Animal_ and override _breathe_ in _Fish_, it will replace the slot2 with `Fish::breathe()` during the activation in the runtime.

#### Virtual destructor
Now, I think we can explain why the base class destructor need to be virtual. 
```
int main()
{
    Animal *pAn = new Fish;
    delete pAn;
}
```
When we do the `delete pAn` it will call the _Animal_ destructor to destruct an object of _Animal_ but it actually created a _Fish_ object. it will cause memory problem.
