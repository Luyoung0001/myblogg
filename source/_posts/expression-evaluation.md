---
title: "表达式求值实现：词法分析、中缀转后缀与测试"
date: 2024-07-22 16:57:40
tags:
  - "expression evaluator"
  - "parser"
  - "ysyx"
categories:
  - "Systems Programming"
---

<!--more-->

## 一、构建 tokens

所有的代码基本上都在 sdb.c中实现，没有轮子自己造。



构建 tokens 这步，基本上就是实现一个简单的词法分析器，从 args[0]开始，一个一个模式进行识别。比如对于表达式：
- 0x80100000+   ($a0 +5)*4 - ( $ t1 + 8) + 32

首先就要能识别出 0x开头的 16进制数、$开头的寄存器，识别出来以后，我们首先就是把这些能计算的都先计算，然后转化为 10 进制数的字符串，做成 token，存到 tokens 中。因此首先需要一个辅助函数：
```c
// 根据寄存器的名字返回寄存器的值

int getVfromRegs(const char* reg) {
    const char* regs[] = {"$0", "ra", "sp",  "gp",  "tp", "t0", "t1", "t2",
                          "s0", "s1", "a0",  "a1",  "a2", "a3", "a4", "a5",
                          "a6", "a7", "s2",  "s3",  "s4", "s5", "s6", "s7",
                          "s8", "s9", "s10", "s11", "t3", "t4", "t5", "t6"};
    int numRegs = sizeof(regs) / sizeof(regs[0]);

    for (int i = 0; i < numRegs; i++) {
        if (strcmp(regs[i], reg + 1) == 0) {
            return cpu.gpr[i];
        }
    }

    // 如果没有找到匹配的寄存器
    fprintf(stderr, "Error: Register '%s' No found.\n", reg);
    return -1;  // -1 代表错误，可以结合打印信息
}

```
这个函数能根据寄存器的名字，返回寄存器的值。

接着就可以分析了：
```c
unsigned calc(char* args) {
    Token tokens[1024] = {0};
    int tokens_len = 0;
    char patte[64] = {0};
    int patte_len = 0;
    char* ptr = args;

    while (*ptr) {
        // 如果是一个空格，那么前面的 pattern 一定可以配对
        if (isspace(*ptr)) {
            if (patte_len) {
                tokens[tokens_len++] = make_token(patte);
                patte_len = 0;
                memset(patte, 0, sizeof(patte));
            }
            ptr++;
            continue;
        }
        // 如果是运算符，那么前面的也一定能配对，并且当前也一定能配对
        if (*ptr == '+' || *ptr == '-' || *ptr == '*' || *ptr == '/' ||
            *ptr == '(' || *ptr == ')') {
                // 配对两次
            if (patte_len) {
                tokens[tokens_len++] = make_token(patte);
                patte_len = 0;
                memset(patte, 0, sizeof(patte));
            }
            char op[2] = {*ptr, '\0'};
            tokens[tokens_len++] = make_token(op);
            ptr++;
        } else {
            patte[patte_len++] = *ptr;
            ptr++;
        }
    }
    // 结尾也配对
    if (patte_len) {
        tokens[tokens_len++] = make_token(patte);
    }

    // 开始计算
    // 将中缀表达式转化为后缀表达式
    Token post[2048];
    int len = 0;
    midToPost(tokens, post, tokens_len, &len);

    return evalRPN(post, len);
}
```

看起来很简单，接下来 make_token():

```c
Token make_token(const char* patte) {
    Token* ret = malloc(sizeof(Token));
    if (ret == NULL) {
        fprintf(stderr, "Memory allocation failed\n");
        exit(EXIT_FAILURE);
    }
    switch (patte[0]) {
        case '+':
            ret->type = TK_PLUS;
            strcpy(ret->str, patte);
            break;
        case '-':
            ret->type = TK_MINUS;
            strcpy(ret->str, patte);
            break;
        case '*':
            ret->type = TK_MULTIPLY;
            strcpy(ret->str, patte);
            break;
        case '/':
            ret->type = TK_DIVIDE;
            strcpy(ret->str, patte);
            break;
        case '(':
            ret->type = TK_LPAREN;
            strcpy(ret->str, patte);
            break;
        case ')':
            ret->type = TK_RPAREN;
            strcpy(ret->str, patte);
            break;
        default:
            if (patte[0] == '0') {  // 寄存器或者 16 进制的值直接转化
                unsigned decimalValue = strtoul(patte, NULL, 16);
                sprintf(ret->str, "%u", decimalValue);
            } else if (patte[0] == '$') {
                // 查找寄存器
                unsigned num = getVfromRegs(patte);
                sprintf(ret->str, "%u", num);
            } else {
                strcpy(ret->str, patte);
            }
            ret->type = TK_NUMBER;

            break;
    }
    return *ret;
}

```

make_token 的关键点在于，如果一个 pattern 是以 0 开始的，那么它一定是一个 16 进制数；如果是以 $ 开始的，那么它一定是一个寄存器。这就有了关键的计算：

```c
    if (patte[0] == '0') {  // 寄存器或者 16 进制的值直接转化
                unsigned decimalValue = strtoul(patte, NULL, 16);
                sprintf(ret->str, "%u", decimalValue);
            } else if (patte[0] == '$') {
                // 查找寄存器
                unsigned num = getVfromRegs(patte);
                sprintf(ret->str, "%u", num);
            } else {
                strcpy(ret->str, patte);
            }
            ret->type = TK_NUMBER;

            break;
    }

```

这里面处理的时候，将所有的值都是按照无符号保存的。

这时候，就可以认为，tokens 已经大功告成了！

## 二、中缀转后缀

这个需要造一点轮子：
```c
// 栈
typedef struct {
    Token data[100];
    int top;
} Stack;

// 栈操作
void push(Stack* s, Token token) {
    s->data[++s->top] = token;
}

Token pop(Stack* s) {
    return s->data[s->top--];
}

Token peek(Stack* s) {
    return s->data[s->top];
}

int isEmpty(Stack* s) {
    return s->top == -1;
}
```

还需要确定 token 类型：
```c
// 0:number
// 1:op
// 2:()
// END
int isType(TokenType type) {
    switch (type) {
        case TK_NUMBER:
            return 0;
        case TK_DIVIDE:
        case TK_MINUS:
        case TK_MULTIPLY:
        case TK_PLUS:
            return 1;
        case TK_LPAREN:
        case TK_RPAREN:
            return 2;
        default:
            return -1;
    }
}
```
还要比较优先级：
```c
// 比较优先级
int priority(TokenType type) {
    switch (type) {
        case TK_PLUS:
        case TK_MINUS:
            return 0;
        case TK_MULTIPLY:
        case TK_DIVIDE:
            return 1;
        default:
            return -1;
    }
}
```

接着就可以中缀转后缀了：
```c
// 中缀转化为后缀表达式
void midToPost(Token* mid, Token* post, int token_len, int* new_len) {
    Stack ops;
    ops.top = -1;
    int j = 0;

    for (int i = 0; i < token_len; i++) {
        Token token = mid[i];
        if (isType(token.type) == 0) {
            post[j++] = token;
        } else if (token.type == TK_LPAREN) {
            push(&ops, token);
        } else if (token.type == TK_RPAREN) {
            while (!isEmpty(&ops) && peek(&ops).type != TK_LPAREN) {
                post[j++] = pop(&ops);
            }
            pop(&ops);
        } else {
            while (!isEmpty(&ops) && isType(peek(&ops).type) &&
                   priority(token.type) <= priority(peek(&ops).type)) {
                post[j++] = pop(&ops);
            }
            push(&ops, token);
        }
    }

    while (!isEmpty(&ops)) {
        post[j++] = pop(&ops);
    }
    *new_len = j;  // 长度
}
```
思路就是，如果它是一个数，就存到 post 中；如果是一个左括号，就进栈；如果是一个右括号，就退栈，一直退到左括号，并且将这些元素送到 post 中，然后弹出左括号；如果是一个运算符，这时候就得比较运算符的优先级了：
- 如果token 优先级小于等于栈顶元素，那么就退栈，存到 post 中，一直到优先级大于栈顶；
- 然后将 token 进栈；

最后，再将栈中所有元素 pop，并存到 post中。

以上就是中缀表达式转后缀表达式的过程。

有了后缀表达式后，就可以轻松的计算了：

```c
int evalRPN(Token* tokens, int len) {
    Stack nums;
    nums.top = -1;
    for (int i = 0; i < len; i++) {
        if (isType(tokens[i].type) == 0) {
            push(&nums, tokens[i]);
        } else {
            int num2 = atoi(peek(&nums).str);
            pop(&nums);
            int num1 = atoi(peek(&nums).str);
            pop(&nums);

            if (tokens[i].type == TK_PLUS) {
                char str[64] = {0};
                sprintf(str, "%u", num1 + num2);
                push(&nums, make_token(str));
            }
            if (tokens[i].type == TK_MINUS) {
                char str[64] = {0};
                sprintf(str, "%u", num1 - num2);
                push(&nums, make_token(str));
            }
            if (tokens[i].type == TK_MULTIPLY) {
                char str[64] = {0};
                sprintf(str, "%u", num1 * num2);
                push(&nums, make_token(str));
            }
            if (tokens[i].type == TK_DIVIDE) {
                char str[64] = {0};
                sprintf(str, "%u", num1 / num2);
                push(&nums, make_token(str));
            }
        }
    }
    return atoi(peek(&nums).str);
}
```

同样，这个计算过程也需要用到栈，不在累赘。

然后完善接口：
```c
static int cmd_p(char* args) {
    // 执行表达式 or 执行测试
    if (args != NULL) {
        printf("Result is %u\n", calc(args));
        return 0;
    }
    return -1; // 表示错误
}

```

然后就可以计算了：
```bash
(nemu) p 1+1+2+3
Result is 7
(nemu) p 5 + 4 * 3 / 2 - 1
Result is 10
(nemu) p 0x1+1
Result is 2
(nemu) p 0x23
Result is 35
(nemu) p 0x23 +1
Result is 36
(nemu) p 0x1+1
Result is 2
(nemu) p 0xa+1
Result is 11
(nemu) p 0xa+ $a5
Result is 10
(nemu) p 0x80100000 + ($a0 +5)*4 - (  $t1 + 8) + 1
Result is 2148532237
```
总之，除了解引用都可以完成任务。

## 三、测试

如果一个一个测试将会非常麻烦，我们利用编程语言天然地计算中缀表达式，将这个结果和我们的计算结果进行比较，如果一样，就输出 passed!。那么如何获取中缀表达式呢？可以写一个程序，让它自己生成。

程序 gen-expr.c 就实现了这样的功能：
```c
#include <assert.h>
#include <stdint.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>

// this should be enough
static char buf[65536] = {0};
static char code_buf[65536 + 128] = {};  // a little larger than `buf`
static char* code_format =
    "#include <stdio.h>\n"
    "int main() { "
    "  unsigned result = %s;"
    "  printf(\"%%u\", result); "
    "  return 0; "
    "}\n";

int pos = 0;
int choose(int n) {
    if (n < 0) {
        return -1;
    }
    return rand() % n;
}
void gen_num() {
    buf[pos++] = choose(5) + '3';  // 3~7
}
void gen(char c) {
    buf[pos++] = c;
    if (choose(2) == 1) {  // 有一半的概率插入空格
        buf[pos++] = ' ';
    }
}
void gen_rand_op() {
    int n = choose(4);
    switch (n) {
        case 0:
            buf[pos++] = '+';
            break;
        case 1:
            buf[pos++] = '-';
            break;
        case 2:
            buf[pos++] = '*';
            break;
        default:
            buf[pos++] = '*';
    }
    if (choose(2) == 1) {  // 有一半的概率插入空格
        buf[pos++] = ' ';
    }
}

static void gen_rand_expr(int n) {
    if (n == 0) {
        gen_num();
        return;
    }
    switch (choose(3)) {
        case 0:
            gen_num();
            break;
        case 1:
            gen('(');
            gen_rand_expr(n - 1);
            gen(')');
            break;
        default:
            gen_rand_expr(n - 1);
            gen_rand_op();
            gen_rand_expr(n - 1);
            break;
    }
}

int main(int argc, char* argv[]) {
    int seed = time(0);
    srand(seed);
    int loop = 1;
    if (argc > 1) {
        sscanf(argv[1], "%d", &loop);
    }
    int i;
    for (i = 0; i < loop; i++) {
        pos = 0;
        memset(buf, 0, strlen(buf));
        gen_rand_expr(5);

        sprintf(code_buf, code_format, buf);

        FILE* fp = fopen("/tmp/.code.c", "w");
        assert(fp != NULL);
        fputs(code_buf, fp);
        fclose(fp);

        int ret = system("gcc /tmp/.code.c -o /tmp/.expr");
        if (ret != 0)
            continue;

        fp = popen("/tmp/.expr", "r");
        assert(fp != NULL);

        int result;
        ret = fscanf(fp, "%d", &result);
        pclose(fp);

        printf("%u %s\n", result, buf);
    }
    return 0;
}
```

我为了防止它递归太深，就用参数进行了限制；至于中途产生的除以 0 行为，我直接禁掉了，原因很简单：
- 如果+、-、* 都没问题，那么说明算法没有问题；
- 如果一个表达式长度适中可以计算，那么长得表达式一定也可以计算，如果不能，只需要扩大 buf；
- 这是在进行算法正确性验证，因此有理由这样做。

接着，生成了如下的值：表达式后：
```txt
4 ( (7)) - (3)
3 3
5 5
6 6
11 (( 3*4- 4*5) ) +4+ ( 4*5) - (5)
4294967285 ( 5+6* 6-6+3-7*7)
7 (7)
4294967271 (( (5) - 5*6))
4294967295 6-7
32 4+ ( ((4* 7)))
5 5
4294967245 ( 5* 7+ 4) + 5*6* ( 3- 6)
3 3
2 6-4
3 (3)
33 (( 7+4) *( 6-7)* (4-( 7) ))
...
```

直接在 cmd_p()中进行验证：
```c
static int cmd_p(char* args) {
    // 执行表达式 or 执行测试
    if (args != NULL) {
        printf("Result is %u\n", calc(args));
        return 0;
    }

    FILE* file;
    char line[1024];  // 假设一行的最大长度为1024字符
    int prefixValue;
    char expression[1000];  // 存储表达式的部分

    // 打开文件
    file = fopen("/home/luyoung/ysyx-workbench/nemu/tools/gen-expr/input", "r");
    if (file == NULL) {
        perror("Failed to open file");
        return -1;
    }

    // 读
    int i = 0;
    while (fgets(line, sizeof(line), file) != NULL) {
        if (sscanf(line, "%u %[^\n]", &prefixValue, expression) == 2) {
            // printf("Number: %u\n", prefixValue);
            // printf("Expression: %s\n", expression);

            if (calc(expression) == prefixValue) {
                printf("test %d passed!\n", ++i);
            } else {
                printf("test %d not passed!\n", ++i);
            }

        } else {
            printf("Failed to parse the line.\n");
        }
    }

    // 关闭文件
    fclose(file);
    return 0;
}
```
结果如下：
```bash
(nemu) p
test 1 passed!
test 2 passed!
test 3 passed!
test 4 passed!
test 5 passed!
test 6 passed!
test 7 passed!
test 8 passed!
test 9 passed!
test 10 passed!
test 11 passed!
test 12 passed!
test 13 passed!
test 14 passed!
test 15 passed!
test 16 passed!
test 17 passed!
test 18 passed!
...
test 96 passed!
test 97 passed!
test 98 passed!
test 99 passed!
test 100 passed!
```
100 个表达式全部通过，因此可以认为表达式求值没有问题。

## 四、总结
以上的做法比较复杂，因此我打算利用系统提供的模板，用递归求值、正则匹配等重新做一遍。



