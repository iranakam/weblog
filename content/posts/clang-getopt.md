---
title: Cでコマンドラインオプションを取得する
date: 2019-06-24T00:00:00+09:00
tags: ["clang"]
draft: false

---

Cでコマンドラインオプションの利用方についてメモ。

# getopt()

Cでコマンドラインオプションを解析する場合は`getopt()`を利用します。

> #include \<unistd.h\>
> 
> int getopt(int argc, char * const argv[],
>            const char *optstring);
> 
> extern char \*optarg;
> extern int optind, opterr, optopt;

`getopt()`は`argv`と`argc`と`optstring`を引数に、`argv`の内'-'から始まるなどいくつかの条件で解析します。

`getopt()`は呼び出されるごとに、次のオプション文字列を返し、オプション文字列が見つからなくなる場合は-1を返します。また、`optstring`で指定されていないオプション文字列のの場合は'?'を戻り値とし、エラーメッセージを出力します。

`optstring`は受け付けるオプション文字列を連結した文字列、例えば`-a`と`-b`をオプションとする場合は'ab'とします。また':'を付け指定することで、引数を取るオプションである事を示します。

また、`getopt()`はグローバルの変数を3つ利用します。1つ目はエラーが発生した際にのオプション文字列が代入される`optopt`、2つ目は次の検査対象のオプション`argv`のインデックスが代入される`optind`、3つ目はエラーが発生した場合の挙動を制御するための`opterr`です。

```c
#include <stdio.h>
#include <unistd.h>

int
main(int argc, char* argv[])
{
  int opt;
  char* optargb = NULL;

  while ((opt = getopt(argc, argv, "ab:")) != -1) {
    switch (opt) {
      case 'a':
        printf("option: %c\n", opt);
        break;
      case 'b':
        printf("option: %c\n", opt);
        optargb = optarg;
        printf("arg: %s\n", optargb);
        break;
      case '?':
        printf("unknown option: %c\n", optopt);
        break;
    }
  }

  while (optind < argc) {
    printf("%s\n", argv[optind]);
    optind++;
  }
}
```

```bash
$ ./a.out  -a -b hoge -c fuga
option: a
option: b
arg: hoge
./a.out: invalid option -- 'c'
unknown option: c
fuga
```

# getopt_long()

長いオプションを解析する場合は`getopt_long()`を利用します。

> #include \<getopt.h\>
> 
> int getopt_long(int argc, char * const argv[],
>            const char *optstring,
>            const struct option *longopts, int *longindex);
> 
> int getopt_long_only(int argc, char * const argv[],
>            const char *optstring,
>   const struct option \*longopts, int \*longindex);

`argc`、`argv`、`optstring`については`getopt()`と同様、それに加えて`longopts`と`longindex`を引数とします。

`longopts`は構造体`option`の配列で、配列の最後の要素は各要素を0で埋めたものとします。`longindex`はポインタを指定すると、`longopts`で指定したオプションmojireどれに合致したかインデックスが代入されます。

## option

```c
struct option {
    const char *name;
    int         has_arg;
    int        *flag;
    int         val;
};
```

<dl>
  <dt>name</dt>
  <dd>オプションの名前を指定</dd>
  <dt>has_arg</dt>
  <dd>オプションが引数を取るかどうか、取らない場合はno_argument(もしくは0)、取る場合はrequired_argument(もしくは1)、オプショナルである場合はoptional_argument(もしくは2)、何れかを指定</dd>
  <dt>flag</dt>
  <dd>getopt_long()の戻り値の返し方を指定、NULLであればvalを、NULL以外の場合には0を返す</dd>
  <dt>val</dt>
  <dd>は戻り値、またはflagがポイントする変数へロードされる値</dd>
</dl>

```c
#include <stdio.h>
#include <unistd.h>
#include <getopt.h>

int
main(int argc, char* argv[])
{
  int opt;
  int longindex = -1;

  char* optargb = NULL;
  char* optargc = NULL;

  struct option longopts[] = {
    {"aaa", no_argument, NULL, 'a'},
    {"bbb", required_argument, NULL, 'b'},
    {"ccc", optional_argument, NULL, 'c'},
    {0, 0, 0, 0},
  };

  while ((opt = getopt_long(argc, argv, "ab:c:", longopts, &longindex)) != -1) {
    printf("%d %s\n", longindex, longopts[longindex].name);
    switch (opt) {
      case 'a':
        printf("option: %c\n", opt);
        break;
      case 'b':
        printf("option: %c\n", opt);
        optargb = optarg;
        printf("arg: %s\n", optargb);
        break;
      case 'c':
        printf("option: %c\n", opt);
        optargc = optarg;
        printf("arg: %s\n", optargc);
        break;
      case '?':
        printf("unknown option: %c\n", optopt);
        break;
    }
  }

  while (optind < argc) {
    printf("%s\n", argv[optind]);
    optind++;
  }
}
```

```bash
$ ./a.out --aaa --b hoge --ccc fuga piyo
0 aaa
option: a
1 bbb
option: b
arg: hoge
2 ccc
option: c
arg: (null)
fuga
piyo
```

なお、`--ccc fuga`では値が渡せません...値を渡す場合には'='での指定が必要です。

[c - getopt does not parse optional arguments to parameters - Stack Overflow:](https://stackoverflow.com/questions/1052746/getopt-does-not-parse-optional-arguments-to-parameters)

> Although not mentioned in glibc documentation or getopt man page, optional arguments to long style command line parameters require 'equals sign' (=). Space separating the optional argument from the parameter does not work.

```bash
$ ./a.out --aaa --b hoge --ccc=fuga piyo
0 aaa
option: a
1 bbb
option: b
arg: hoge
2 ccc
option: c
arg: fuga
piyo
```

んん、うまくまとまらない...また加筆します。
