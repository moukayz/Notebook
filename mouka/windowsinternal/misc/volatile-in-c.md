# Volatile in C

In C, the **volatile** keyword indicates that a value may change between different accesses, even if it does not appear to be modified.

The `volatile` keyword prevents the compiler from performing **optimization** on code involving volatile objects, thus ensuring that each volatile variable assignment and read has a corresponding memory access\([Wikipedia reference](https://en.wikipedia.org/wiki/Volatile_%28computer_programming%29)\). 

It is intended to prevent the compiler from applying any optimizations on objects that can change in ways that cannot be determined by the compiler.

[For example](https://www.geeksforgeeks.org/understanding-volatile-qualifier-in-c/) a global variable can represent a data port \(usually global pointer referred as memory mapped IO\) which will be updated dynamically. The code reading data port must be declared as volatile in order to fetch latest data available at the port..

Below is a demo which shows how the compiler will behave when there is `volatile` or not

Obviously **without volatile** the four variable was optimized by the compiler so they are not shown in the assembly code, but **with volatile** keyword, the four variable was not optimized so everyone has a location in memory stack.

{% tabs %}
{% tab title="Without volatile" %}
```c
int main() {
    /* These variables will never be created on stack*/
    int a = 10, b = 100, c = 0, d = 0;

    /* "printf" will be called with arguments "%d" and
      110 (the compiler computes the sum of a+b),
      hence no overhead of performing addition at
     run-time */
    printf("%d", a + b);

   /* This code will be removed via optimization, but
    the impact of 'c' and 'd' becoming 100 can be
    seen while calling "printf" */
    a = b;
    c = b;
    d = b;

    /* Compiler will generate code where printf is
      called with arguments "%d" and 200 */
    printf("%d", c + d);

    return 0;
}

/* 
Corresponding assembly code is:

 sub     rsp, 8
 mov     esi, 110
 mov     edi, OFFSET FLAT:.LC0
 xor     eax, eax
 call    printf
 mov     esi, 200
 mov     edi, OFFSET FLAT:.LC0
 xor     eax, eax
 call    printf
 xor     eax, eax
 add     rsp, 8
```
{% endtab %}

{% tab title="With volatile" %}
```c
int main() {

    volatile int a = 10, b = 100, c = 0, d = 0;

    printf("%d", a + b);

    a = b;
    c = b;
    d = b;

    printf("%d", c + d);

    return 0;
}

/*
Corresponding assembly code :

 sub     rsp, 24
 mov     edi, OFFSET FLAT:.LC0
 mov     DWORD PTR [rsp], 10
 mov     DWORD PTR [rsp+4], 100
 mov     DWORD PTR [rsp+8], 0
 mov     DWORD PTR [rsp+12], 0
 mov     esi, DWORD PTR [rsp]
 mov     eax, DWORD PTR [rsp+4]
 add     esi, eax
 xor     eax, eax
 call    printf
 mov     eax, DWORD PTR [rsp+4]
 mov     edi, OFFSET FLAT:.LC0
 mov     DWORD PTR [rsp], eax
 mov     eax, DWORD PTR [rsp+4]
 mov     DWORD PTR [rsp+8], eax
 mov     eax, DWORD PTR [rsp+4]
 mov     DWORD PTR [rsp+12], eax
 mov     esi, DWORD PTR [rsp+8]
 mov     eax, DWORD PTR [rsp+12]
 add     esi, eax
 xor     eax, eax
 call    printf
 xor     eax, eax
 add     rsp, 24
 
*/
```
{% endtab %}
{% endtabs %}





