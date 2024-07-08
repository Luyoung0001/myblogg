---
title: Design a Dynamic Expansion Buffer (strbuf) with C Language
date: 2023-01-01 19:23:31
tags:
    - c语言
    - 开发语言
categories: Tests for Basic Knowledge of C
---

<!--more-->

## Here is it!
```c
#include <ctype.h>
#include "strbuf.h"
typedef struct strbuf strbuf;
/*Initialize the structure. The second parameter can be zero or a bigger number
to allocate memory, in case you want to prevent further reallocs.*/
void strbuf_init(struct strbuf* sb, size_t alloc) {
    sb->len = 0;
    sb->alloc = alloc;
    if (alloc)
        sb->buf = (char*)malloc(sizeof(char) * alloc);
}
/*Release a string buffer and the memory it used. You should not use the string
 * buffer after using this function, unless you initialize it again.*/
void strbuf_release(struct strbuf* sb) {
    if (sb->alloc > 0) {
        free(sb->buf);       // release the buf
        strbuf_init(sb, 0);  // init it again
    }
    /*
    strbuf sb;
    strbuf_init(&sb,10);
    sb->len = 5;
    sb->buf = "hello"
    strbuf_release(&sb);
    */
}
/*Determine the amount of allocated but unused memory.*/
size_t strbuf_avail(const struct strbuf* sb) {
    // '\0'
    return (size_t)(sb->alloc - sb->len - 1);
}
/*Set the length of the buffer to a given value. This function does not allocate
 * new memory, so you should not perform a strbuf_setlen() to a length that is
 * larger than len + strbuf_avail(). strbuf_setlen() is just meant as a please
 * fix invariants from this strbuf I just messed with.*/
void strbuf_setlen(struct strbuf* sb, size_t len) {
    if (len > sb->len + strbuf_avail(sb))
        return;
    sb->len = len;
    sb->buf[len] = '\0';
}
/*Attach a string to a buffer. You should specify the string to attach, the
current length of the string and the amount of allocated memory. The amount must
be larger than the string length, because the string you pass is supposed to be
a NUL-terminated string. This string must be malloc()ed, and after attaching,
the pointer cannot be relied upon anymore, and neither be free()d directly.*/
void strbuf_attach(struct strbuf* sb, void* str, size_t len, size_t alloc) {
    sb->alloc = alloc;
    sb->len = len;
    sb->buf = (char*)str;
}
/*Swap the contents of two string buffers.*/
void strbuf_swap(struct strbuf* a, struct strbuf* b) {
    struct strbuf temp = *a;
    *a = *b;
    *b = temp;
}
/*Detach the string from the strbuf and returns it; you now own the storage the
 * string occupies and it is your responsibility from then on to release it with
 * free(3) when you are done with it.*/
char* strbuf_detach(struct strbuf* sb, size_t* sz) {
    *sz = sb->alloc;
    return sb->buf;
}
/*Compare two buffers. Returns an integer less than, equal to, or greater than
 * zero if the first buffer is found, respectively, to be less than, to match,
 * or be greater than the second buffer.*/
int strbuf_cmp(const struct strbuf* first, const struct strbuf* second) {
    if (first->len > second->len) {
        return 1;
    } else if (first->len < second->len) {
        return -1;
    } else {
        return 0;
    }
}
/*Empty the buffer by setting the size of it to zero.*/
void strbuf_reset(struct strbuf* sb) {
    (*sb).len = 0;
    *(*sb).buf = '\0';
}

// Part 2B

/*Ensure that at least this amount of unused memory is available after len. This
 * is used when you know a typical size for what you will add and want to avoid
 * repetitive automatic resizing of the underlying buffer. This is never a
 * needed operation, but can be critical for performance in some cases.*/
void strbuf_grow(struct strbuf* sb, size_t extra) {
    // int new_alloc = sb->alloc + extra;
    // sb->alloc = new_alloc;
    // char* ret = (char*)malloc(sizeof(char) * new_alloc);
    // for (int i = 0; i < sb->len; i++) {
    //     ret[i] = sb->buf[i];
    // }
    // ret[sb->len] = '\0';
    // sb->buf = ret;
    sb->buf = (char*)realloc(sb->buf, sb->len + extra + 1);
    sb->alloc = sb->len + extra + 1;
}
/*Add data of given length to the buffer.*/
void strbuf_add(struct strbuf* sb, const void* data, size_t len) {
    if (len == 0) {
        return;
    } else if (sb->len + len >= sb->alloc) {
        strbuf_grow(sb, len);
    }
    memcpy(sb->buf + sb->len, data, len);
    sb->len += len;
    sb->buf[sb->len] = '\0';
}
/*Add a single character to the buffer.*/
void strbuf_addch(struct strbuf* sb, int c) {
    // change ch into a string
    char* s = (char*)malloc(sizeof(char));
    s[0] = c;
    strbuf_add(sb, s, 1);
    free(s);
}
/*Add a NUL-terminated string to the buffer.*/
void strbuf_addstr(struct strbuf* sb, const char* s) {
    // counting the length of s
    int p = 0;
    while (s[p] != '\0') {
        p++;
    }
    strbuf_add(sb, s, p);
}
/*Copy the contents of an other buffer at the end of the current one.*/
void strbuf_addbuf(struct strbuf* sb, const struct strbuf* sb2) {
    char* s1 = sb2->buf;
    strbuf_addstr(sb, s1);
}

/*Remove the bytes between pos..pos+len and replace it with the given data.*/
void strbuf_splice(struct strbuf* sb,
                   size_t pos,
                   size_t len,
                   const void* data,
                   size_t dlen) {
    if (dlen >= len)
        strbuf_grow(sb, dlen - len);
    memmove(sb->buf + pos + dlen, sb->buf + pos + len, sb->len - pos - len);
    /* it is better than memcpy()

    const char dest[] = "oldstring";
    memmove(&dest[1], &dest[2], 4); // odstrring
    */
    // 01234567 2 3 aaa 3
    // 01234/567 01234/567 -->01234567

    memcpy(sb->buf + pos, data, dlen);
    // 01aaa567
    strbuf_setlen(sb, sb->len + dlen - len);
    // 01aaa567
}
/*Insert data to the given position of the buffer. The remaining contents will
 * be shifted, not overwritten.*/
void strbuf_insert(struct strbuf* sb,
                   size_t pos,
                   const void* data,
                   size_t len) {
    strbuf_splice(sb, pos, 0, data, len);
}
/*Remove given amount of data from a given position of the buffer.*/
void strbuf_remove(struct strbuf* sb, size_t pos, size_t len) {
    strbuf_splice(sb, pos, len, NULL, 0);
}
// Part 2C
/*Strip whitespace from the front of a string.*/
void strbuf_ltrim(struct strbuf* sb) {
    char* b = sb->buf;
    while (sb->len > 0 && isspace(*b)) {
        b++;
        sb->len--;
    }
    memmove(sb->buf, b, sb->len);
    sb->buf[sb->len] = '\0';
}
/*Strip whitespace from the end of a string.*/
void strbuf_rtrim(struct strbuf* sb) {
    while (sb->len > 0 && isspace((unsigned char)sb->buf[sb->len - 1])) {
        sb->len--;
    }
    sb->buf[sb->len] = '\0';
}
/*Read the contents of a given file descriptor. The third argument can be used
 * to give a hint about the file size, to avoid reallocs.*/
ssize_t strbuf_read(struct strbuf* sb, int fd, size_t hint) {
    FILE* file = fdopen(fd, "r");
    char ch;
    if ((ch = fgetc(file)) == EOF)
        return sb->len;
    sb->buf[sb->len++] = ch;

    // big enough for the file
    sb->alloc <<= 13;
    sb->buf = (char*)realloc(sb->buf, sizeof(char) * (sb->alloc));
    while ((ch = fgetc(file)) != EOF) {
        sb->buf[sb->len] = ch;
        sb->len++;
    }
    sb->buf[sb->len] = '\0';
    return sb->len;
}

/*Read a line from a FILE* pointer. The second argument specifies the line
 * terminator character, typically \n.*/
int strbuf_getline(struct strbuf* sb, FILE* fp) {
    int ch;
    while ((ch = fgetc(fp)) != EOF) {
        if (ch != '\n') {
            strbuf_grow(sb, 1);
            sb->buf[sb->len++] = ch;
        } else {
            break;
        }
    }
    sb->buf[sb->len] = '\0';
    return 1;
}
// CHALLENGE

struct strbuf** strbuf_split_buf(const char* str,
                                 size_t len,
                                 int terminator,
                                 int max) {
    strbuf** ans = (strbuf**)malloc(sizeof(strbuf*) * (max + 1));

    /* this is a possible way:
    char *p = (char*)str,*q = (char*)(str+len-1); // points to the last elem
    // off the heads and tails
    while (*p == terminator) {
        p++;
    }
    while (*q == terminator) {
        q--;
    }
    int pointer = 0;
    for (char* t = p; t <= q+1; t++) {
        if (*t == terminator || t == q+1) {
    */

    // set 2 pointers
    char *p = (char*)str,
         *q = (char*)(str + len);  // points to the last elem's next position
    // off the heads
    while (*p == terminator) {
        p++;
    }
    int pointer = 0;
    for (char* t = p; t <= q; t++) {
        if (*t == terminator || t == q) {
            int length = t - p;  // the last strbuf->len could be right, so q
                                 // must points to the last elem's next position
            ans[pointer] = (strbuf*)malloc(sizeof(strbuf));
            // value it
            ans[pointer]->len = length;
            ans[pointer]->alloc = length + 1;
            ans[pointer]->buf =
                (char*)malloc(sizeof(char) * (length + 1));  // 1 for '\0'

            memcpy(ans[pointer]->buf, p, length);
            ans[pointer++]->buf[length] = '\0';
            while (*t == terminator && t <= q) {
                t++;
            }
            // set a new start
            p = t;
        }
        if (pointer == max)
            break;
    }
    ans[pointer] = NULL;
    return ans;
}
bool strbuf_begin_judge(char* target_str, const char* str, int len) {
    if (len == 0)
        return true;
    if (strlen(str) > len) {
        return false;
    }
    for (int i = 0; i < strlen(str); i++) {
        if (str[i] != target_str[i]) {
            return false;
        }
    }
    return true;
}
char* strbuf_get_mid_buf(char* target_buf, int begin, int end, int len) {
    if (begin >= len || end >= len)
        return NULL;
    char* ret = (char*)malloc(sizeof(char) * (end - begin + 1));

    memcpy(ret, target_buf + begin, end - begin);
    ret[end - begin] = '\0';
    return ret;
} // for success.
```
*This text is over, thank you for reading~*
