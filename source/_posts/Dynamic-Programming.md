---
title: Dynamic Programming
date: 2021-11-08 19:38:03
tags: DP
categories: algorithm
---

### Backtracing
Fibonacci sequence, this is a classic math calculation.
```
F(1)=1
F(2)=1
F(n)=F(n-1) + F(n-2)
```
If we need to calcute F(20), we need F(19) and F(18). For F(19) we need to know F(18) and F(17) util it reaches F(1) and F(2). we can have the following Tree structure.
```
                           F(20)
              |--------------|---------------|
            F(19)                          F(18)
        |-----|-----|                  |-----|-----|
      F(18)       F(17)              F(17)       F(16)
```
And the leaf of every branch is F(1) and F(2). This looks like we can use recursive method to solve the problem. The recursive method has two condition:
1. Baseline, the end of recursive which is F(1)=1, F(2)=1
2. Recursive condition, F(n)=F(n-1)+F(n-2)
```
int fib(int N) {
    if (N == 1 || N == 2) return 1;
    return fib(N - 1) + fib(N - 2);
}
```
The bracktracing approach will go to the leaf of every branch, it is a Brute-force algorithm. For Tree structure, it is also called DFS. From the Tree graphic, we noticed that F(18) appeared twice. and F(17) will appear three times, which means we have a lots of duplicated calculation. Brachtracing is calculating from top to bottom.

### Dynamic Programming
Since we know the leaf of every branch is F(1) or (F2), we consider to calculate every node of this tree from bottom to top, this will be much simplier. This is the whole thought of Dynamic Programming. DP problem we need to have a state transition function
```
F(n) = |  1  n=1,2
       |  F(n-1)+F(n-2), n>2
```
It is look exactly same with the bracktracing conditions. Here is the code
```
int fib(int n) {
    vector<int> dp(n, 0);
    dp[0] = 1;
    for(int i = 1; i < n; i++ ) {
        if( i == 1) dp[1]=2;
        else dp[i] = dp[i-1] + dp[i-2];
    }
    return dp[n-1];
}
```
Iterate from bottom to top, calculator every node to avoid duplicated calculation. We also can consider dp problem as how to get N from function F previous state. Leetcode 70, Climbing Stairs, for the number N stair, it only has two way to get there from its previous state, either from the N-1 stair or N-2 stair. So for the number N stair, it is the sum of ways to get to N-1 stair and ways of N-2 stair.

underconstruction
bracktracing checkerboard/jump game
dp Palindromic Substring
