---
title: Solutions for Xiyou Linux Group 2020 Interview Questions
date: 2022-11-16 22:32:19
tags:
    - linux
    - c++
    - 算法
categories: Tests for Basic Knowledge of C
---

<!--more-->

# Solutions for Xiyou Linux Group 2020 Interview Questions

> Thanks to [Zhilu](https://github.com/L33Z22L11) for re-entering the original title. Good people live in peace.

>Note:
> 1. This question is only used as a limited reference for the interview
> 2. To save layout, omit all `#include` directives
> 3. The difficulty of the question has nothing to do with the serial number
> 4. If there is no special statement, it is assumed to be under the `Linux x86_64 GCC` compiler environment


## 1. Please try to explain its output.
```c
int main(int argc , char *argv[]) {
  unsigned char a = 255;
  // a = 1111 1111
  char ch = 128;
  // ch = 127+1 = 1000 0000 = -128
  a -= ch;
  // a = a - ch = 1111 1111 + 1000 0000 = 0111 1111 = 127
  printf("a = %d ch = %d\n", a, ch);
  // a = 127 ch = -128
}
```

## 2. What is the output of the code below, talk about your understanding.
```c
int main(int argc, char *argv[]) {
  char *str = "Xi You Linux Group 20";
  printf("%d\n", printf(str));
  return 0;
}
```
> - printf() has a return value whose size is the number of characters printed.
## 3. What is the output of this code? Why does this result occur?
```c
#include <stdio.h>
int i = 2;
void func() {
    if (i != 0) {
        // true
        static int m = 0;
        int n = 0;
        n++;    // n = 1
        m++;    // m = 1
        printf("m = %d, n = %d\n", m, n);
        i--;    // i = 1
        func();
        // new n = 1;
        // m = 2;
        // i = 0
        // false, loop ends here.
    } else {
        return;
    }
}
// So the results are:
// m = 1, n = 1
// m = 2, n = 1
int main(int argc, char* argv[]) {
    func();
    return 0;
}
```
## 4. What is the result of the following program? Why does this result occur?
```c
int main(int argc, char * argv[]) {
  char ch = 'A';
  int i = 65;
  unsigned int f = 33554433;
  *(int *)&f >>= 24;
  *(int *)&f = *(int *)&f + '?';
  printf("ch = %c i = %c f = %c\n", ch, i, *(int *)&f);
  return 0;
}
```

```c
#include <stdio.h>
int main(int argc, char* argv[]) {
    char ch = 'A';
    int i = 65;
    unsigned int f = 33554433;
    // f = 0000 0010 0000 0000 0000 0000 0000 0001
    *(int*)&f >>= 24;
    // move right 24 bits, f = 0000 0000 0000 0000 0000 0000 0000 0010
    *(int*)&f = *(int*)&f + '?';
    // '?' == 63, so f + 63 = 0000 0000 0000 0000 0000 0000 0100 0001 = 65
    printf("ch = %c i = %c f = %c\n", ch, i, *(int*)&f);
    // So: ch = A i = A f = A
    return 0;
}
```

## 5. What is the output of the code below, talk about your understanding.
```c
int main(int argc, char *argv[]) {
  int a[2][2];
  printf("&a = %p\t&a[0] = %p\t&a[0][0] = %p\n", &a, &a[0], &a[0][0]);
  printf("&a+1 = %p\t&a[0]+1 = %p\t&a[0][0]+1= %p\n", &a+1, &a[0]+1, &a[0][0]+1);
  return 0;
}
```

```c
#include <stdio.h>
int main(int argc, char* argv[]) {
    int a[2][2];
    printf("&a = %p\t&a[0] = %p\t&a[0][0] = %p\n", &a, &a[0], &a[0][0]);
    // These three results are all the same thing, all of which are the starting address of the container a
    // &a = 0x16f0b7528        &a[0] = 0x16f0b7528     &a[0][0] = 0x16f0b7528
    printf("&a+1 = %p\t&a[0]+1 = %p\t&a[0][0]+1= %p\n", &a + 1, &a[0] + 1,
           &a[0][0] + 1);
    // The size of array a is 4*4 = 16 bytes, the size of a[] is 2*4 = 8, and the size of an integer is 4.
    // So the output is: &a+1 = 0x16f0b7528 + 16      &a[0]+1 = 0x16f0b7528 + 8   &a[0][0]+1= 0x16f0b7528 + 4
    // &a+1 = 0x16f0b7538      &a[0]+1 = 0x16f0b7530   &a[0][0]+1= 0x16f0b752c
    return 0;
}
```

## 6. What is the function of the following programs? What's the problem, can you figure it out and fix it?
```c
int* get_array() {
  int array[1121];
  for (int i = 0; i < sizeof(array) / sizeof(int); i++) {
    array[i] = i;
  }
  return array;
}
int main(int argc, char *argv[]) {
  int *p = get_array();
}
```

```c
#include <stdio.h>
int* get_array() {
    int array[1121];
    printf("%lu\n",sizeof(array));
    for (int i = 0; i < sizeof(array)/sizeof(int); i++) {
        array[i] = i;
    }
    return array;
}
// This is a pretty stupid mistake, if the address of the local variable is returned, the local array variable is located in the stack area.
// After the function ends, the data in this address will lose its meaning;
// what to do in this case, you can add static to the local variable array;
int main(int argc, char* argv[]) {
    int* p = get_array();
    for (int i = 0; i < 1121; i++) {
        printf("%d ", *(p+i));
    }

}
```
**Here is it:**

```c
#include <stdio.h>
int* get_array() {
    int static array[1121];
    for (int i = 0; i < sizeof(array)/sizeof(int); i++) {
        array[i] = i;
    }
    return array;
}
int main(int argc, char* argv[]) {
    int* p = get_array();
    for (int i = 0; i < 1121; i++) {
        printf("%d ", *(p+i));
        if(i%30 == 0 && i != 0){
            printf("\n");
        }
    }

// 0 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 27 28 29 30
// 31 32 33 34 35 36 37 38 39 40 41 42 43 44 45 46 47 48 49 50 51 52 53 54 55 56 57 58 59 60
// 61 62 63 64 65 66 67 68 69 70 71 72 73 74 75 76 77 78 79 80 81 82 83 84 85 86 87 88 89 90
// 91 92 93 94 95 96 97 98 99 100 101 102 103 104 105 106 107 108 109 110 111 112 113 114 115 116 117 118 119 120
// 121 122 123 124 125 126 127 128 129 130 131 132 133 134 135 136 137 138 139 140 141 142 143 144 145 146 147 148 149 150
// 151 152 153 154 155 156 157 158 159 160 161 162 163 164 165 166 167 168 169 170 171 172 173 174 175 176 177 178 179 180
// 181 182 183 184 185 186 187 188 189 190 191 192 193 194 195 196 197 198 199 200 201 202 203 204 205 206 207 208 209 210
// 211 212 213 214 215 216 217 218 219 220 221 222 223 224 225 226 227 228 229 230 231 232 233 234 235 236 237 238 239 240
// 241 242 243 244 245 246 247 248 249 250 251 252 253 254 255 256 257 258 259 260 261 262 263 264 265 266 267 268 269 270
// 271 272 273 274 275 276 277 278 279 280 281 282 283 284 285 286 287 288 289 290 291 292 293 294 295 296 297 298 299 300
// 301 302 303 304 305 306 307 308 309 310 311 312 313 314 315 316 317 318 319 320 321 322 323 324 325 326 327 328 329 330
// 331 332 333 334 335 336 337 338 339 340 341 342 343 344 345 346 347 348 349 350 351 352 353 354 355 356 357 358 359 360
// 361 362 363 364 365 366 367 368 369 370 371 372 373 374 375 376 377 378 379 380 381 382 383 384 385 386 387 388 389 390
// 391 392 393 394 395 396 397 398 399 400 401 402 403 404 405 406 407 408 409 410 411 412 413 414 415 416 417 418 419 420
// 421 422 423 424 425 426 427 428 429 430 431 432 433 434 435 436 437 438 439 440 441 442 443 444 445 446 447 448 449 450
// 451 452 453 454 455 456 457 458 459 460 461 462 463 464 465 466 467 468 469 470 471 472 473 474 475 476 477 478 479 480
// 481 482 483 484 485 486 487 488 489 490 491 492 493 494 495 496 497 498 499 500 501 502 503 504 505 506 507 508 509 510
// 511 512 513 514 515 516 517 518 519 520 521 522 523 524 525 526 527 528 529 530 531 532 533 534 535 536 537 538 539 540
// 541 542 543 544 545 546 547 548 549 550 551 552 553 554 555 556 557 558 559 560 561 562 563 564 565 566 567 568 569 570
// 571 572 573 574 575 576 577 578 579 580 581 582 583 584 585 586 587 588 589 590 591 592 593 594 595 596 597 598 599 600
// 601 602 603 604 605 606 607 608 609 610 611 612 613 614 615 616 617 618 619 620 621 622 623 624 625 626 627 628 629 630
// 631 632 633 634 635 636 637 638 639 640 641 642 643 644 645 646 647 648 649 650 651 652 653 654 655 656 657 658 659 660
// 661 662 663 664 665 666 667 668 669 670 671 672 673 674 675 676 677 678 679 680 681 682 683 684 685 686 687 688 689 690
// 691 692 693 694 695 696 697 698 699 700 701 702 703 704 705 706 707 708 709 710 711 712 713 714 715 716 717 718 719 720
// 721 722 723 724 725 726 727 728 729 730 731 732 733 734 735 736 737 738 739 740 741 742 743 744 745 746 747 748 749 750
// 751 752 753 754 755 756 757 758 759 760 761 762 763 764 765 766 767 768 769 770 771 772 773 774 775 776 777 778 779 780
// 781 782 783 784 785 786 787 788 789 790 791 792 793 794 795 796 797 798 799 800 801 802 803 804 805 806 807 808 809 810
// 811 812 813 814 815 816 817 818 819 820 821 822 823 824 825 826 827 828 829 830 831 832 833 834 835 836 837 838 839 840
// 841 842 843 844 845 846 847 848 849 850 851 852 853 854 855 856 857 858 859 860 861 862 863 864 865 866 867 868 869 870
// 871 872 873 874 875 876 877 878 879 880 881 882 883 884 885 886 887 888 889 890 891 892 893 894 895 896 897 898 899 900
// 901 902 903 904 905 906 907 908 909 910 911 912 913 914 915 916 917 918 919 920 921 922 923 924 925 926 927 928 929 930
// 931 932 933 934 935 936 937 938 939 940 941 942 943 944 945 946 947 948 949 950 951 952 953 954 955 956 957 958 959 960
// 961 962 963 964 965 966 967 968 969 970 971 972 973 974 975 976 977 978 979 980 981 982 983 984 985 986 987 988 989 990
// 991 992 993 994 995 996 997 998 999 1000 1001 1002 1003 1004 1005 1006 1007 1008 1009 1010 1011 1012 1013 1014 1015 1016 1017 1018 1019 1020
// 1021 1022 1023 1024 1025 1026 1027 1028 1029 1030 1031 1032 1033 1034 1035 1036 1037 1038 1039 1040 1041 1042 1043 1044 1045 1046 1047 1048 1049 1050
// 1051 1052 1053 1054 1055 1056 1057 1058 1059 1060 1061 1062 1063 1064 1065 1066 1067 1068 1069 1070 1071 1072 1073 1074 1075 1076 1077 1078 1079 1080
// 1081 1082 1083 1084 1085 1086 1087 1088 1089 1090 1091 1092 1093 1094 1095 1096 1097 1098 1099 1100 1101 1102 1103 1104 1105 1106 1107 1108 1109 1110
// 1111 1112 1113 1114 1115 1116 1117 1118 1119 1120
// The result is normal!
}
```

## 7. What is the output of the code below, and talk about your understanding.

```c
int main(int argc, char *argv[]) {
  char str[] = "XiyouLinuxGroup";
  char *p = str;
  char x[] = "XiyouLinuxGroup\t\106F\bamily";
  printf("%zu %zu %zu %zu\n", sizeof(str), sizeof(p), sizeof(x), strlen(x));
  return 0;
}
```

```c
#include <stdio.h>
#include <string.h>
int main(int argc, char* argv[]) {
    char str[] = "XiyouLinuxGroup";
    char* p = str;
    char x[] = "XiyouLinuxGroup\t\106F\bamily";
    for(int i = 0; i < 25; i++){
        printf("%c", x[i]);
    }
    // XiyouLinuxGroup Family
    printf("%zu %zu %zu %zu\n", sizeof(str), sizeof(p), sizeof(x), strlen(x));
    // 15+1('\0') = 16, 8, 24+1('\0'), 24
    return 0;
}
```
> - There is no regulation or standard for how many spaces to skip. Each output device will stipulate that **\t** will be positioned at a multiple of an integer unit on its own device.

## 8. The following program, according to the print results, what do you think?
```c
int add(int *x, int y) {
  return *x = (*x^y) + ((*x&y)<<1);
}
int a;
int main(int argc, char *argv[]) {
  int b = 2020;
  if(add(&b, 1) || add(&a, 1)) {
  printf("XiyouLinuxGroup%d\n", b);
  printf("Waiting for y%du!\n", a);
  }
  if(add(&b, 1) && a++) {
  printf("XiyouLinuxGroup%d\n", b);
  printf("Waiting for y%du!\n", a);
}
  return 0;
}
```
> - This involves a short circuit **||** .

## 9. In the next program, we can print out the address of `a` through the first step, if the result printed on your machine is `0x7ffd737c6db4`; we use the `scanf` function in the second step to input this address value into the variable `c `Middle; the third step, randomly input a number, what is the final output, do you know the principle?
> - Definition of reference pointer.
```c
void func() {
  int a = 2020;
  unsigned long c;
  printf("%p\n", &a);
  printf("我们想要修改的地址：");
  scanf("%lx", &c);
  printf("请随便输入一个数字：");
  scanf("%d", (int *)c);
  printf("a = %d\n", a);
}
```


## 10. What process will a C language program go through from source code to executable file, can you briefly describe what each link does?
> - From source files to executable files generally need to go through several steps: preprocessing -> compile -> assembly -> link these four processes.
> - Preprocessing: Preprocessing is equivalent to converting the source code into a new c program according to the preprocessing command, but usually with an i extension.
> - Compile: Translate the resulting i file into assembly code, usually with an s extension.
> - Assembly: Translate assembly files into machine instructions and package them into o files for relocatable object programs
## 11. Please explain what this line of code does

```c
int main(){
    puts((char*)(int const[]){0X6F796958, 0X6E694C75, 0X72477875, 0X3270756F,
                          0X313230, 0X00000A});
    return 0;
}
```
> - This is a bit like Java's anonymous objects, but that's not the point.
> - The focus of this question is endianness, but this concept is very simple, so I won't talk about it more.

## 12. Please enter a random string, can you explain the output?
```c
int main(int argc, char *argv[]) {
  char str[1121];
  int key;
  char t;
  fgets(str, 1121, stdin);
  for(int i = 0; i < strlen(str) - 1; i++) {
    key = i;
    for(int j = i + 1; j < strlen(str); j++) {
      if(str[key] > str[j]) {
        key = j;
      }
    }
    t = str[key];
    str[key] = str[i];
    str[i] = t;
  }
  puts(str);
  return 0;
}
```
> - // This is actually a variant of selection sort, it is the same as the code below:

```c
#include <stdio.h>
#include <string.h>
int main(int argc, char* argv[]) {
    char str[100];
    int key;
    char t;
    fgets(str, 100, stdin);
    for (int i = 0; i < strlen(str) - 1; i++) {
        for (int j = i + 1; j < strlen(str); j++) {
            if (str[i] > str[j]) {
                t = str[i];
                str[i] = str[j];
                str[j] = t;
            }
        }
    }
    puts(str);
    return 0;
}
```

## 13. Use loop and recursion to find `Fibonacci` sequence, which one do you think is better? Give your opinion. If you are asked to find the 100th term of the `Fibonacci` sequence, do you think it can be solved by conventional methods? Please try to find the value of the first 100 items (tip operation with large numbers).
**The answer for recursion is stated below:**

```c
#include <stdio.h>
#include <string.h>
int fei(int n){
    if(n == 1 || n == 2){
        return 1;
    }
    return fei(n-1)+fei(n-2);
}
// 1, 1, 2, 3, 5, 8, 13, 21
int main() {
    int n;
    scanf("%d", &n);
    printf("%d\n", fei(n));

}
```
**The answer for loop is stated below:**

```c
#include <stdio.h>
#include <string.h>

// 1, 1, 2, 3, 5, 8, 13, 21
int main() {
    int n;
    scanf("%d", &n);
    int a1 = 1, a2 = 1, a3;
    for(int i = 3; i <= n; i++){
        a3 = a1+a2;
        a1 = a2;
        a2 = a3;
    }
    printf("%d", a3);

}
```
> - Obviously, a loop is better because recursion consumes a lot of stack space.
> - Output the first 100 items, because the number is too large, it can only be calculated by simulation.

**The answer is stated below:**
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <math.h>
char* add(char* num1, char* num2) {
    int i = strlen(num1) - 1, j = strlen(num2) - 1, add = 0;
    char* ans = (char*)malloc(sizeof(char) * (fmax(i, j) + 3));
    int len = 0;
    while (i >= 0 || j >= 0 || add != 0) {
        int x = i >= 0 ? num1[i] - '0' : 0;
        int y = j >= 0 ? num2[j] - '0' : 0;
        int result = x + y + add;
        ans[len++] = '0' + result % 10;
        add = result / 10;
        i--, j--;
    }
    for (int i = 0; 2 * i < len; i++) {
        int t = ans[i];
        ans[i] = ans[len - i - 1], ans[len - i - 1] = t;
    }
    ans[len++] = 0;
    return ans;
}
int main() {
    char *a1 = "1";
    char *a2 = "1";
    char *a3;
    printf("1th: %s\n", "1");
    printf("2th: %s\n", "1");
    for(int i = 3; i <= 100; i++){
        a3 = add(a1, a2);
        printf("%dth: %s\n", i, a3);
        a1 = a2;
        a2 = a3;
    }
    return 0;
}
```
> - 1th: 1
2th: 1
3th: 2
4th: 3
5th: 5
6th: 8
7th: 13
8th: 21
9th: 34
10th: 55
11th: 89
12th: 144
13th: 233
14th: 377
15th: 610
16th: 987
17th: 1597
18th: 2584
19th: 4181
20th: 6765
21th: 10946
22th: 17711
23th: 28657
24th: 46368
25th: 75025
26th: 121393
27th: 196418
28th: 317811
29th: 514229
30th: 832040
31th: 1346269
32th: 2178309
33th: 3524578
34th: 5702887
35th: 9227465
36th: 14930352
37th: 24157817
38th: 39088169
39th: 63245986
40th: 102334155
41th: 165580141
42th: 267914296
43th: 433494437
44th: 701408733
45th: 1134903170
46th: 1836311903
47th: 2971215073
48th: 4807526976
49th: 7778742049
50th: 12586269025
51th: 20365011074
52th: 32951280099
53th: 53316291173
54th: 86267571272
55th: 139583862445
56th: 225851433717
57th: 365435296162
58th: 591286729879
59th: 956722026041
60th: 1548008755920
61th: 2504730781961
62th: 4052739537881
63th: 6557470319842
64th: 10610209857723
65th: 17167680177565
66th: 27777890035288
67th: 44945570212853
68th: 72723460248141
69th: 117669030460994
70th: 190392490709135
71th: 308061521170129
72th: 498454011879264
73th: 806515533049393
74th: 1304969544928657
75th: 2111485077978050
76th: 3416454622906707
77th: 5527939700884757
78th: 8944394323791464
79th: 14472334024676221
80th: 23416728348467685
81th: 37889062373143906
82th: 61305790721611591
83th: 99194853094755497
84th: 160500643816367088
85th: 259695496911122585
86th: 420196140727489673
87th: 679891637638612258
88th: 1100087778366101931
89th: 1779979416004714189
90th: 2880067194370816120
91th: 4660046610375530309
92th: 7540113804746346429
93th: 12200160415121876738
94th: 19740274219868223167
95th: 31940434634990099905
96th: 51680708854858323072
97th: 83621143489848422977
98th: 135301852344706746049
99th: 218922995834555169026
100th: 354224848179261915075
## 14. Linux Practical Questions

> Please create a directory through the command, create several files with the suffix `.Linux` in the directory, and then query the basic attribute information of these files (such as file size, file creation time, etc.) through the command, and then use the command Check the number of files with "`.Linux`" in the file name in the directory (excluding files in subdirectories), write the obtained number into a file, and finally delete the directory.
> - makedir aaa
> - touch 1.Linux
> - touch 2.Linux
> - ls -l
> - ls *.Linux
