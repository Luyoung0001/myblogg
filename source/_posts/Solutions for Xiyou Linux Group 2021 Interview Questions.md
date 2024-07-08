---
title: Solutions for Xiyou Linux Group 2021 Interview Questions
date: 2022-11-16 17:37:21
tags:
    - linux
    - c++
    - 算法
categories: Tests for Basic Knowledge of C
---

<!--more-->

# Solutions for Xiyou Linux Group 2021 Interview Questions


> Thanks to [Zhilu](https://github.com/L33Z22L11) for re-entering the original title. Good people live in peace.

> - This topic is only a limited reference for `Xiyou Linux Interest Group` 2021 new interview questions.
> - The `#include` directive has been omitted from the program source code of the test questions to save layout.
> - The program source code in this test question is only used to examine the basics of C language and should not be used as an example of C language code style.
> - The difficulty of the question has nothing to do with the serial number.
> - All topics assume `x86_64 GNU/Linux` environment is compiled and run.
>
> Copyright © Xiyou Linux Group, All Rights Reserved.
> This exam is licensed under the [Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International License](http://creativecommons.org/licenses/by-nc-sa/4.0/).

## 1. Size and length don't mean the same thing
>What are the similarities and differences between `sizeof()` and `strlen()`?
>
> How do their results differ for different parameters? Please try an example.

```c
int main(void) {
  char s[] = "I love Linux\0\0\0";
  int a = sizeof(s);
  int b = strlen(s);
  printf("%d %d\n", a, b);
}
```
> - a is equal to the size of the string array, which is 15
> - b is equal to the length of the string s with size 12
## 2. The size of the box is related to the order in which the items are loaded
> Both `test1` and `test2` contain: 1 `short`, 1 `int`, 1 `double`, so are `sizeof(t1)` and `sizeof(t2)` equal? Why is this?

```c
struct test1 {
  int a;
  short b;
  double c;
};
struct test2 {
  short b;
  int a;
  double c;
};
int main(void) {
  struct test1 t1;
  struct test2 t2;
  printf("sizeof (t1) : %d\n", sizeof(t1));
  printf("sizeof(t2): %d\n", sizeof(t2));
}
```
> **The answer is stated below:**
```c
#include <stdio.h>
struct test1 {
    int a;
    short b;
    double c;
};
// 4 + 2 + (2) + 8 = 16
struct test2 {
    short b;
    int a;
    double c;
};
// 2 + (2) + 4 + 8 = 16
int main() {
    struct test1 t1;
    struct test2 t2;
    printf("sizeof (t1) : %d\n", sizeof(t1));
    printf("sizeof(t2): %d\n", sizeof(t2));
    // So each value is 16!
    return 0;
}
```


## 3. Oh, it's a function again
> You must be familiar with the concept of function under the teaching of advanced mathematics teacher. So do you understand functions in computer programming? Please write a `func` function that outputs the value of each element in the two-dimensional array `arr`.

```c
void func(int *p){
    for(int i = 0; i < 10; i++){
        for(int j = 0; j < 13; j++) {
            printf("%d ", *((p+i*10)+j));
        }
        printf("\n");
    }
}
int main(void) {
  int arr[10][13];
  for (int i = 0; i < 10; i++) {
    for (int j = 0; j < 13; j++) {
      arr[i][j] = rand();
    }
  }
  func((int*)arr);
}
```
## 4. Can't you change the variable name?
> - Please briefly discuss the difference between `pass-by-value` and `pass-by-reference` in combination with the following procedure.
> - Briefly talk about your understanding of the life cycle of variables in C language.
```c
int ver = 123;
void func1(int ver) {
  ver++;
  printf("ver = %d\n", ver);
}
void func2(int *pr) {
  *pr = 1234;
  printf("*pr = %d\n", *pr);
  pr = 5678;
  printf("ver = %d\n", ver);
}
int main() {
  int a = 0;
  int ver = 1025;
  for (int a = 3; a < 4; a++) {
    static int a = 5;
    printf("a = %d\n", a);
    a = ver;
    func1(ver);
    int ver = 7;
    printf("ver = %d\n", ver);
    func2(&ver);
  }
  printf("a = %d\tver = %d\n", a, ver);
}
```

```c
#include <stdio.h>
int ver = 123;
void func1(int ver) {
    ver++;
    printf("ver = %d\n", ver);
}
void func2(int* pr) {
    *pr = 1234;
    printf("*pr = %d\n", *pr);
    // 1234
    pr = 5678; //Type mismatches
    printf("ver = %d\n", ver); // 123
}
int main() {
    int a = 0;
    int ver = 1025;
    for (int a = 3; a < 4; a++) {
        static int a = 5;
        printf("a = %d\n", a);
        a = ver; // 3
        func1(ver); // 4
        int ver = 7;
        printf("ver = %d\n", ver);
        // 7
        func2(&ver);
    }
    printf("a = %d\tver = %d\n", a, ver);
    // 0, 1025
    return 0;
}
```

## 5. Matryoshka is so much fun!

> Please explain how the following procedure accomplishes the summation?

```c
unsigned sum(unsigned n) {
	return n ? sum(n - 1) + n : 0;
}
int main(void) {
	printf("%u\n", sum(100));
}
```
> - This is actually a recursive function call, thus implementing the summation. Specifically, a tail recursion.

## 6. Wrong arithmetic?
```c
void func(void) {
  short a = -2;
  unsigned int b = 1;
  b += a;
  int c = -1;
  unsigned short d = c * 256;
  c <<= 4;
  int e = 2;
  e = ~e | 6;
  d = (d & 0xff) + 0x2022;
  printf("a=0x%hx\tb=0x%x\td=0x%hx\te=0x%x\n", a, b, d, e);
  printf("c=Ox%hhx\t\n", (signed char)c);
}
```
**The answer is stated below:**

```c
#include <stdio.h>
void func(void) {
    short a = -2;
    // -a = 0000 0000 0000 0010
    // So the complement of a is: 1111 1111 1111 1110
    unsigned int b = 1;
    b += a;
    // b = 1-2 = -1 = 1111 1111 1111 1111 1111 1111 1111 1111
    int c = -1;
    // -c = 0000 0000 0000 0000 0000 0000 0000 0001
    // So the complement of c is: 1111 1111 1111 1111 1111 1111 1111 1111
    unsigned short d = c * 256;
    // c*256 is -256, and 256 = 0000 0000 0000 0000 0000 0001 0000 0000
    // So the complement of -256 is:1111 1111 1111 1111 1111 1111 0000 0000
    // d is of type unsigned short, only two bytes, so only the lower 16 bits are taken, that is: 1111 1111 0000 0000
    c <<= 4;
    // c is shifted left by 4 bits, that is: 1111 1111 1111 1111 1111 1111 1111 0000
    int e = 2;
    e = ~e | 6;
    // e = 0000 0000 0000 0000 0000 0000 0000 0010
    // ~e = 1111 1111 1111 1111 1111 1111 1111 1101
    // ~e | 6 = 1111 1111 1111 1111 1111 1111 1111 1101 | 0000 0000 0000 0000
    // 0000 0000 0000 0110 = 1111 1111 1111 1111 1111 1111 1111 1111
    d = (d & 0xff) + 0x2022;
    // d & 0xff = 1111 1111 0000 0000 & 0000 0000 1111 1111 = 0x0000
    // 0x0000 + 0x2022 = 0x2022
    printf("a=0x%hx\tb=0x%x\td=0x%hx\te=0x%x\n", a, b, d, e);
    // a = 1111 1111 1111 1110 = 0xfffe
    // b = 1111 1111 1111 1111 1111 1111 1111 1111 = 0xffffffff
    // d = 0x2022
    // e = 1111 1111 1111 1111 1111 1111 1111 1111 = 0xffffffff
    printf("c=Ox%hhx\t\n", (signed char)c);
    // c = 1111 0000 = 0xf0
}
int main() {
    func();
    // a=0xfffe        b=0xffffffff    d=0x2022        e=0xffffffff
    // c=Oxf0
    return 0;
}
```

## 7. The feud between pointers and arrays

```c
int main(void) {
  int a[3][3] = {{1, 2, 3}, {4, 5, 6}, {7, 8, 9}};
  int(*b)[3] = a;
  ++b;
  b[1][1] = 10;
  int *ptr = (int *)(&a + 1);
  printf("%d %d %d \n", a[2][1], **(a + 1), *(ptr - 1));
}
```
> **The answer is stated below:**

```c
#include <stdio.h>
int main() {
    int a[3][3] = {{1, 2, 3}, {4, 5, 6}, {7, 8, 9}};
    int(*b)[3] = a;
    ++b;
    b[1][1] = 10; // a[2][1] = 10;
    int* ptr = (int*)(&a + 1);
    // Pointer ptr points to the next address unit of the last element of the array a
    printf("%d %d %d \n", a[2][1], **(a + 1), *(ptr - 1));
    // 10, 4, 9
    return 0;
}
```
## 8. Transposition technique
> There are three variables `a`, `b`, `c` and 4 similar functions below.
> - Can you tell if it is syntactically correct to call each of these 5 functions using the values or addresses of these three variables as arguments?
> - Please find the error in the code below.
> - Is there a difference between `const int` and `int const`? If there are differences, talk about their differences.
> - Is there a difference between `const int *` and `int const *`? If there are differences, talk about their differences.

> - There is no difference between `const int` and `int const`.
> -  There is a difference between `const int *` and `int const *`. The former declares a pointer, and the value of the variable pointed to by the pointer cannot be changed, while the latter declares a pointer, which cannot be modified.

```c
int a = 1;
int const b = 2;
const int c = 3;
void funco(int n) {
  n += 1;
  n = a;
}
void func1(int *n) {
  *n += 1;
  n = &a;
}
void func2(const int *n) {
  *n += 1;
  n = &a;
}
void func3(int *const n) {
  *n += 1;
  n = &a;
}
void func4(const int *const n) {
  *n += 1;
  n = &a;
}
```
> **The answer is stated below:**

```c
#include <stdio.h>
int a = 1;
int const b = 2;
const int c = 3;
void funco(int n) {
  n += 1;
  n = a;
}
// no errors
void func1(int *n) {
  *n += 1;
  n = &a;
}
// no errors
void func2(const int *n) {
  *n += 1;
  n = &a;
}
// const int *n, declares a pointer, the variable pointed to by this pointer cannot be modified.
void func3(int *const n) {
  *n += 1;
  n = &a;
}
// int *const n declares a pointer, the value of this pointer cannot be modified, which means that it always points to the same piece of memory.
void func4(const int *const n) {
  *n += 1;
  n = &a;
}
// const int *const n declares a pointer, the memory pointed to by this pointer cannot be modified;
// The content it points to can't be modified either.
```

## 9. I heard that flipping the case of letters does not affect the reading of English?

> Please write the `convert` function to convert uppercase letters to lowercase letters and lowercase letters to uppercase letters in the string as a parameter. Returns the new string resulting from the conversion.
>
```c
char *convert(const char *s){
    char *ret = malloc(sizeof(char)*(strlen(s)+1));
    ret[strlen(s)] = '\0'; // string terminator
    strcpy(ret, s);
    for (int i = 0; i < strlen(s); i++) {
        if (ret[i] >= 'A' && ret[i] <= 'Z') {
            ret[i] = ret[i] + 32;
        } else if (ret[i] >= 'a' && ret[i] <= 'z') {
            ret[i] = ret[i] - 32;
        }
    }
    return ret;
}
int main(void) {
  char *str = "XiyouLinux Group 2022";
  char *temp = convert(str);
  puts(temp);
}
```

## 10. How to exchange gifts
 > - Please judge the right and wrong of the following three `Swap`, and analyze their advantages and disadvantages respectively.
> - A:The first two are two-number exchange implemented by macro definitions, and the last one is to define a function implementation to implement two-number exchange, but the latter is wrong.
> - Do you know what `do {...} while(0)` does here?
> - A: The effect is to loop only once, which is equivalent to only one statement
>  - Do you have any other way to implement `Swap` functionality?
> **The last answer is stated below:**
> `void Swap4(int *a, int *b){
    int temp = *a;
    *a = *b;
    *b = temp;
}
`
```c
#define Swap1(a, b, t) \
  do {                 \
    t = a;             \
    a = b;             \
    b = t;             \
  } while (0)
#define Swap2(a, b) \
  do {              \
    int t = a;      \
    a = b;          \
    b = t;          \
  } while (0)
void Swap3(int a, int b) {
  int t = a;
  a = b;
  b = t;
}
```
## 11. It is said that there is a thing called a parameter
> Do you know the meaning of `argc` and `argv`? Please explain the procedure below. Can you traverse `argv` without using `argc`？

> That is bery easy! Here it is.

```c
#include <stdio.h>
#include <string.h>
int main(int argc, char* argv[]) {
    printf("argc = %d\n", argc);
  for(int i = 0; i < strlen(*argv); i++){
    printf("%s\n", argv[i]);
  }
}
```
```c
int main(int argc, char *argv[]) {
  printf("argc = %d\n", argc);
  for (int i = 0; i < argc; i++)
    printf("%s\n", argv[i]);
}
```
> - argc is the number of parameters, here is a signed Integer type. Its value represents the number of parameters entered when executing the program.
> - If it is run directly, its value is 1. After entering the loop,  its value will overflow, the conditional statement will be executed.
> - If your program is a.out, if you run the program on the command line, (First, you should use the cd command to enter the directory where the a.out file is located on the command line)
> -  The running command is: ./a.out Linux hello
> - Then, the value of argc is 3, argv[0] is "a.out", argv[1] is "Linux", and argv[2] is "hello".

## 12. People left the building empty
> Is there any error in this code? Talk about the similarities and differences between static variables and other variables.

```c
int *func1(void) {
  static int n = 0;
  n = 1;
  return &n;
}
int *func2(void) {
  int *p = (int *)malloc(sizeof(int));
  *p = 3;
  return p;
}
int *func3(void) {
  int n = 4;
  return &n;
}
int main(void) {
  *func1() = 4;
  *func2() = 5;
  *func3() = 6;
}
```

```c
int main() {
    *func1() = 4;
    printf("%d\n", *func1() = 4);
    // static variables can be modified.
    *func2() = 5;
    printf("%d\n", *func2() = 5);
    // This can be modified.
    *func3() = 6;
    printf("%d\n", *func3() = 6);
    // This can be modified.
    return 0;
}
```

## 13. Strange outputs

```c
int main(void) {
  int data[] = {0x636c6557, 0x20656d6f, 0x78206f74,
                0x756f7969, 0x6e694c20, 0x67207875,
                0x70756f72, 0x32303220, 0x00000a31};
  puts((const char*)data);
}
```
> - Welcome to xiyou Linux group 2021/n (0a is a newline)
> - The computer circuit processes the low-order byte first, which is more efficient, because the calculation starts from the low-order byte.
> - So, the computer's internal processing is little-endian.
> - Inside computers, little endian is widely used to store data inside modern CPUs,
> - In other scenarios, such as network transfers and file storage, big endian is used.

## 14. Please talk about your understanding of the process from "C source file to exe file"
> - From source files to executable files generally need to go through several steps: preprocessing -> compile -> assembly -> link these four processes.
> - Preprocessing: Preprocessing is equivalent to converting the source code into a new c program according to the preprocessing command, but usually with an i extension.
> - Compile: Translate the resulting i file into assembly code, usually with an s extension.
> - Assembly: Translate assembly files into machine instructions and package them into o files for relocatable object programs
## 15. (Optional) Heap and Stack

> Do you understand the stack and heap in your program? What is the difference between them in use? Please briefly explain.

> 1. Different storage content
Stack: When a function is called, the stack stores each parameter (local variable) in the function. At the bottom of the stack is the next instruction after the function call.
Heap: Generally, one byte is used to store the size of the heap at the head of the heap. The specific content of the heap is arranged by the programmer.
>2. Different management methods
Stack: The system automatically allocates space, and the system automatically releases the space. For example, declare a local variable "int b" in a function. The system automatically creates space for b in the stack, and the stack space is automatically released when the corresponding life cycle ends.
Heap: requires the programmer to manually apply for and release manually, and specify the size. In the C language, the malloc function is applied, the free function is released, and new and delete are implemented in C++.
>3. Different size of space
Stack: The acquisition space is small. Under Windows, the general size is 1M or 2M. When the remaining stack space is insufficient, the allocation fails overflow.
Heap: The space obtained is related to the effective virtual memory of the system, which is more flexible and larger.
>4. Can different fragments be produced?
Stack: No fragmentation, continuous space.
Heap: The storage method of linked list is adopted, which will cause fragmentation.
>5. Different growth directions
Stack: A data structure that extends to a lower address and is a contiguous area of ​​memory.
Heap: A data structure that extends to high addresses and is a discontinuous area of ​​memory. This is because the system uses a linked list to store free memory addresses, which are naturally discontinuous, and the traversal direction of the linked list is from low addresses to high addresses.
>6. Different distribution methods
Stack: There are 2 allocation methods - static allocation and dynamic allocation. Static is done by the compiler, such as local variables; dynamic is done by the alloca function, and the compiler will do the deallocation.
Heap: All are dynamically allocated, there is no statically allocated heap.
>7. Different distribution efficiency
Stack: automatically allocated by the system, faster. But the programmer is out of control.
Heap: The memory allocated by new is generally slow and prone to memory fragmentation, but it is convenient to use.

## 16. (Optional) Multiple files
> How a program can use a function in another file without using any header file.
> - A : Just VC!

## 17.(Optional) `GNU/Linux` and files
> - Do you know how to use the command line to create files and execute files under `GNU/Linux`
folder?

> - touch a.c; gcc a.c -o a; ./a

> - Do you know the meaning of each column of the command ls under `GNU/Linux`?

> - Because Linux is a multi-user and multi-tasking system, a file may be used by many people at the same time, so we must set the permissions of each file. The order of the file's permission positions is (take -rwxr-xr-x as an example) : **rwx(Owner)r-x(Group)r-x(Other)**
> - The permissions represented by this example are: readable, writable, and executable by the user; readable, non-writable, and executable by users in the same group; readable, non-writable, and executable by other users. In addition, the execution part of some program attributes is not X, but S, which means that the user who executes the program can temporarily have the same power as the owner to execute the program. It generally appears in the system management and other commands or programs, and when the user executes it, it has the root identity.​​
> - The second column indicates the number of files. If it is a file, then the number is naturally. If it is a directory, then its number is the number of files in the directory.​​
> - The third field indicates the owner of the file or directory. If the user is currently in his own Home, then this column is probably its account name.​​
> - The fourth column indicates the group to which it belongs. Each user can have more than one group, but most users should belong to only one group. Only when the system administrator wants to give a user special permissions, he may give him another group.​​
> - The fifth column indicates the file size. The file size is represented by bytes, and the empty directory is generally 1024 bytes. Of course, you can use other parameters to make the file display unit different. For example, using ls -k is to display the size unit of a file in kb, but generally we still use byte. main.​​
> - The sixth field indicates the creation date. It is expressed in the format of "month, day, time", such as Aug 15 5:46 means 5:46 am on August 15th.​​
> - The seventh column indicates the file name. We can display hidden filenames with ls -a.

> - Do you know how to check the access time, modification time and change time of files under `GNU/Linux`? And briefly talk about their differences.

> - Accesstime: Read the content of the file once, and the time will be updated. For example, use the **less** command or the **more** command on this file. (Commands such as **ls** and **stat** do not modify file access time)
> - Modifytime: modifying the file content once will update the time. For example, after using tools such as **vim** to change the file content and save it, the file modification time changes. To see the file access time use the **ls -ul** command.
> - Change time : Changing the attributes of the file will update the time, such as using the chmod command to change the file attributes, or executing other commands that implicitly change the attributes of the file such as file size and so on.

>- Congratulations, you have completed the whole set of interview questions. Come and participate in the interview of the Xiyou Linux Group!
> - West Post Linux Interest Group Interview Time:
> - October 25, 2021 to October 31, 2021 at 8pm.
> - I heard that if you come for the interview earlier, you will leave a better impression on us.
> We are waiting for you at FZ103!

