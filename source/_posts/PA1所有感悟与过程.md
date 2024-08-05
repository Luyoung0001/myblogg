---
title: PA1 感悟与过程
date: 2024-08-05 16:57:40
tags:
    - ysyx
    - PA1
    - sbd
categories:
    - ysyx
    - PA1
---

<!--more-->

## 一、单步执行
单步执行就是传入 1，让它执行一个指令就好。
```c

static int cmd_si(char* args) {
    if (args == NULL)
        cpu_exec(1);
    else
        cpu_exec((uint64_t)(atoi(args)));
    return 0;
}
```
其中源码这里接受的是一个 uint64_t 的一个类型：
```c
static void execute(uint64_t n) {
    Decode s;
    for (; n > 0; n--) {
        exec_once(&s, cpu.pc);
        g_nr_guest_inst++;
        trace_and_difftest(&s, cpu.pc);
        if (nemu_state.state != NEMU_RUNNING)
            break;
        IFDEF(CONFIG_DEVICE, device_update());
    }
}
```

但是我们写的时候，可以不用强转 64 位的数据，它会自己转化且安全。

## 二、打印寄存器

这里就直接打印就好了，系统提供了半成品接口，我们首先完善它：
```c
void isa_reg_display() {
    int len = sizeof(regs) / sizeof(regs[0]);
    for (int i = 0; i < len; i++) {
        printf("reg:%s---->0x%x\n", regs[i], cpu.gpr[i]);
    }
}
```

然后我们实现打印：
```c
static int cmd_info(char* args) {
    if (args == NULL) {
        // 打印所有信息
        isa_reg_display();
        sdb_watchpoint_display();
    } else if (strcmp(args, "r") == 0) {
        // 打印寄存器
        isa_reg_display();

    } else if (strcmp(args, "w") == 0) {
        // 打印监视点
        sdb_watchpoint_display();

    } else {
        printf("Wrong args.\n");
    }

    return 0;
}
```

上面还实现了打印监测点，后面补充。

## 三、扫描内存
这里系统提供了一个非常好的调用：
```c
word_t paddr_read(paddr_t addr, int len) {
  if (likely(in_pmem(addr))) return pmem_read(addr, len);
  IFDEF(CONFIG_DEVICE, return mmio_read(addr, len));
  out_of_bound(addr);
  return 0;
}
```

这个调用接口的作用是打印以 addr 开始的内存，长度为 len。因此，扫描内存应该这样写：
```c
static int cmd_x(char* args) {
    char* n = strtok(args, " ");
    char* baseaddr = strtok(NULL, " ");
    int len = 0;
    paddr_t addr = 0;
    sscanf(n, "%d", &len);
    sscanf(baseaddr, "%x", &addr);
    for (int i = 0; i < len; i++) {
        printf("addr:0x%x---->0x%x\n", addr, paddr_read(addr, 4));
        addr = addr + 4;
    }
    return 0;
}

```

## 四、表达式求值

这是重头戏，几乎是 PA1 的 80%的内容，因此这里得详细的记录一下。

### 1、词法分析
对于一个中缀表达式 expr，我们先将它分解成一个一个 token。具体操作是，首先定义正则规则，然后遍历expr，然后遍历正则规则，进行匹配：


```c
enum {
    TK_NOTYPE = 256,
    TK_EQ = 1,
    TK_NOTEQ = 2,
    TK_NUM = 3,
    TK_REGISTER = 4,
    TK_HEX = 5,
    TK_OR = 6,
    TK_AND = 7,
    TK_LEFT = 8,
    TK_RIGHT = 9,
    TK_LEQ = 10,
    TK_MEQ = 11

};

static struct rule {
    const char* regex;
    int token_type;
} rules[] = {

    /* TODO: Add more rules.
     * Pay attention to the precedence level of different rules.
     */

    {" +", TK_NOTYPE},  // spaces
    {"\\+", '+'},       // plus
    {"\\-", '-'},       // sub
    {"\\*", '*'},       // multi
    {"\\/", '/'},       // div

    {"\\(", TK_LEFT},   // (
    {"\\)", TK_RIGHT},  // )

    {"==", TK_EQ},     // equal
    {"<=", TK_LEQ},    // less or equal
    {">=", TK_MEQ},    // more or equal
    {"!=", TK_NOTEQ},  // !=

    {"\\|\\|", TK_OR},
    {"\\&\\&", TK_AND},
    {"\\!", '!'},

    {"\\$(ra|sp|gp|tp|t[0-6]|s[0-9]|a[0-7])", TK_REGISTER},  // register

    {"0[xX][0-9a-fA-F]+", TK_HEX},  // 0xX
    {"[0-9]*", TK_NUM},             // num
};
```
我们要识别的是表达式中常见的字符，比如'('、"=="、"!="、">=" 等等。有的字符在正则表达式中有特殊的含义，因此我们必须进行转义。

接着进行正则编译、匹配：


```c
#define NR_REGEX ARRLEN(rules)

static regex_t re[NR_REGEX] = {};

/* Rules are used for many times.
 * Therefore we compile them only once before any usage.
 */
void init_regex() {
    int i;
    char error_msg[128];
    int ret;

    for (i = 0; i < NR_REGEX; i++) {
        ret = regcomp(&re[i], rules[i].regex, REG_EXTENDED);
        if (ret != 0) {
            regerror(ret, &re[i], error_msg, 128);
            panic("regex compilation failed: %s\n%s", error_msg,
                  rules[i].regex);
        }
    }
}
```
编译是为了加快匹配速度，正则表达式的编译过程包括：
解析：正则表达式库会解析提供的模式字符串，检查其语法正确性，并理解其中的各种元字符和构造（如字符类、量词、分组等）。
构建：基于解析结果，构建一个内部表示，通常是一种状态机，例如确定性有限自动机（DFA）或非确定性有限自动机（NFA）。这个内部表示用于实际执行模式匹配。
优化：在某些情况下，正则表达式引擎可能会对生成的内部结构进行优化，以提高匹配性能。

### 2、make_token()

接着进行匹配、以及生成 tokens。生成 tokens 的过程中，我们需要做很多事情，首先就是普通的符号，比如运算符、括号等等。剩下的还有 hex、register、pointer 等。
```c
static bool make_token(char* e) {
    int position = 0;
    int i;
    regmatch_t pmatch;

    nr_token = 0;

    while (e[position] != '\0') {
        /* Try all rules one by one. */
        for (i = 0; i < NR_REGEX; i++) {
            if (regexec(&re[i], e + position, 1, &pmatch, 0) == 0 &&
                pmatch.rm_so == 0) {  // 匹配成功
                // char* substr_start = e + position;
                int substr_len = pmatch.rm_eo;

                // Log("match rules[%d] = \"%s\" at position %d with len %d:
                // %.*s",
                //     i, rules[i].regex, position, substr_len, substr_len,
                //     substr_start);

                position += substr_len;

                /* TODO: Now a new token is recognized with rules[i]. Add codes
                 * to record the token in the array `tokens'. For certain types
                 * of tokens, some extra actions should be performed.
                 */

                // 匹配
                Token temp_token;
                switch (rules[i].token_type) {
                    case '+':
                        temp_token.type = '+';
                        tokens[nr_token++] = temp_token;
                        break;
                    case '-':
                        temp_token.type = '-';
                        tokens[nr_token++] = temp_token;
                        break;
                    case '*':
                        temp_token.type = '*';
                        tokens[nr_token++] = temp_token;
                        break;
                    case '/':
                        temp_token.type = '/';
                        tokens[nr_token++] = temp_token;
                        break;

                    case '!':
                        temp_token.type = '!';
                        tokens[nr_token++] = temp_token;
                        break;
                    case TK_LEFT:
                        temp_token.type = TK_LEFT;
                        tokens[nr_token++] = temp_token;
                        break;
                    case TK_RIGHT:
                        temp_token.type = TK_RIGHT;
                        tokens[nr_token++] = temp_token;
                        break;

                    case TK_NUM:
                        tokens[nr_token].type = TK_NUM;
                        strncpy(tokens[nr_token].str, &e[position - substr_len],
                                substr_len);
                        nr_token++;
                        break;
                    case TK_REGISTER:
                        tokens[nr_token].type = TK_REGISTER;
                        strncpy(tokens[nr_token].str, &e[position - substr_len],
                                substr_len);
                        nr_token++;
                        break;
                    case TK_HEX:  // hex
                        tokens[nr_token].type = TK_HEX;
                        strncpy(tokens[nr_token].str, &e[position - substr_len],
                                substr_len);
                        nr_token++;
                        break;

                    case TK_EQ:
                        tokens[nr_token].type = TK_EQ;
                        strcpy(tokens[nr_token].str, "==");
                        nr_token++;
                        break;
                    case TK_NOTEQ:
                        tokens[nr_token].type = TK_NOTEQ;
                        strcpy(tokens[nr_token].str, "!=");
                        nr_token++;
                        break;
                    case TK_OR:  // or
                        tokens[nr_token].type = TK_OR;
                        strcpy(tokens[nr_token].str, "||");
                        nr_token++;
                        break;
                    case TK_AND:  // and
                        tokens[nr_token].type = TK_AND;
                        strcpy(tokens[nr_token].str, "&&");
                        nr_token++;
                        break;
                    case TK_LEQ:  // <=
                        tokens[nr_token].type = TK_LEQ;
                        strcpy(tokens[nr_token].str, "<=");
                        nr_token++;
                        break;
                    case TK_MEQ:  // >=
                        tokens[nr_token].type = TK_MEQ;
                        strcpy(tokens[nr_token].str, ">=");
                        nr_token++;
                        break;
                    case TK_NOTYPE:  // 空格跳过
                        break;
                    default:  // 待处理其他情况
                        printf("i = %d and No rules is com.\n", i);
                        break;
                }

                break;
            }
        }

        if (i == NR_REGEX) {
            printf("no match at position %d\n%s\n%*.s^\n", position, e,
                   position, "");
            return false;
        }
    }
    // 继续处理，将寄存器、hex、引用等预处理，将所有的数全部转化为 10 进制字符串

    // 寄存器
    for (int i = 0; i < nr_token; i++) {
        if (tokens[i].type == TK_REGISTER) {
            bool flag = false;
            int tmp = isa_reg_str2val(tokens[i].str, &flag);
            if (flag) {
                sprintf(tokens[i].str, "%u", tmp);
                tokens[i].type = TK_NUM;
            } else {
                printf("Transfrom error. \n");
                assert(0);
            }
        }
    }

    // 16进制
    for (int i = 0; i < nr_token; i++) {
        if (tokens[i].type == TK_HEX) {
            int value = strtol(tokens[i].str, NULL, 16);
            sprintf(tokens[i].str, "%u", value);
            tokens[i].type = TK_NUM;
        }
    }

    // 处理负值
    int tokens_len = nr_token;

    for (int i = 0; i < nr_token; i++) {
        // 解决 --1 == 1
        if (i == 0 && i + 2 < nr_token && tokens[i].type == '-' &&
            tokens[i + 1].type == '-' && tokens[i + 2].type == TK_NUM) {
            tokens[i].type = TK_NOTYPE;
            tokens[i + 1].type = TK_NOTYPE;
        }
        //  每次产生新的 TK_NOTYPE，及时去掉它
        for (int j = 0; j < nr_token; j++) {
            if (tokens[j].type == TK_NOTYPE) {
                for (int k = j + 1; k < nr_token; k++) {
                    tokens[k - 1] = tokens[k];
                }
                tokens_len--;
            }
        }
        // 5---1 ==> 5-1，也就是 op -- ==> op
        if (i > 0 && i + 1 < nr_token && isType(tokens[i - 1].type) == 1 &&
            tokens[i].type == '-' && tokens[i + 1].type == '-') {
            tokens[i].type = TK_NOTYPE;
            tokens[i + 1].type = TK_NOTYPE;
        }
        for (int j = 0; j < nr_token; j++) {
            if (tokens[j].type == TK_NOTYPE) {
                for (int k = j + 1; k < nr_token; k++) {
                    tokens[k - 1] = tokens[k];
                }
                tokens_len--;
            }
        }

        if ((tokens[i].type == '-' && i > 0 && tokens[i - 1].type != TK_NUM &&
             tokens[i - 1].type != TK_RIGHT && tokens[i + 1].type == TK_NUM) ||
            (tokens[i].type == '-' && i == 0)) {
            tokens[i].type =
                TK_NOTYPE;  //虽然这是一个负数标志，但是后面必须要忽略它，因此将它定义为空格类型

            // 右移一位，加上'-'
            for (int j = 31; j > 0; j--) {
                tokens[i + 1].str[j] = tokens[i + 1].str[j - 1];
            }
            tokens[i + 1].str[0] = '-';

            for (int j = 0; j < nr_token; j++) {
                if (tokens[j].type == TK_NOTYPE) {
                    for (int k = j + 1; k < nr_token; k++) {
                        tokens[k - 1] = tokens[k];
                    }
                    tokens_len--;
                }
            }
        }
    }

    nr_token = tokens_len;  // 恢复
    // 非运算
    for (int i = 0; i < nr_token; i++) {
        if (tokens[i].type == '!') {
            tokens[i].type = TK_NOTYPE;
            int tmp = atoi(tokens[i + 1].str);
            if (tmp == 0) {
                memset(tokens[i + 1].str, 0, sizeof(tokens[i + 1].str));
                tokens[i + 1].str[0] = '1';
                tokens[i + 1].type = TK_NUM;
            } else {
                memset(tokens[i + 1].str, 0, sizeof(tokens[i + 1].str));
            }
            for (int j = 0; j < nr_token; j++) {
                if (tokens[j].type == TK_NOTYPE) {
                    for (int k = j + 1; k < tokens_len; k++) {
                        tokens[k - 1] = tokens[k];
                    }
                    tokens_len--;
                }
            }
        }
    }
    nr_token = tokens_len;  // 恢复

    // 解引用
    for (int i = 0; i < nr_token; i++) {
        // 如何判断它是一个引用？在表达式语法正确的情况下：
        // 类似于 *123；3**123；(2)**3
        // 引用前面的 token 一定是一个操作符,后面一定是一个数；
        // i=0 && tokens[i].type == '*'
        if ((tokens[i].type == '*' && i > 0 &&
             isType(tokens[i - 1].type) == 1 && tokens[i + 1].type == TK_NUM) ||
            (tokens[i].type == '*' && i == 0)) {
            tokens[i].type = TK_NOTYPE;

            // 转化
            int temp = 0;
            if (tokens[i + 1].type == TK_HEX) {
                temp = strtol(tokens[i + 1].str, NULL, 16);
            } else if (tokens[i + 1].type == TK_NUM) {
                temp = atoi(tokens[i + 1].str);
            }

            // 解引用
            uintptr_t a = (uintptr_t)temp;
            int value = *((int*)a);
            sprintf(tokens[i + 1].str, "%u", value);
            tokens[i + 1].type = TK_NUM;

            for (int j = 0; j < tokens_len; j++) {
                if (tokens[j].type == TK_NOTYPE) {
                    for (int k = j + 1; k < tokens_len; k++) {
                        tokens[k - 1] = tokens[k];
                    }
                    tokens_len--;
                }
            }
        }
    }
    nr_token = tokens_len;  // 恢复

    return true;
}


```
现在有了处理好后的中缀表达式 tokens，比如从 `1+2*((2+3))-*0x9734982+--6` 到 `1+2*((2+3))-10+6`，可以看到，预处理把 16 进制、寄存器、引用等都做了处理，甚至还!运算。

### 3、计算

首先得创建栈以及栈的运操作：
```c
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
然后再做一些辅助函数，来辅助后面的计算判断过程：
```c
// 0:number
// 1:op
// 2:()
int isType(int type) {
    switch (type) {
        case TK_NUM:
            return 0;
        case '/':
        case '-':
        case '*':
        case '+':
            return 1;
        case '(':
        case ')':
            return 2;
        default:
            return -1;
    }
}
// 比较优先级
int priority(int type) {
    switch (type) {
        case TK_OR:
            return -4;
        case TK_AND:
            return -3;
        case TK_NOTEQ:
        case TK_EQ:
            return -2;
        case TK_LEQ:
        case TK_MEQ:
            return -1;

        case '+':
        case '-':
            return 0;
        case '*':
        case '/':
            return 1;
        default:
            return -5;
    }
}
```
中缀转后缀，这个算法难以理解，主要就是判断当前 token 的优先级以及栈顶元素的优先级：

```c
// 中缀转化为后缀表达式
void midToPost(Token* mid, Token* post, int token_len, int* new_len) {
    Stack ops;
    ops.top = -1;
    int j = 0;

    for (int i = 0; i < token_len; i++) {
        Token token = mid[i];
        if (token.type == TK_NUM) {
            post[j++] = token;
        } else if (token.type == TK_LEFT || isEmpty(&ops)) {
            push(&ops, token);

        } else if (token.type == TK_RIGHT) {
            while (!isEmpty(&ops) && (peek(&ops).type != TK_LEFT)) {
                post[j++] = pop(&ops);
            }
            pop(&ops);
        } else {
            while (!isEmpty(&ops) &&
                   (priority(peek(&ops).type) >= priority(token.type))) {
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
计算后缀表达式，这很简单：

```c
// 计算后缀表达式
int evalRPN(Token* tokens, int len) {
    Stack nums;
    nums.top = -1;
    for (int i = 0; i < len; i++) {
        if (tokens[i].type == TK_NUM) {
            push(&nums, tokens[i]);
        } else {
            int num2 = atoi(peek(&nums).str);
            pop(&nums);
            int num1 = atoi(peek(&nums).str);
            pop(&nums);

            Token temp = {0};

            if (tokens[i].type == '+') {
                sprintf(temp.str, "%u", num1 + num2);
                temp.type = TK_NUM;
                push(&nums, temp);
            }
            if (tokens[i].type == '-') {
                sprintf(temp.str, "%u", num1 - num2);
                temp.type = TK_NUM;
                push(&nums, temp);
            }
            if (tokens[i].type == '*') {
                sprintf(temp.str, "%u", num1 * num2);
                temp.type = TK_NUM;
                push(&nums, temp);
            }
            if (tokens[i].type == '/') {
                if (num2 == 0)
                    goto EXCEPTZION;
                else
                    sprintf(temp.str, "%u", num1 / num2);
                temp.type = TK_NUM;
                push(&nums, temp);
            }
            if (tokens[i].type == TK_EQ) {
                sprintf(temp.str, "%u", num1 == num2);
                temp.type = TK_NUM;
                push(&nums, temp);
            }
            if (tokens[i].type == TK_NOTEQ) {
                sprintf(temp.str, "%u", num1 != num2);
                temp.type = TK_NUM;
                push(&nums, temp);
            }
            if (tokens[i].type == TK_LEQ) {
                sprintf(temp.str, "%d", num1 <= num2);
                temp.type = TK_NUM;
                push(&nums, temp);
            }
            if (tokens[i].type == TK_MEQ) {
                sprintf(temp.str, "%d", num1 >= num2);
                temp.type = TK_NUM;
                push(&nums, temp);
            }

            if (tokens[i].type == TK_AND) {
                sprintf(temp.str, "%u", num1 && num2);
                temp.type = TK_NUM;
                push(&nums, temp);
            }
            if (tokens[i].type == TK_OR) {
                sprintf(temp.str, "%u", num1 || num2);
                temp.type = TK_NUM;
                push(&nums, temp);
            }
        }
    }
    return atoi(peek(&nums).str);
EXCEPTZION:
    return 10086;
}
```
我使用了 goto 语句，当一个表达式中有除 0 操作时，直接返回特定的值比如 10086来标记除0 异常。
然后是计算：
```c
Token post[1024] = {0};
word_t expr(char* e, bool* success) {
    // 清空 tokens
    memset(tokens, 0, sizeof(tokens));
    memset(post, 0, sizeof(post));
    if (!make_token(e)) {
        *success = false;
        return 0;
    }
    int len = 0;
    *success = true;
    midToPost(tokens, post, nr_token, &len);
    return evalRPN(post, len);
}
```

接着在我们的 sdb 中写上接口：
```c
static int cmd_p(char* args) {
    if (args == NULL) {
        printf("No args\n");
        return 0;
    }
    bool flag = false;
    int ans = expr(args, &flag);
    printf("%u", ans);
    return 0;
}
```
### 4、测试
这里我采用的处理方式一样，如果有除 0 操作，那么表达式的 result 未定义，但是我可以规定他为 10086，这样以便于通过测试点。其它没什么好说的：
```c

// this should be enough
static char buf[65536] = {0};
static char code_buf[65536 + 128] = {};  // a little larger than `buf`
static char* code_format =
    "#include <stdio.h>\n"
    "int main() { "
    "  int result = %s;"
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
    int n = choose(7);
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
        case 3:
            buf[pos++] = '=';
            buf[pos++] = '=';
            break;
        case 4:
            buf[pos++] = '!';
            buf[pos++] = '=';
            break;
        case 5:
            buf[pos++] = '<';
            buf[pos++] = '=';
            break;
        case 6:
            buf[pos++] = '>';
            buf[pos++] = '=';
            break;
        default:
            buf[pos++] = '/';
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
        gen_rand_expr(10);

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
        if (ret != 1) {      // 读取失败，除0 行为
            result = 10086;  // 10086 定义为 除0行为
        }
        pclose(fp);

        printf("%u %s\n", result, buf);
    }
    return 0;
}
```

为了防止生成的表达式太深，我加了参数，来递归生成的时候，可以控制深度。生成的表达式类似如下：

```txt
0 ( 3)!= 7<= ( ( 6)) !=( ( 3))-( 3)!= 5*5>=6- ( ((7)>= 3* 7) )+3
0 ( ((( 3)) *4<=4+ 3))
7 7
4 4
6 6
4 4
4 ( 4)
0 4== 3== 6
3 3
1 7<= 5!= ( 7>=7)>= 3<=7*( 5) * ( 3)>= 7<=4+7
3 3
3 3
4 ( 4)
1 5<= 4!=5!=6<= 6!=((5))<=( ( (7<=6))) == 4== 3*( (( 7))) !=7!= 4
11 3+ 6-3+ 5
4 4
4 4
0 ( ( (5)==7-5<= (( (4) ))!= ( (4) - 4- 5) <= 6== 5==(( 7+( (5)>=( 5)- 4) ) )) )
1 3!=7>=7==5!=((3) )==3!= ((7<=4<=6==7*7))>=5- 3>=6- ((5)>= ( 7) <=7<=6<= 4*3) <=( 4-5*( 4))!=(5)
14 (5== ((7)) )+ 7+ 7
...

```
测试结果如下：

```c
test 1 passed!
test 2 passed!
test 3 passed!
test 4 passed!
test 5 passed!
...
test 197 passed!
test 198 passed!
test 199 passed!
test 200 passed!
```

全部通过测试点！

## 五、检测点
监测点就是检测一个寄存器或者引用，比如地址的值，将表达式设置成监测点是没有任何意义的。

这里我取消了框架提供的 head，我认为所有的监测点都可以使用 free_ 来进行管理，而且这种管理没有任何增删链表节点的行为。速度快，稳定，不易出错、编程难度小。

```c
// free 上是潜在的空 wp，它的状态不应该发生改变
WP* new_wp() {
    WP* p = free_;
    while (p != NULL) {
        if (p->tag == false) {
            p->tag = true;
            return p;
        }
        p = p->next;
    }

    assert(0);
    return NULL;
}
```
在 new 一个监测点的时候，我们只需要在 free_上找到一个false节点，然后修改它的 tag 为true，并返回就行了。

```c
// 只需要将free_wp 的状态改为 false，然后清空一些值
void free_wp(WP* wp) {
    WP* p = free_;
    while (p != NULL) {
        if (wp->NO == p->NO) {
            p->NO = 0;
            memset(p->expr, 0, sizeof(p->expr));
            p->old_value = 0;
            p->new_value = 0;
            p->tag = false;
            break;
        }
        p = p->next;
    }
}
```
free_wp 就更简单了，只需要清空它的值，并将 tag 设置为 false。

```c
// wp 打印
void sdb_watchpoint_display() {
    WP* p = free_;
    int flag = 0;
    while (p != NULL) {
        if (p->tag == true) {
            flag = 1;
            printf(
                "Watchpoint.No: %d, expr = %s, old_value = %d, new_value = "
                "%d\n",
                p->NO, p->expr, p->old_value, p->new_value);
        }
        p = p->next;
    }
    if (flag == 0) {
        printf("No Watch Points.\n");
    }
}
```
打印的话，就在 free_ 中进行遍历，如果没有找到，输出响应的信息。
```c
// wp_delete 直接 free 掉

int delete_wp(int No) {
    WP* p = free_;
    while (p != NULL) {
        if (p->NO == No) {
            free_wp(p);
            return 0;
        }
        p = p->next;
    }
    return -1;  // 删除行为错误
}
```
删除就直接 free 掉。
```c
int create_wp(char* args) {
    WP* temp = new_wp();
    strcpy(temp->expr, args);
    bool success = false;
    int v = expr(temp->expr, &success);
    if (success == true) {
        temp->old_value = v;
        temp->tag = true;
    } else {
        printf("wrong expr() process.\n");
        return -1;
    }
    printf("Watch Point created success.\n");

    return 0;
}
```
至于创建，那就更简单了，首先 new 一个，然后设置它的值就好了。

按照文档提示，我们还需要比较观测点的新值和旧值:
```c
static void trace_and_difftest(Decode* _this, vaddr_t dnpc) {
#ifdef CONFIG_ITRACE_COND
    if (ITRACE_COND) {
        log_write("%s\n", _this->logbuf);
    }
#endif
    if (g_print_step) {
        IFDEF(CONFIG_ITRACE, puts(_this->logbuf));
    }
    IFDEF(CONFIG_DIFFTEST, difftest_step(_this->pc, dnpc));

    // 扫描监视点
    WP* p = free_;
    while (p != NULL) {
        if (p->tag == true) {
            bool success = false;
            int tmp = expr(p->expr, &success);
            if (success) {
                if (tmp != p->old_value) {
                    p->new_value = tmp;
                    nemu_state.state = NEMU_STOP;
                    printf("Not EQ: old_value:%u-->current_value:%u\n",
                           p->old_value, p->new_value);
                    return;
                }
            } else {
                printf("expr error.\n");
                assert(0);
            }
        }
        p = p->next;
    }
}
```

这里的细节是，我们必须在 new_v 和old_v 不等的时候，对这个值的 new_v 进行更新。比如：
```c
(nemu) w $t0 == 0
Watch Point created success.
(nemu) si
0x80000000: 00 00 02 97 auipc   t0, 0
0x80000000: 00 00 02 97 auipc   t0, 0
Not EQ: old_value:1-->current_value:0
(nemu) info w
Watchpoint.No: 0, expr = $t0 == 0, old_value = 1, new_value = 0
(nemu)
```
可以看到，监测点的值确实发生了改变。

## 六、Q & A

## 1、计算机可以没有寄存器吗?

如果没有寄存器，计算机理论上仍然可以工作，但它将极大地影响硬件的性能、设计和提供的编程模型。这种设计需要重大的架构调整和对程序设计方法的根本改变。

### **计算机的工作方式**

1. **基于存储器的处理器**：在没有寄存器的设计中，一个概念是完全基于存储器的处理器，如一些早期的计算机和少数现代的特殊架构，例如堆栈机。在这种设计中，所有操作（包括算术运算和逻辑运算）直接在内存上进行，而不是首先将数据加载到寄存器中。

2. **性能影响**：由于内存访问速度远慢于寄存器，这种设计显著降低了数据处理的速度。内存操作的延迟和带宽限制成为性能的主要瓶颈。

3. **简化的硬件设计**：从某种角度看，没有寄存器的处理器可能有更简单的硬件设计。例如，指令集可能更简单，因为所有指令都直接作用于内存地址。

### **对硬件提供的编程模型的影响**

1. **编程复杂性增加**：开发者需要编写更多的代码来处理常见任务，因为常用的数据和中间结果不能简单地存储在快速访问的寄存器中。这可能导致代码更加冗长且难以维护。

2. **优化挑战**：在没有寄存器的系统中，编译器和开发者面临更大的挑战，以优化程序性能，因为传统的优化策略，如寄存器分配和调度，不再适用。相反，他们可能需要依赖于优化内存访问模式和减少内存带宽需求。

3. **指令集的改变**：没有寄存器的架构可能需要一个完全不同的指令集，这些指令集直接操作内存而不是通过寄存器。这可能使得指令执行更直接，但每个指令的成本更高。

4. **并行处理限制**：寄存器在现代多核和多线程处理器中扮演着关键角色，它们允许快速上下文切换和独立的线程执行。没有寄存器，实现有效的并行处理会更加困难。

## 2、从状态机视角理解程序运行
以上一小节中1+2+...+100的指令序列为例, 尝试画出这个程序的状态机.

这个程序比较简单, 需要更新的状态只包括PC和r1, r2这两个寄存器, 因此我们用一个三元组(PC, r1, r2)就可以表示程序的所有状态, 而无需画出内存的具体状态. 初始状态是(0, x, x), 此处的x表示未初始化. 程序PC=0处的指令是mov r1, 0, 执行完之后PC会指向下一条指令, 因此下一个状态是(1, 0, x). 如此类推, 我们可以画出执行前3条指令的状态转移过程:
```txt
​ (0,x ,x )->(1,0 ,x )->(2,0 ,0 )->(3,0 ,1 )->(4,1 ,1 )->(5,1 ,2 )->(6,3 ,2 )->(7,3 ,3 )->(8,6 ,3 )->(9,6 ,4 )->(10,10 ,4 )…依次类推（两次一循环，r2+1,r1+=r2）

```

## 3、为什么printf()的输出要换行?
输出延迟：由于行缓冲的特性，输出可能会在缓冲区中停留，直到程序结束或缓冲区满，这可能会导致用户界面上的显示延迟。

## 4、表达式生成器如何获得C程序的打印结果?
`popen()` 函数是一个在 POSIX 系统上常用的标准库函数，它允许你从一个普通的 C 程序中执行另一个程序，并且可以捕获被执行程序的输出或向其输入数据。该函数执行一个命令，并创建一个进程，并通过一个管道（pipe）读取或写入该进程的标准输入/输出。

### 函数原型和行为
`popen()` 的原型如下：
```c
FILE *popen(const char *command, const char *type);
```
- **command**：一个 C 字符串，包含要执行的 shell 命令。
- **type**：一个 C 字符串，通常是 `"r"` 或 `"w"`。当 `type` 是 `"r"` 时，你可以从返回的 `FILE` 指针读取执行命令的输出；当 `type` 是 `"w"` 时，你可以向执行命令的输入写入数据。

`popen()` 返回一个 `FILE` 指针，就像 `fopen()` 一样，这允许你使用标准的 I/O 函数如 `fread()`, `fwrite()`, `fgets()`, `fputs()`, `fscanf()` 等来读写数据。

### 使用 `popen()` 读取命令输出的例子
```c
fp = popen("/tmp/.expr", "r");
assert(fp != NULL);

int result;
ret = fscanf(fp, "%d", &result);
if (ret != 1) {      // 读取失败，除0 行为
    result = 10086;  // 10086 定义为 除0行为
}
pclose(fp);
```
这段代码执行了以下操作：
1. 使用 `popen()` 执行位于 `/tmp/.expr` 的程序，并打开一个管道来读取它的输出。
2. 检查 `fp` 是否为 `NULL`，以确保命令成功启动。
3. 尝试从命令的输出中读取一个整数值。
4. 如果 `fscanf()` 返回值不是 `1`，意味着没有成功读取一个整数，这可能是因为程序没有输出，或输出了非数字内容，或程序可能因错误（如除以零）而未输出任何内容。
5. 最后，使用 `pclose()` 关闭与这个管道相关联的文件指针，`pclose()` 还会等待命令终止并返回命令的终止状态。


## 5、选择无符号类型进行运算有几个优点：

确定性：无符号类型的行为是确定和可预测的，这使得它们在需要精确控制数据运算时非常有用，特别是在嵌入式编程或低级编程中。
有效利用范围：无符号类型可以有效地使用所有的位来表示数据，提供了比相同位宽的有符号类型更大的正数范围。
简化逻辑：在某些应用中，如计数器、数组索引等，负数是没有意义的。使用无符号类型可以自然地避免负数，简化错误处理，还有就是在除0判断时能够省略正无穷和负无穷。

## 6、除0的确切行为
如果生成的表达式有除0行为, 你编写的表达式生成器的行为又会怎么样呢?
无所谓，我会特殊处理。

## 7、框架代码中定义wp_pool等变量的时候使用了关键字static, static在此处的含义是什么? 为什么要在此处使用它?

在 C 语言中，`static` 关键字用于变量和函数具有多种用途，其中一种用途是控制变量的作用域和存储期。具体来说，在框架代码中使用 `static` 关键字定义 `wp_pool` 等变量时，其目的和效果包括：

### 1. **局部化作用域**
当 `static` 关键字用于全局变量时，它限制了变量的作用域只在定义它的文件中。这意味着 `static` 全局变量虽然在程序的整个运行期间都存在，但只能被同一个源文件内的函数访问。这有助于避免在其他文件中不小心使用同名变量引起的命名冲突。

### 2. **保持持久存储**
`static` 关键字还保证变量在程序启动时初始化（通常初始化为零，除非另有指定），并在程序结束前一直保持其值。这与自动变量（如局部变量）不同，自动变量只在函数调用期间存在。

### 为什么在此处使用它？

1. **封装**：在模块化编程中，封装是一个重要概念。通过使用 `static` 关键字，`wp_pool` 变量被封装在其定义的源文件中，其他源文件无法直接访问它。这有助于隐藏实现细节，使得模块的接口与实现分离，从而降低模块间的耦合。

2. **避免命名冲突**：在大型项目中，尤其是涉及多个开发者的项目中，全局变量如果没有适当的命名空间控制，很容易造成命名冲突。`static` 提供了一种简单的命名空间控制方法，确保全局变量不会与其他源文件中的变量冲突。

3. **控制访问**：使用 `static` 关键字可以控制变量的访问级别，仅允许定义它们的源文件内部的代码访问这些变量。这不仅有助于封装，也增加了代码的安全性，因为其他部分的代码无法意外地修改这些变量的状态。

但是，由于要在 cpu-exec.c中要使用watchpoint.c中定义的 static 类型数据，这是不可能的，因此我取消了 static，换做 extern 来共享。

## 8、x86的int3指令不带任何操作数, 操作码为1个字节, 因此指令的长度是1个字节. 这是必须的吗?

是的，这是必须的，因为它可以修改任何长度的指令。如果 int3 是两字节，显然对于 1 字节的指令束手无策，若要换，会覆盖后面的指令。

## 9、 假设有一种x86体系结构的变种my-x86, 除了int3指令的长度变成了2个字节之外, 其余指令和x86相同. 在my-x86中, 上述文章中的断点机制还可以正常工作吗? 为什么?

不能，正如上述所说，对于 1 字节指令打断点时，会覆盖后面的指令。因此 int3 必须设计为单指令。


## 10、模拟器(Emulator)和调试器(Debugger)有什么不同? 更具体地, 和NEMU相比, GDB到底是如何调试程序的?

这两个几乎是两个概念，调试器是主进程控制子进程，给子进程发送一些命令，比如通过控制 ptrace()中的一些参数来控制子进程的行为。模拟器则是让原本设计运行在一个系统上的程序能够在另一个系统上执行。

我之前看到过一个小项目，然后重构了它，是关于一个 C++ 版本的 mini dubugger，这是我的项目地址：[mydebugger](https://github.com/Luyoung0001/mydebugger)。

事实上，GDB的原理比较复杂，但最重要的还是 ptrace()以及 dwarf 文件解析。通过这些信息，来生成有用的调试信息以及控制子进程（被调试的程序）的行为。比如关于中断int3 的处理：
```cpp
void BreakPoint::enable() {
    auto data = ptrace(PT_READ_I, m_pid, reinterpret_cast<caddr_t>(m_addr), 0);
    m_saved_data =
        static_cast<uint8_t>(data & 0xff);  // 拿到最低的一个字节（指令）
    uint64_t int3 = 0xcc;
    uint64_t data_with_int3 =
        ((data & ~0xff) | int3);  // 得到被修改的最低的一个字节（指令）
    ptrace(PT_WRITE_I, m_pid, reinterpret_cast<caddr_t>(m_addr),
           data_with_int3);  // 修改

    m_enabled = true;
}

void BreakPoint::disable() {
    auto data = ptrace(PT_READ_I, m_pid, reinterpret_cast<caddr_t>(m_addr), 0);
    auto restored_data = ((data & ~0xff) | m_saved_data);
    ptrace(PT_WRITE_I, m_pid, reinterpret_cast<caddr_t>(m_addr), restored_data);
    m_enabled = false;
}

```

这里之所以要拿最低字节，是因为 int3 指令必须替换为每一条指令的起始一个字节，但是 ptrace()取出来的指令的字节是逆序的。低地址的内存单元数据放在了数组的最低位，这是小端字节序。
















