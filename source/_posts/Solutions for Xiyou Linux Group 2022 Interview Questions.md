---
title: Solutions for Xiyou Linux Group 2022 Interview Questions
date: 2022-11-15 21:13:14
tags:
    - linux
    - c++
    - 算法
categories: Tests for Basic Knowledge of C
---

<!--more-->

# Solutions for Xiyou Linux Group 2022 Interview Questions

> Thanks to [Zhilu](https://github.com/L33Z22L11) for re-entering the original title. Good people live in peace.

> - This topic is only a limited reference for the `Xiyou Linux  Group` 2022 interview.
> - In order to save layout, the program source code of this question omits the `#include` directive.
> - The program source code in this question is only used to examine the basics of the C language, and should not be used as an example of the "code style" of the C language.
> - The questions is arranged randomly on difficulty.
>  All topics are compiled and run on `x86_64 GNU/Linux` environment.
>
> Message from the seniors:
> For a long time, the interview questions of Xiyou Linux Group have been known in XUPT for their high difficulty. We, as test-makers, also know that this test is slightly difficult. Please do not worry. **If you can complete half of the questions, it is already very good. ** Secondly, we are more interested in your thinking and process than the answers to the questions. Maybe your answer is slightly flawed, but your correct thinking and understanding of knowledge are enough to win more points. Finally, the process of doing the test is also a process of learning and growth. We believe the test questions will definitely help you to master the C language more familiarly. Good luck. See you at FZ103!
>
> Copyright © Xiyou Linux Group, All Rights Reserved.
> This exam is licensed under the [Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International License](http://creativecommons.org/licenses/by-nc-sa/4.0/).


## 0. Is my calculator broken?!
> `2^10=1024` corresponds to 4 digits in decimal, so how many digits does `2^10000` correspond to in decimal?

> **The answer is stated below:**
```c
#include <stdio.h>
int main(){
    int count = 0;
    double temp = 1;
    for(int i = 0; i < 1000; i++){
        temp *= 1024;               // Must be of type double, the last digit may carry
        while(temp >= 10){
            count++;
            temp /= 10.0;
        }
    }
    printf("%d\n", count+1);
}
```
## 1. Can printf play like this?

```c
int main(void) {
  if ((3 + 2 < 2) > (3 + 2 > 2))
    printf("Welcome to Xiyou Linux Group\n");
  else
    printf("%d\n", printf("Xiyou Linux Group - 2%d", printf("")));
}
```
> - printf() has a return value, the return value is the length of the printed characters. So the first output is empty and the length is 0;
> - The second output is "Xiyou Linux Group - 20", a total of 22 characters so outputs "22";
> - Hence: "Xiyou Linux Group - 2022"

## 2. Hellllllllllloooooooo!

> - The output of the program is a bit strange, please try to explain the output.
> - Please talk about your understanding of `sizeof()` and `strlen()`.

```c
int main(void){
    char p0[] = "Hello,Linux";
    char *p1 = "Hello,Linux";
    char p2[11] = "Hello,Linux";
    printf("p0==p1: %d, strcmp(p0,p2): %d\n", p0 == p1, strcmp(p0, p2));
    printf("sizeof(p0): %zu, sizeof(p1): %zu, sizeof(*p2): %zu \n",
           sizeof(p0), sizeof(p1), sizeof(*p2));
    printf("strlen(p0): %zu, strlen(p1): %zu\n", strlen(p0), strlen(p1));
}
```
> - p has been specified, the size is 15, and p1 is a pointer, and the size of the pointer is generally 8 bytes (4 bytes and 32 bits only support a maximum of 4GB memory, which is obviously to be eliminated). *p2 is'H', 1 byte in size.
> -  strlen() calculates the length of the string, obviously they are all 11.

## 3. Can you change the name of  the variable?

> Please combine this question and talk about your understanding of the "life cycle" of "global variable" and "local variable" in the C language.

```c
int a = 3;
void test() {
    int a = 1;
    a += 1;
    {
        int a = a + 1;
        printf("a = %d\n", a);
    }
    printf("a = %d\n", a);
}
int main(void) {
    test();
    printf("a= %d\n", a);
}
```
> - **The answer is stated below:**
```c
#include <stdio.h>
int a = 3;
void test() {
    int a = 1;
    a += 1;
    {
        printf("%d\n", a);  // so it is 2
        int a = a + 1;
        printf("a = %d\n", a);  // local variable is not initialized, so it is random
    }
    printf("a = %d\n", a);  // a = 2
}
int main(void) {
    test();
    printf("a = %d\n", a);  //  global variable is effective,  it is 2
    return 0;
}
```


## 4. Memory misalignment!

> What are the characteristics of `union` and `struct`, do you understand their memory allocation mode?

```c
typedef union{
    long l;
    int i[5];
    char c;
} UNION;
typedef struct{
    int like;
    UNION coin;
    double collect;
} STRUCT;
int main(void){
    printf("sizeof (UNION) = %zu \n", sizeof(UNION));
    printf("sizeof (STRUCT) = %zu \n", sizeof(STRUCT));
}
```
> -  Memory alignment mainly follows the following three principles:
     1. The starting address of a structure variable is divisible by the size of its widest member;
     2. The offset of each member of the structure relative to the starting address can be divisible by its own size, if not, add bytes after the previous member;
     3. The overall size of the structure can be divisible by the size of the widest member, if not, add bytes after
> -  For UNION, it is a union, and the space occupied is the maximum value of the member, which is 20, but 20 is not an integer multiple of 8-byte long, so it will increase to 24
> -  For data, since it contains a UNION, and the widest one in UNION is 8, so 4+(4) (for divisibility)+24+8 = 40

## 5. Bitwise

> - Please use  a pen and some paper to deduce the outputs of this program.
> - Please talk about your understanding of bit operations.

```c
int main(void){
    unsigned char a = 4 | 7;
    a <<= 3;
    unsigned char b = 5 & 7;
    b >>= 3;
    unsigned char c = 6 ^ 7;
    c = ~c;
    unsigned short d = (a ^ c) << 3;
    signed char e = -63;
    e <<= 2;
    printf("a: %d, b: %d, c: %d, d: %d \n", a, b, c, (char)d);
    printf("e: %#x \n", e);
}
```
> - **The answer is stated below:**

```c
#include <stdio.h>
int main(void) {
    unsigned char a = 4 | 7;
    // a = 0000 0100 | 0000 0111 --> 0000 0111 = 7
    a <<= 3;
    // a = 0011 1000 = 32+16+8 = 56
    unsigned char b = 5 & 7;
    // b = 0000 0101 & 0000 0111 --> 0000 0101 = 5
    b >>= 3;
    // b = 0000 0000 = 0;
    unsigned char c = 6 ^ 7;
    // c = 0000 0110 ^ 0000 0111 --> 0000 0001 = 1
    c = ~c;
    // c = 1111 1110 = 255-1 = 254
    unsigned short d = (a^c) << 3;
    // d = 0011 1000 ^ 1111 1110 = 1100 0110 << 3 = 0011 0000->0000 0000 0011 0000 = 32+16 = 48
    signed char e = -63;
    // 63 = 0100 0000-1 = 0011 1111 --> 1100 0001 = -63
    e <<= 2;
    // e = 0000 0100 = 4

    printf("a: %d, b: %d, c: %d, d: %d\n", a,b,c,(char)d);
    // 56, 0, 254, 48
    printf("e: %#x\n", e);
    // 0x4
    return 0;
}
```

## 6. English to Chinese

> Please talk about the meaning of the following data types, talk about the role of `const`.
> 1. `char *const p`.
> 2. `char const *p`.
> 3. `const char *p`.

> - `char const *p` and `const char *p` are the same thing, The value pointed by p cannot be changed.
> - `char *const p`means that the p always points the same variable, cannot be changed.
## 7. Chinese to English

> Please use the variable `p` to give the following definition:
> 1. An array of 10 pointers to `int`.
> 2. A pointer to an array of 10 `int`s.
> 3. An array of 3 "pointers to functions", the pointed function has 1 `int` parameter and returns `int`.

```c
1. int *p[10];
2. int p[10];int *t = (int*)p; // now pointer t points to array p;
3. int (*p[3])(int); // First it is an array. Second, every return value of the member of the array is a pointer. Because of the name of function is a pointer too, so we just let the array contains the functions.
```
## 8. Order in Chaos

> How much do you know about sorting algorithms?
> Please talk about the idea, stability, time complexity and space complexity of the sorting algorithms you know.
> Hint: It's better to knock it out with your own hands~

> - Bubble sorting, similar to bubbling in water, the larger number sinks, and the smaller number slowly rises. Assuming from small to large, that is, the larger number is slowly arranged to the back, and the smaller number is slowly moved to the back. front row.

```c
int nums[10] = {2,1,3,4,5,6,4,5,3,7};
for(int i = 0; i < 10-1; i++){
    for(int j = 0; j < 10-i-i; j++){
        if(nums[i] > nums[j]){
            int temp = nums[i];
            nums[i] = nums[j];
            nums[j] = temp;
        }
    }
}
```
> - Selection sorting: First find the smallest element in the unsorted sequence, store it at the starting position of the sorted sequence, then continue to find the smallest element from the remaining unsorted elements, and then put it in the second element position. And so on until all elements are sorted.

```c
int nums[10] = {2,1,3,4,5,6,4,5,3,7};
for(int i = 0; i < 10-1; i++){
    for(int j = i+1; j < 10; j++){
        if(nums[i] > nums[j]){
            int temp = nums[i];
            nums[i] = nums[j];
            nums[j] = temp;
        }
    }
}
```
> - Bubble sort is to compare the left and right numbers, while selection sort is to compare the latter numbers with the first number in each round;
> - Bubble sort has more exchanges per round, while selection sort only exchanges once per round;
> - Bubble sort is to find positions by numbers, and selection sort is to find numbers at a given position;
> - When an array encounters the same number, bubble sort is relatively stable, while selection sort is unstable;
> - In terms of time efficiency, selection sort is better than bubble sort.

## 9. Hands and Brain

> - Please implement the ConvertAndMerge function:
> -  Concatenate the two input strings, and flip the case of all letters in the new string obtained after concatenation.
>  - Hint: You need to allocate space for the new string.


```c
char* convertAndMerge(char s[2][20]) {
    int len1 = strlen(s[0]);
    int len2 = strlen(s[1]);
    char* ret = (char*)malloc(sizeof(char) * (len1 + len2+1));
    strcpy(ret, s[0]);
    strcat(ret, s[1]);
    for (int i = 0; i < len1 + len2; i++) {
        if (ret[i] >= 'A' && ret[i] <= 'Z') {
            ret[i] = ret[i] + 32;
        } else if (ret[i] >= 'a' && ret[i] <= 'z') {
            ret[i] = ret[i] - 32;
        }
    }
    return ret;
}
int main(void) {
  char words[2][20] = {"Welcome to Xiyou ", "Linux Group 2022"};
  printf("%s\n", words[0]);
  printf("%s\n", words[1]);
  char *str = convertAndMerge(words);
  printf("str = %s\n", str);
  free(str);
}
```

## 10. Give you my pointers, access my inner voice

>　The output of the program is a bit strange, please try to explain the output of the program.
```c
int main(int argc, char **argv) {
  int arr[5][5];
  int a = 0;
  for (int i = 0; i < 5; i++) {
    int *temp = *(arr + i);
    for (; temp < arr[5]; temp++) *temp = a++;
  }
  for (int i = 0; i < 5; i++) {
    for (int j = 0; j < 5; j++) {
      printf("%d\t", arr[i][j]);
    }
  }
}
```
> - This is because temp < arr[5] will be established many times;
> -  For example, when i == 0, temp points to the 0th address, and arr[5] points to the 25th address. After the loop ends, a == 25
> - When i == 1, temp points to the 5th address, and arr[5] points to the 25th address, looping 20 times, overwriting the value after the last loop a[1][*],
> -  But this time it can only loop 20 times, so after the loop is over, a == 45;
## 11. Strange parameters

> Do you understand argc and argv?
> Why is the value of argc 1 when running the program directly?
> Will the program be in a loop?

```c
#include <stdio.h>
int main(int argc, char **argv){
    printf("argc = %d\n", argc);
    while (1){
        argc++;
        if (argc < 0){
            printf("%s\n", (char *)argv[0]);
            break;
        }
    }
}
```
> - argc is the number of parameters, here is a signed Integer type. Its value represents the number of parameters entered when executing the program.
> - If it is run directly, its value is 1. After entering the loop,  its value will overflow, the conditional statement will be executed.
> - If your program is a.out, if you run the program on the command line, (First, you should use the cd command to enter the directory where the a.out file is located on the command line)
> -  The running command is: ./a.out Linux hello
> - Then, the value of argc is 3, argv[0] is "a.out", argv[1] is "Linux", and argv[2] is "hello".

## 12. Strange characters

> The output of the program is a bit strange, please try to explain the output of the program.

```c
int main(int argc, char **argv){
    int data1[2][3] = {{0x636c6557, 0x20656d6f, 0x58206f74},
                       {0x756f7969, 0x6e694c20, 0x00000000}};
    int data2[] = {0x47207875, 0x70756f72, 0x32303220, 0x00000a32};
    char *a = (char *)data1;
    char *b = (char *)data2;
    char buf[1024];
    strcpy(buf, a);
    strcat(buf, b);
    printf("%s \n", buf);
    if(*buf='W') printf("LE");
    else printf("BE");
}
```
> - The data in the memory is continuously discharged, so after the connection is:
> - 636c6557 20656d6f 58206f74 756f7969 6e694c20 00000000 47207875 70756f72 32303220 00000a32
> - Because computers generally read in little-endian byte order, the value stored in the computer is:
> - 57656c63 6f6d6520 746f2058 69796f75 204c696e 00000000 75782047 726f7570 20323032 320a0000
> - 00 is a null character, so it is ignored when being printed, in fact, it prints 57656c63 6f6d6520 746f2058 69796f75 204c696e 320a
> - After printing them in order, the result is: Welcome to Xiyou Linux Group 2022/n (0a is a newline)

> - The computer circuit processes the low-order byte first, which is more efficient, because the calculation starts from the low-order byte.
> - So, the computer's internal processing is little-endian.
> - Inside computers, little endian is widely used to store data inside modern CPUs,
> - In other scenarios, such as network transfers and file storage, big endian is used.

## 13. Small tests on macro definition

> - Please talk about your understanding of `#define`.
> - try to interpret the output of the program.

```c
#include <stdio.h>
#define SWAP(a, b, t) t = a; a = b; b = t
#define SQUARE(a) a *a
#define SWAPWHEN(a, b, t, cond) if (cond) SWAP(a, b, t)
int main() {
  int tmp;
  int x = 1;
  int y = 2;
  int z = 3;
  int w = 3;
  SWAP(x, y, tmp);
  printf("x = %d, y = %d, tmp = %d\n", x, y, tmp);
  if (x > y) SWAP(x, y, tmp);
  printf("x = %d, y = %d, tmp = %d\n", x, y, tmp);
  SWAPWHEN(x, y, tmp, SQUARE(1 + 2 + z++ + ++w) == 100);
  printf("x = %d, y = %d,tmp=%d\n", x, y, tmp);
  printf("z = %d, w = %d ,tmp = %d\n", z, w, tmp);
}
```
> -  The macro definition uses the macro name to represent a string, and when the macro is expanded, the string is used to replace the macro name, which is just a simple and rude replacement.
> -  A string can contain any character, it can be a constant, an expression, an if statement, a function, etc. The preprocessor does not check it, and if there is an error, it can only be found when compiling the source program that has been expanded by macros.
> -  The macro definition is not a description or a statement. There is no need to add a semicolon at the end of the line. If a semicolon is added, even the semicolon is replaced together.

**Here we go using the rules obove!**

```c
#include <stdio.h>
#define SWAP(a, b, t) \
    t = a;            \
    a = b;            \
    b = t
#define SQUARE(a) a* a
#define SWAPWHEN(a, b, t, cond) \
    if (cond)                   \
    SWAP(a, b, t)
int main() {
    int tmp;
    int x = 1;
    int y = 2;
    int z = 3;
    int w = 3;
    // SWAP(x, y, tmp);
    tmp = x;
    x = y;
    y = tmp;
    printf("x = %d, y = %d, tmp = %d\n", x, y, tmp);
    // 2,1,1
    if (x > y)  // SWAP(x, y, tmp);
    tmp = x; // This sentence will not execute
    x = y; //2
    y = tmp; // 2
    printf("x = %d, y = %d, tmp = %d\n", x, y, tmp);
    // 1,2,2
    // SWAPWHEN(x, y, tmp, SQUARE(1 + 2 + z++ + ++w) == 100);
    // Let's expand SQUARE first:
    // SWAPWHEN(x, y, tmp,1 + 2 + z++ + ++w* 1 + 2 + z++ + ++w == 100);
    // Expand SWAPWHEN too:
    if(1 + 2 + z++ + ++w* 1 + 2 + z++ + ++w == 100)
    tmp = x; // This sentence will not execute
    x = y; // 2
    y = tmp; // 2
    printf("x = %d, y = %d,tmp=%d\n", x, y, tmp);
    // 2,2,2
    printf("z = %d, w = %d ,tmp = %d\n", z, w, tmp);
    // UB behavior, indeterminate value, indeterminate value, 2
}
```

## 14. GNU/Linux commands (Optional)
> Do you know the meaning and usage of the following commands:
>
> Note:
> > Hey! You may not be very familiar with Linux commands, or even you haven't heard of Linux.
> > But don't worry, this is an optional question and won't have a big impact on your interview!
> > Knowing about Linux is a plus, but no point is deducted if you don't!

> - `ls`
> - `rm`
> - `whoami`
> - What other GNU/Linux commands do you know?

> - `ls`: list all visible files and folders
> - `rm` : delete a file or folder
> - `whoami`: print the username associated with the current effective user ID
>  - `lsmod`: used to display modules loaded into the system

>- Congrats on getting here! Your perseverance has overcome the vast majority of students who see these test questions.
> -  Maybe you  are not satisfied with your performance, don't worry, please be confident as before.
> -  Persistence to get here has proven you are good.
>  - What are you waiting for, bring your laptop and come to the FZ103 interview!

