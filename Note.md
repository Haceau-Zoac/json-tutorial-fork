# 从零开始的 JSON 库教程
## 一　启程
### 1　JSON 是什么
[JSON](https://www.ecma-international.org/publications/files/ECMA-ST/ECMA-404.pdf)（JavaScript Object Notation）是用于数据交换的文本格式。
JSON 树状结构且只包含 6 种数据类型：
- null
- boolean：true/false
- number：一般的浮点数
- string："..."
- array：[...]
- object：{...}

JSON 库有 3 个需求：
1. 把 JSON 文本解析成树状结构（parse）。
2. 提供接口访问数据（access）。
3. 把数据结构转换回 JSON 文本（stringify）。

### 2　搭建编译环境
头文件（header file）含有对外的类型与 API 声明。
实现文件（implementation file）含有内部类型声明和函数实现，会编译成库。
一般在 CMake 工程内新建 build 文件夹保存生成文件，-DCMAKE_BUILD_TYPE=Debug 标志可生成 Debug 工程。

### 3　头文件与 API 设计
C 语言使用 `#include` 引入头文件的类型与函数声明。
头文件使用宏进行 include 防范（include guard）。宏名唯一，一般以 `_H__` 作为后缀，命名为 `项目名_目录_文件名_H__` 的形式。
枚举类型（enumeration type）一般使用项目简写作为标识符前缀，通常全大写，而类型及函数通常全小写。这里声明 `lept_type` 类型表示 JSON 类型。
```c
typedef enum { LEPT_NULL, LEPT_FALSE, LEPT_TRUE, LEPT_NUMBER, LEPT_STRING, LEPT_ARRAY, LEPT_OBJECT } lept_type;
```
JSON 的数据实现为数，节点用 `lept_value` 表示。`typedef` 简化定义。
```c
typedef struct {
  lept_type type;
} lept_value;
```
解析 API：
```c
int lept_parse(lept_value* v, char const* json);
```
JSON 文本使用 C 字符串（空结尾字符串/null-terminated string），使用 `char const*` 类型防止改变。
返回值为下列枚举，无错返回 `LEPT_PARSE_OK`。
```c
enum {
  LEPT_PARSE_OK = 0,
  LEPT_PARSE_EXPECT_VALUE,
  LEPT_PARSE_INVALID_VALUE,
  LEPT_PARSE_ROOT_NOT_SINGULAR
};
```
访问函数：
```c
lept_type lept_get_type(lept_value const* v);
```
### 4　JSON 语法子集
```ABNF
JSON-text = ws value ws
ws = *(%x20 / %x09 / %x0A / %x0D)
value = null / false / true
null  = "null"
false = "false"
true  = "true"
```
`%xhh` 表示 16 进制表示的字符，`/` 表示多选一，`*` 表示零或多个，`()` 用于分组。
JSON 文本由 3 部分组成，空白（whitespace）+ 值 + 空白。
空白是零或多个空格（space U+0020）、制表符（tab U+0009）、换行符（LF U+000A）、回车符（CR U+000D）构成的。
值包括 null、false 或 true，它们有对应的字面量（literal）。
当出现 JSON 语法错误，我们将会返回错误码：
- `LEPT_PARSE_EXPECT_VALUE`：只含有空白。
- `LEPT_PARSE_ROOT_NOT_SINGULAR`：在值后面的空白后还有字符。
- `LEPT_PARSE_INVALID_VALUE`：值不是三种字面值。

### 5　单元测试
单元测试（unit testing）等测试为自动测试，可确保其他人修改代码后，源代码依旧正确（回归测试/regression testing）。
测试驱动开发（test-driven development，TDD）：
1. 加入一个测试。
2. 运行所有测试，新测试应该失败。
3. 编写实现代码。
4. 运行测试，若失败返回 3。
5. 重构代码。
6. 回到 1。

### 6　宏的编写技巧
反斜杠代表该行未结束，串接下一行。
宏内语句（statement）多于一个时，使用 `do { ... } while(0)` 包裹为单个语句。

### 7　实现解析器
使用 `lept_context` 结构体传递参数：
```c
typedef struct {
  char const* json;
} lept_context;

/* .. */

/* JSON-text = ws value ws */
int lept_parse(lept_value* v, char const* json) {
  lept_context c;
  assert(v != NULL);
  c.json = json;
  v->type = LEPT_NULL;
  lept_parse_whitespace(&c);
  return lept_parse_value(&c, v);
}
```
若 `lept_parse()` 失败，则会置 `v` 为 `null` 类型。
leptjson 是一个手写递归下降解析器（recursive descent parser）。因为 JSON 语法特别简单，所以无需分词器（tokenizer），只需检测字符。
- n -> null
- t -> true
- f -> false
- " -> string
- 0-9/- -> number
- [ -> array
- { -> object

```c
#define EXPECT(c, ch) do { assert(*c->json == (ch)); c->json++; } while(0)

/* ws = *(%x20 / %x09 / %x0A / %x0D) */
static void lept_parse_whitespace(lept_context* c) {
    const char *p = c->json;
    while (*p == ' ' || *p == '\t' || *p == '\n' || *p == '\r')
        p++;
    c->json = p;
}

/* null  = "null" */
static int lept_parse_null(lept_context* c, lept_value* v) {
    EXPECT(c, 'n');
    if (c->json[0] != 'u' || c->json[1] != 'l' || c->json[2] != 'l')
        return LEPT_PARSE_INVALID_VALUE;
    c->json += 3;
    v->type = LEPT_NULL;
    return LEPT_PARSE_OK;
}

/* value = null / false / true */
static int lept_parse_value(lept_context* c, lept_value* v) {
    switch (*c->json) {
        case 'n':  return lept_parse_null(c, v);
        case '\0': return LEPT_PARSE_EXPECT_VALUE;
        default:   return LEPT_PARSE_INVALID_VALUE;
    }
}
```

### 8　关于断言
断言（assertion）是 C 语言常用的防御式编程方式，减少错误。在函数开始检测参数，在函数后检测上下文是否正确。
C 语言有 `assert()` 宏（`assert.h`）进行断言。release（`NDEBUG`）不会检测；debug（不定义 `NDEBUG`）会检测条件是否为真，不为真则崩溃。
含副作用的代码不可断言，影响 release 与 debug 行为。

### 9　总结与练习

### 10　常见问答
- 命名项目时，google 看名称是否独特。
- `__LINE__` 宏表示编译时的行号。

## 二　解析数字
### 1　初探重构
TDD 的步骤之一是重构（refactoring）：在不改变代码外在行为的前提下，对代码作出修改，以改进程序的内部结构。
TDD 的过程中，我们可能过于关注正确率而忽视了其他方面，因此，我们之后要进行重构。
`lept_parse_null()`、`lept_parse_true()`、`lept_parse_false()` 过于相像，违反了 DRY（Don't Repeat Yourself）原则。
单元测试也有重复代码，例如 `test_parse_invalid_value()` 中，每次测试不合法的 JSON 值，都有 4 行相似代码，可以使用宏来简化：
```c
#define TEST_ERROR(error, json) \
  do { \
    lept_value v; \
    v.type = LEPT_FALSE; \
    EXPECT_EQ_INT(error, lept_parse(&v, json)); \
    EXPECT_EQ_INT(LEPT_NULL, lept_get_type(&v)); \
  } while (0)

static void test_parse_expect_value() {
  TEST_ERROR(LEPT_PARSE_EXPECT_VALUE, "");
  TEST_ERROR(LEPT_PARSE_EXPECT_VALUE, " ");
}
```

### 2　JSON 数字语法
```ABNF
number = [ "-" ] int [ frac ] [ exp ]
int = "0" / digit1-9 *digit
frac = "." 1*digit
exp = ("e" / "E") ["-" / "+"] 1*digit
```
number 是十进制，包括负号、整数、小数、指数，只有整数必需。
整数部分若 0 开头，只能是单个 0；以 1-9 开头，后面可接任意数量的数字（0-9）。
小数部分是小数点后一或多个数字（0-9）。
JSON 可用科学计数法，以 E 或 e 开始，接着可有正负号，然后是一或多个数字。
![](tutorial02/images/number.png)

### 3　数字表示方式
以 `double` 存储。为 `lept_value` 添加成员：
```c
typedef struct lept_value {
  double n;
  lept_type type;
} lept_value;
```
仅当 `type == LEPT_NUMBER` 时，`n` 才表示数值。
```c
double lept_get_number(lept_value const* v) {
  assert(v != NULL && v->type == LEPT_NUMBER);
  return v->n;
}
```
使用者应确保类型正确。

### 4　单元测试
```c
#define TEST_NUMBER(expect, json) \
  do { \
    lept_value v; \
    EXPECT_EQ_INT(LEPT_PARSE_OK, lept_parse(&v, json)); \
    EXPECT_EQ_INT(LEPT_NUMBER, lept_get_type(&v)); \
    EXPECT_EQ_DOUBLE(expect, lept_get_number(&v)); \
  } while(0)

static void test_parse_number() {
  TEST_NUMBER(0.0, "0");
  TEST_NUMBER(0.0, "-0");
  TEST_NUMBER(0.0, "-0.0");
  TEST_NUMBER(1.0, "1");
  TEST_NUMBER(-1.0, "-1");
  TEST_NUMBER(1.5, "1.5");
  TEST_NUMBER(-1.5, "-1.5");
  TEST_NUMBER(3.1416, "3.1416");
  TEST_NUMBER(1E10, "1E10");
  TEST_NUMBER(1e10, "1e10");
  TEST_NUMBER(1E+10, "1E+10");
  TEST_NUMBER(1E-10, "1E-10");
  TEST_NUMBER(-1E10, "-1E10");
  TEST_NUMBER(-1e10, "-1e10");
  TEST_NUMBER(-1E+10, "-1E+10");
  TEST_NUMBER(-1E-10, "-1E-10");
  TEST_NUMBER(1.234E+10, "1.234E+10");
  TEST_NUMBER(1.234E-10, "1.234E-10");
  TEST_NUMBER(0.0, "1e-10000"); /* must underflow */
}
```
以及不合法用例：
```c
static void test_parse_invalid_value() {
  /* ... */
  /* invalid number */
  TEST_ERROR(LEPT_PARSE_INVALID_VALUE, "+0");
  TEST_ERROR(LEPT_PARSE_INVALID_VALUE, "+1");
  TEST_ERROR(LEPT_PARSE_INVALID_VALUE, ".123"); /* at least one digit before '.' */
  TEST_ERROR(LEPT_PARSE_INVALID_VALUE, "1.");   /* at least one digit after '.' */
  TEST_ERROR(LEPT_PARSE_INVALID_VALUE, "INF");
  TEST_ERROR(LEPT_PARSE_INVALID_VALUE, "inf");
  TEST_ERROR(LEPT_PARSE_INVALID_VALUE, "NAN");
  TEST_ERROR(LEPT_PARSE_INVALID_VALUE, "nan");
}
```

### 5　十进制转换至二进制
`strtod()` 可转换十进制数字至 `double`。
```c
#include <stdlib.h>  /* NULL, strtod() */

static int lept_parse_number(lept_context* c, lept_value* v) {
    char* end;
    /* \TODO validate number */
    v->n = strtod(c->json, &end);
    if (c->json == end)
        return LEPT_PARSE_INVALID_VALUE;
    c->json = end;
    v->type = LEPT_NUMBER;
    return LEPT_PARSE_OK;
}
```
value 语法变为：
```ABNF
value = null / false / true / number
```
`lept_parse_number()` 内部会检测值，所以可交余下情况至 `lept_parse_number()`：
```c
static int lept_parse_value(lept_context* c, lept_value* v) {
  switch (*c->json) {
    case 't':  return lept_parse_true(c, v);
    case 'f':  return lept_parse_false(c, v);
    case 'n':  return lept_parse_null(c, v);
    default:   return lept_parse_number(c, v);
    case '\0': return LEPT_PARSE_EXPECT_VALUE;
  }
}
```

## 三　解析字符串
### 1　JSON 字符串语法
```ABNF
string = quotation-mark *char quotation-mark
char = unescaped /
   escape (
       %x22 /          ; "    quotation mark  U+0022
       %x5C /          ; \    reverse solidus U+005C
       %x2F /          ; /    solidus         U+002F
       %x62 /          ; b    backspace       U+0008
       %x66 /          ; f    form feed       U+000C
       %x6E /          ; n    line feed       U+000A
       %x72 /          ; r    carriage return U+000D
       %x74 /          ; t    tab             U+0009
       %x75 4HEXDIG )  ; uXXXX                U+XXXX
escape = %x5C          ; \
quotation-mark = %x22  ; "
unescaped = %x20-21 / %x23-5B / %x5D-10FFFF
```

### 2　字符串表示
JSON 支持空字符（`\0`），所以不能用 `\0` 作为结束——我们分配内存来储存解析后的字符与数目。
`lept_value` 实际上是一个变体类型（variant type），通过 `type` 来决定它的类型，我们使用 `union` 节省内存。
```c
typedef struct {
    union {
        struct { char* s; size_t len; }s;  /* string */
        double n;                          /* number */
    }u;
    lept_type type;
}lept_value;
```

### 3　内存管理
我们使用 `stdlib.h` 中的 `malloc()`、`realloc()`、`free()` 来管理内存。
```c
// h
#define lept_set_null(v) lept_free(v)

int lept_get_boolean(const lept_value* v);
void lept_set_boolean(lept_value* v, int b);

double lept_get_number(const lept_value* v);
void lept_set_number(lept_value* v, double n);

const char* lept_get_string(const lept_value* v);
size_t lept_get_string_length(const lept_value* v);
void lept_set_string(lept_value* v, const char* s, size_t len);
```
```c
// c
#define lept_init(v) do { (v)->type = LEPT_NULL; } while(0)

void lept_set_string(lept_value* v, const char* s, size_t len) {
    assert(v != NULL && (s != NULL || len == 0));
    lept_free(v);
    v->u.s.s = (char*)malloc(len + 1);
    memcpy(v->u.s.s, s, len);
    v->u.s.s[len] = '\0';
    v->u.s.len = len;
    v->type = LEPT_STRING;
}

void lept_free(lept_value* v) {
    assert(v != NULL);
    if (v->type == LEPT_STRING)
        free(v->u.s.s);
    v->type = LEPT_NULL;
}
```
```c
static void test_access_string() {
    lept_value v;
    lept_init(&v);
    lept_set_string(&v, "", 0);
    EXPECT_EQ_STRING("", lept_get_string(&v), lept_get_string_length(&v));
    lept_set_string(&v, "Hello", 5);
    EXPECT_EQ_STRING("Hello", lept_get_string(&v), lept_get_string_length(&v));
    lept_free(&v);
}
```
### 4　缓冲区与堆栈
解析字符串时，缓冲区大小无法预知，因此采用动态数组（dynamic array）自动扩展。
每次解析 JSON 时只需一个动态数组，并且先进后出，所以要一个动态栈（stack）。
```c
typedef struct {
    const char* json;
    char* stack;
    size_t size, top;
}lept_context;
```
`size` 是堆栈容量，`top` 是栈顶（我们会扩展 `stack`，所以不能用指针）。
```c
#ifndef LEPT_PARSE_STACK_INIT_SIZE
#define LEPT_PARSE_STACK_INIT_SIZE 256
#endif

int lept_parse(lept_value* v, const char* json) {
    lept_context c;
    int ret;
    assert(v != NULL);
    c.json = json;
    c.stack = NULL;        /* <- */
    c.size = c.top = 0;    /* <- */
    lept_init(v);
    lept_parse_whitespace(&c);
    if ((ret = lept_parse_value(&c, v)) == LEPT_PARSE_OK) {
        /* ... */
    }
    assert(c.top == 0);    /* <- */
    free(c.stack);         /* <- */
    return ret;
}

static void* lept_context_push(lept_context* c, size_t size) {
    void* ret;
    assert(size > 0);
    if (c->top + size >= c->size) {
        if (c->size == 0)
            c->size = LEPT_PARSE_STACK_INIT_SIZE;
        while (c->top + size >= c->size)
            c->size += c->size >> 1;  /* c->size * 1.5 */
        c->stack = (char*)realloc(c->stack, c->size);
    }
    ret = c->stack + c->top;
    c->top += size;
    return ret;
}

static void* lept_context_pop(lept_context* c, size_t size) {
    assert(c->top >= size);
    return c->stack + (c->top -= size);
}
```
使用 `#ifndef` 是为了用户可自行设置宏。

### 5　解析字符串
```c
#define PUTC(c, ch) do { *(char*)lept_context_push(c, sizeof(char)) = (ch); } while(0)

static int lept_parse_string(lept_context* c, lept_value* v) {
    size_t head = c->top, len;
    const char* p;
    EXPECT(c, '\"');
    p = c->json;
    for (;;) {
        char ch = *p++;
        switch (ch) {
            case '\"':
                len = c->top - head;
                lept_set_string(v, (const char*)lept_context_pop(c, len), len);
                c->json = p;
                return LEPT_PARSE_OK;
            case '\0':
                c->top = head;
                return LEPT_PARSE_MISS_QUOTATION_MARK;
            default:
                PUTC(c, ch);
        }
    }
}
```

## 四　Unicode
### 1　Unicode
ASCII 将 128 个字符映射至整数 0~127。
Unicode 的范围是 0~0x10FFFF，码点记作 U+XXXX（16 进位数字）。
### 2　需求
非转义字符直接复制。
`\uXXXX` 的字符需要：
1. 解析 4 位十六进制整数为码点；
2. 编码成 UTF-8。

U+0000~U+FFFF 属于 BMP（basic multilingual plane，基本多文种平面）。BMP 以外的字符使用代理对（surrogate pair）表示`\uXXXX\uYYYY`。若第一个码点在 U+D800~U+DBFF，我们便知道了它的高代理项（high surrogate），之后跟着 U+DC00~U+DFFF 的低代理项（low surrogate），使用公式变代理对（H, L）为真实码点：
```
codepoint = 0x10000 + (H - 0xD800) x 0x400 + (L - 0xDC00)
```
若只有高代理项/低代理项码点不合法 -> `LEPT_PARSE_INVALID_UNICODE_SURROGATE`
若 `\u` 后不是 4 位十六进位数字 -> `LEPT_PARSE_INVALID_UNICODE_HEX`
### 3　UTF-8 编码
UTF-8 网页使用率第一。
UTF-8 的编码单元为 8 位（1 字节），每个码点编成 1~4 个字节。编码要把码点的二进制位分拆成 1~4 个字节：

| 码点范围            | 码点位数  | 字节1     | 字节2    | 字节3    | 字节4     |
|:------------------:|:--------:|:--------:|:--------:|:--------:|:--------:|
| U+0000 ~ U+007F    | 7        | 0xxxxxxx |
| U+0080 ~ U+07FF    | 11       | 110xxxxx | 10xxxxxx |
| U+0800 ~ U+FFFF    | 16       | 1110xxxx | 10xxxxxx | 10xxxxxx |
| U+10000 ~ U+10FFFF | 21       | 11110xxx | 10xxxxxx | 10xxxxxx | 10xxxxxx |
与 ASCII 相兼容。
例如解析 U+20AC：
1. U+20AC 在 U+0800 ~ U+FFFF 范围，编码为 3 字节。
2. U+20AC 为 10000010101100
3. 3 字节需 16 位码点，补两个 0，得 0010000010101100
4. 按表分为三组：0010 000010 101100
5. 加前缀：11100010 10000010 10101100
6. 转换为十六进制：0xE2 0x82 0xAC

```c
if (u >= 0x0800 && u <= 0xFFFF) {
  OutputByte(0xE0 | ((u >> 12) && 0xFF)); /* 0xE0 = 11100000 */
  OutputByte(0x80 | ((u >> 6 ) && 0x3F)); /* 0x80 = 10000000 */
  OutputByte(0x80 | ((u      ) && 0x3F)); /* 0x3F = 00111111 */
}
```
### 4　实现 `\uXXXX` 解析
```c
static int lept_parse_string(lept_context* c, lept_value* v) {
    unsigned u;
    /* ... */
    for (;;) {
        char ch = *p++;
        switch (ch) {
            /* ... */
            case '\\':
                switch (*p++) {
                    /* ... */
                    case 'u':
                        if (!(p = lept_parse_hex4(p, &u)))
                            STRING_ERROR(LEPT_PARSE_INVALID_UNICODE_HEX);
                        /* \TODO surrogate handling */
                        lept_encode_utf8(c, u);
                        break;
                    /* ... */
                }
            /* ... */
        }
    }
}
```