---
title: The Rule of Three
date: 2020-07-31 17:49:29
tag: Deep copy
categories: C++
---
Once I saw a comment about C++, said that if you want to check whether the programmer is experienced or not, check if he properly handled or disabled copy constructor or assign operator if he has a destructor or delete resource. I never notice this before. The essential idea of this comment is the rule of three.

>If a class requires a user-defined destructor, a user-defined copy constructor, or a user-defined copy assignment operator, it almost certainly requires all three.

##### Constructor and Destructor
There are three casese that copy constructor will be called which means object will be the initial value for another object.
1. A aa = a;
2. f(A a)
3. return a;

```c++
Class A {
    public:
        A( int size );
        int size;
}
```
If class `A` has been defined with a copy constructor like `A( const& A a)`, it mostly will call this one. What if we do not have the copy constructor? The C++ will follow default memberwise initialization rule, which means copy the data member recursively.

```C++
A a( 5 );
A copy = a;
// no copy constructor, it will be equal to
copy.len = a.len;
```
Copy constructor will copy the value one by one like the last line. If we have a pointer in our class, we need to deal the memory manually. In constructor, we need to apply for memory and in destructor we need to free this memory. This class will be like below.

```C++
Class A {
    public:
        A( int size ) {
            data = new int[size];
        }
        ~A() {
            delete[] data;
        }
        int size;
        int* data;
}
```
##### Shallow Copy and Deep Copy
Now since we have a pointer in our class, we need to manually free the memory so we have a customized destructor to handle that. What if we do not obey the rule of three, that we do not set up a customized copy constructor and override assign operator. When we assign an object to another object, the C++ compiler will call its default copy constructor. Initilize the data member recursively.
```C++
A a( 5 );
A copy = a;
// no copy constructor, it will be equal to
copy.data = a.data;
copy.len = a.len;
```
However, this copy process is _Shallow Copy_, it is only copy the value of each data member. This will cause memory leak. Pointer `copy.data` and `A.data` pointing to the same chunk of memory. When we release object A, destructor of A will free this memory, after that we release object copy, the same chunk of memory will be freed again. Default copy constructor is shallow copy and it will cause memory leak problem.

In order to solve this, we need _Deep Copy_. Now we setup a customized copy constructor
```C++
A(const A& a) :
{
    size = a.size
	data = new int[size];
}
```
In our copy constructor, we use deep copy, which apply for a new memory for its own pointer. When release each object, the destructor will only free the memory this object applied. Now we can see the difference of _Shallow Copy_ and _Deep Copy_. 

This also explain for the rule of three. When we have a customized destructor, that means we need to handle the memory free for the pointer. If we do not obey the rule, the default copy contructor will have a shallow copy. Rule of three force us to have a deep copy.
