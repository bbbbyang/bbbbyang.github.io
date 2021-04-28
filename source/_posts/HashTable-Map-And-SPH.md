---
title: HashTable And SPH
date: 2019-04-05 17:01:55
tags: HashTable
categories: Algorithm
---


#### Data Structure of Hash Table
HashTable is used for storing key-value pairs. In Java, when not considering synchronized(thread safety) , we usually use HashMap like
```java
HashMap<Integer,String> hm=new HashMap<Integer,String>();
hm.put(100,"LIU");
```

The ideal hash table is a fixed array containing some value. Use the key to find the corresponding data. We can check the following diagram.
![](https://upload.wikimedia.org/wikipedia/commons/thumb/7/7d/Hash_table_3_1_1_0_1_0_0_SP.svg/315px-Hash_table_3_1_1_0_1_0_0_SP.svg.png)

We want all keys mapping into different units. However, this is impossible because that the size of the array is fixed while we can have key-value as many as possible. So after some kind of hashing function, we will get collisions which are two or more keys result in same value. One way to resolve this collision called separate chaining. This is how HashMap/Table implemented in Java.

We have two data structure combined, one is array another one is linked list. Every element in the array is a linked list. When it comes to a collision, the new element will get inserted into the linked list. The data structure is like the following picture

![](http://wiki.jikexueyuan.com/project/java-collection/images/hashtable1.png)

#### HashTable in SPH

We can see that HashTable data structure is easy to understand. In Java we can directly use HashMap when not considering thread safety. But how could we use this data structure thoughts to solve problems? Here is an application in classic computer graphics fluid simulation algorithm, SPH, Smoothed particle Hydrodynamics.

Here I will skip all the mathematics principle introduction in SPH. The basic idea of SPH is that there are thousand hundreds of particles in space and every particle will have mainly three forces based on the distance from others (which means one particle will only get affected by the ones within a distance based on itself). Follow this principle, the particles will be moving like fluid.

![](http://rnd-zimmer.de/images/sph_particles2.png)

Here is the 3-D version of SPH simulation. There are so many particles within this space. For every single particle, we need to calculate forces from other particles. Lucky, we only need to find all the neighbour particles, the ones within a distance(h) to it.

![](https://github.com/bbbbyang/PictureRepository/blob/master/SPH/Particles.jpg?raw=true)

HashTable is a good way used to find the neighbour particles. First we need to put our space into a grid structure. This grid structure is an array, we can simply use a 1-D array to represent 2-D space. The cell size will be the distance that neighbour particles will get affected (I will explain it later).

![](https://github.com/bbbbyang/PictureRepository/blob/master/SPH/Gird.jpg?raw=true)

```C++
	Cell_Size = distance;
	Grid_Size = Space_Size / Cell_Size;
	Grid_Size.x = (int)Grid_Size.x;
	Grid_Size.y = (int)Grid_Size.y;
	Number_Cells = (int)Grid_Size.x * (int)Grid_Size.y;
	Cells = (Cell *)malloc(sizeof(Cell) * Number_Cells);
```

So Every particle will be falling into one cell. Here is the problem. There must be cases that many particles fall into one cell. Now, we need to use that separate chaining, linked list structure. Particles that are in one cell, will be stored into a linked list. Here is how we use HashTable structure to store all particles.

```C+++
void SPH::Hash_Grid(){
	for(int i = 0; i < Number_Cells; i++)
		Cells[i].head = NULL;
	int hash;
	Particle *p;
	for(int i = 0; i < Number_Particles; i ++){
		p = &Particles[i];
		//Calculate_Cell_Position is to find the coordinates in the space
		//Calculate_Cell_Hash is to find the index in that 1-D array grid
		hash = Calculate_Cell_Hash(Calculate_Cell_Position(p->pos));
		if(Cells[hash].head == NULL){
			p->next = NULL;
			Cells[hash].head = p;
		}
		else{
			p->next = Cells[hash].head;
			Cells[hash].head = p;
		}
	}
}
```

When we need to find its neighbour particles, first we need to find the one is in which cell. Then, we can only pick the cells around this cell (particles in 8 cells instead of calculating all particles, I did not show all cells) that is why we use distance as the cell size.

![](https://github.com/bbbbyang/PictureRepository/blob/master/SPH/neighbour.jpg?raw=true)

For every particle, the x and y position will be used as key and itself will be used as value. We have an array representing grid space and particles in one cell will be stored in a linked list. If you are interested in SPH source code, please check [here][SPH].

[SPH]:https://github.com/bbbbyang/Smoothed-Particle-Hydrodynamics