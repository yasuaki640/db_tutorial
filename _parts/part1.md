---
title: Part 1 - Introduction and Setting up the REPL
date: 2017-08-30
---

Web開発者として、私は仕事でリレーショナルデータベースを毎日使用していますが、それらは私にとってブラックボックスです。
- データはどのような形式で保存される？ （メモリおよびディスク上で）
- いつメモリからディスクに移動する？
- テーブルごとに主キーが1つしかないのはなぜ？
- トランザクションのロールバックはどのように機能する？
- インデックスはどのようにフォーマットされる？
- フルテーブルスキャンはいつどのように行われる？
- プリペアドステートメントはどのような形式で保存される？

言い換えれば、データベースはどのように**機能**するのか？

事態を理解するために、私はデータベースをスクラッチで書きました。 MySQLやPostgreSQLよりも機能が少なく、小さく設計されているため、sqliteをモデルにしています。データベース全体が1つのファイルに保存されます！

# Sqlite

sqliteの内部についてはサイト上の[ドキュメント](https://www.sqlite.org/arch.htm)が豊富で、さらに[SQLite Database System: Design and Implementation](https://play.google.com/store/books/details?id=9Z6IQQnX1JEC)という書籍のコピーをゲットしました。

{% include image.html url="assets/images/arch1.gif" description="sqlite architecture (https://www.sqlite.org/zipvfs/doc/trunk/www/howitworks.wiki)" %}

クエリは、データを取得または変更するために一連のコンポーネントを通過します。 それらの**front-end**は

- tokenizer
- parser
- code generator

で構成されます。

front-endへの入力はSQLクエリです。 出力はsqlite virtual machineのバイトコード（基本的にはデータベースで動作できるコンパイル済みプログラム）です。

_back-end_ は
- virtual machine
- B-tree
- pager
- os interface

で構成されています。

**virtual machine**は、front-endによって生成されたバイトコードを命令として受け取り、1つ以上のテーブルまたはインデックスに対して操作を実行できます。各テーブルまたはインデックスは、B-treeと呼ばれるデータ構造に格納されています。 VMは本質的に、バイトコード命令種別に関する大きなswitch文なのです。

それぞれの** B-tree**は多くのノードで構成されています。 各ノードの長さは1ページです。 B-treeは、pagerにコマンドを発行することにより、ディスクからページを取得したり、ディスクに保存したりできます。

**pager**は、データのページを読み書きするコマンドを受け取り、データベースファイルの適切な[offset](https://ja.wikipedia.org/wiki/%E3%82%AA%E3%83%95%E3%82%BB%E3%83%83%E3%83%88_(%E3%82%B3%E3%83%B3%E3%83%94%E3%83%A5%E3%83%BC%E3%82%BF))での読み取り/書き込みを担当します。 また、最近アクセスしたページのキャッシュをメモリに保持し、それらのページをいつディスクに書き戻す必要があるかを判断します。

**os interface**は、sqliteがコンパイルされたオペレーティングシステムによって異なるレイヤーです。 このチュートリアルでは、マルチプラットフォームをサポートしません。

[千里の道も一歩から](https://en.wiktionary.org/wiki/a_journey_of_a_thousand_miles_begins_with_a_single_step)、、、それではもう少し簡単なREPLから始めましょう。

## Making a Simple REPL

<!-- Sqlite starts a read-execute-print loop when you start it from the command line: -->
Sqliteは、コマンドラインから起動すると、read-execute-printループに入ります。

```shell
~ sqlite3
SQLite version 3.16.0 2016-11-04 19:09:39
Enter ".help" for usage hints.
Connected to a transient in-memory database.
Use ".open FILENAME" to reopen on a persistent database.
sqlite> create table users (id int, username varchar(255), email varchar(255));
sqlite> .tables
users
sqlite> .exit
~
```

<!-- To do that, our main function will have an infinite loop that prints the prompt, gets a line of input, then processes that line of input: -->
このため、メイン関数にはプロンプトを出力し、入力行を取得して処理する無限ループがあります。

```c
int main(int argc, char* argv[]) {
  InputBuffer* input_buffer = new_input_buffer();
  while (true) {
    print_prompt();
    read_input(input_buffer);

    if (strcmp(input_buffer->buffer, ".exit") == 0) {
      close_input_buffer(input_buffer);
      exit(EXIT_SUCCESS);
    } else {
      printf("Unrecognized command '%s'.\n", input_buffer->buffer);
    }
  }
}
```

<!-- We'll define `InputBuffer` as a small wrapper around the state we need to store to interact with [getline()](http://man7.org/linux/man-pages/man3/getline.3.html). (More on that in a minute) -->
[`getline()`](http://man7.org/linux/man-pages/man3/getline.3.html)との対話のために持つ必要がある状態のラッパーとして `InputBuffer`を定義します。 (これについては後ほど詳しく説明します)

```c
typedef struct {
  char* buffer;
  size_t buffer_length;
  ssize_t input_length;
} InputBuffer;

InputBuffer* new_input_buffer() {
  InputBuffer* input_buffer = (InputBuffer*)malloc(sizeof(InputBuffer));
  input_buffer->buffer = NULL;
  input_buffer->buffer_length = 0;
  input_buffer->input_length = 0;

  return input_buffer;
}
```

[comment]: <> (Next, `print_prompt&#40;&#41;` prints a prompt to the user. We do this before reading each line of input.)
`print_prompt（）`はプロンプトを出力します。入力の各行を読み取る前に、この関数を実行します。

```c
void print_prompt() { printf("db > "); }
```


[comment]: <> (To read a line of input, use [getline&#40;&#41;]&#40;http://man7.org/linux/man-pages/man3/getline.3.html&#41;:)
入力行を読み取るには、 [getline()](http://man7.org/linux/man-pages/man3/getline.3.html)を使用します

```c
ssize_t getline(char **lineptr, size_t *n, FILE *stream);
```

[comment]: <> (`lineptr` : a pointer to the variable we use to point to the buffer containing the read line. If it set to `NULL` it is mallocatted by `getline` and should thus be freed by the user, even if the command fails.)

`lineptr` : 読み取り行を含むバッファーを指すために使用する変数へのポインタ。 NULLに設定すると、getlineによってメモリが動的に割り当てられるため、コマンドが失敗した場合でも、ユーザーはメモリ解放する必要があります。


[comment]: <> (`n` : a pointer to the variable we use to save the size of allocated buffer.)
`n` : 割り当てられたバッファのサイズを保存するために使用する変数へのポインタ。

`stream` : the input stream to read from. We'll be reading from standard input.

`return value` : the number of bytes read, which may be less than the size of the buffer.

We tell `getline` to store the read line in `input_buffer->buffer` and the size of the allocated buffer in `input_buffer->buffer_length`. We store the return value in `input_buffer->input_length`.

`buffer` starts as null, so `getline` allocates enough memory to hold the line of input and makes `buffer` point to it.

```c
void read_input(InputBuffer* input_buffer) {
  ssize_t bytes_read =
      getline(&(input_buffer->buffer), &(input_buffer->buffer_length), stdin);

  if (bytes_read <= 0) {
    printf("Error reading input\n");
    exit(EXIT_FAILURE);
  }

  // Ignore trailing newline
  input_buffer->input_length = bytes_read - 1;
  input_buffer->buffer[bytes_read - 1] = 0;
}
```

Now it is proper to define a function that frees the memory allocated for an
instance of `InputBuffer *` and the `buffer` element of the respective
structure (`getline` allocates memory for `input_buffer->buffer` in
`read_input`).

```c
void close_input_buffer(InputBuffer* input_buffer) {
    free(input_buffer->buffer);
    free(input_buffer);
}
```

Finally, we parse and execute the command. There is only one recognized command right now : `.exit`, which terminates the program. Otherwise we print an error message and continue the loop.

```c
if (strcmp(input_buffer->buffer, ".exit") == 0) {
  close_input_buffer(input_buffer);
  exit(EXIT_SUCCESS);
} else {
  printf("Unrecognized command '%s'.\n", input_buffer->buffer);
}
```

Let's try it out!
```shell
~ ./db
db > .tables
Unrecognized command '.tables'.
db > .exit
~
```

Alright, we've got a working REPL. In the next part, we'll start developing our command language. Meanwhile, here's the entire program from this part:

```c
#include <stdbool.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

typedef struct {
  char* buffer;
  size_t buffer_length;
  ssize_t input_length;
} InputBuffer;

InputBuffer* new_input_buffer() {
  InputBuffer* input_buffer = malloc(sizeof(InputBuffer));
  input_buffer->buffer = NULL;
  input_buffer->buffer_length = 0;
  input_buffer->input_length = 0;

  return input_buffer;
}

void print_prompt() { printf("db > "); }

void read_input(InputBuffer* input_buffer) {
  ssize_t bytes_read =
      getline(&(input_buffer->buffer), &(input_buffer->buffer_length), stdin);

  if (bytes_read <= 0) {
    printf("Error reading input\n");
    exit(EXIT_FAILURE);
  }

  // Ignore trailing newline
  input_buffer->input_length = bytes_read - 1;
  input_buffer->buffer[bytes_read - 1] = 0;
}

void close_input_buffer(InputBuffer* input_buffer) {
    free(input_buffer->buffer);
    free(input_buffer);
}

int main(int argc, char* argv[]) {
  InputBuffer* input_buffer = new_input_buffer();
  while (true) {
    print_prompt();
    read_input(input_buffer);

    if (strcmp(input_buffer->buffer, ".exit") == 0) {
      close_input_buffer(input_buffer);
      exit(EXIT_SUCCESS);
    } else {
      printf("Unrecognized command '%s'.\n", input_buffer->buffer);
    }
  }
}
```
