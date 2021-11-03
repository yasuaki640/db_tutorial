---
title: Part 2 - World's Simplest SQL Compiler and Virtual Machine
date: 2017-08-31
---

[comment]: <> (We're making a clone of sqlite. The "front-end" of sqlite is a SQL compiler that parses a string and outputs an internal representation called bytecode.)
我々はsqliteのクローンを作成しています。

sqliteの "front-end" は、文字列を解析し、バイトコードと呼ばれる内部表現を出力するSQLコンパイラです。

[comment]: <> (This bytecode is passed to the virtual machine, which executes it.)
このバイトコードは仮想マシンに渡され、実行されます。

{% include image.html url="assets/images/arch2.gif" description="SQLite Architecture (https://www.sqlite.org/arch.html)" %}

[comment]: <> (Breaking things into two steps like this has a couple advantages:)
[comment]: <> (- Reduces the complexity of each part &#40;e.g. virtual machine does not worry about syntax errors&#41;)
[comment]: <> (- Allows compiling common queries once and caching the bytecode for improved performance)
このように実行とコンパイルの2つのステップに分割することには、いくつか利点があります。
- 各要素の複雑さを軽減（たとえば、仮想マシンは構文エラーを気にする必要がない）
- 共通クエリを1回のみコンパイルし、バイトコードをキャッシュしパフォーマンス向上

[comment]: <> (With this in mind, let's refactor our `main` function and support two new keywords in the process:)
これを念頭に置いて、`main`関数をリファクタし、プロセスで2つの新しいキーワードをサポートしましょう。

```diff
 int main(int argc, char* argv[]) {
   InputBuffer* input_buffer = new_input_buffer();
   while (true) {
     print_prompt();
     read_input(input_buffer);

-    if (strcmp(input_buffer->buffer, ".exit") == 0) {
-      exit(EXIT_SUCCESS);
-    } else {
-      printf("Unrecognized command '%s'.\n", input_buffer->buffer);
+    if (input_buffer->buffer[0] == '.') {
+      switch (do_meta_command(input_buffer)) {
+        case (META_COMMAND_SUCCESS):
+          continue;
+        case (META_COMMAND_UNRECOGNIZED_COMMAND):
+          printf("Unrecognized command '%s'\n", input_buffer->buffer);
+          continue;
+      }
     }
+
+    Statement statement;
+    switch (prepare_statement(input_buffer, &statement)) {
+      case (PREPARE_SUCCESS):
+        break;
+      case (PREPARE_UNRECOGNIZED_STATEMENT):
+        printf("Unrecognized keyword at start of '%s'.\n",
+               input_buffer->buffer);
+        continue;
+    }
+
+    execute_statement(&statement);
+    printf("Executed.\n");
   }
 }
```

[comment]: <> (Non-SQL statements like `.exit` are called "meta-commands". They all start with a dot, so we check for them and handle them in a separate function.)
`.exit`のようなSQLでない文は"meat-commands"と呼ばれます。それらはすべて"."で始まるので、別の関数で処理します。

[comment]: <> (Next, we add a step that converts the line of input into our internal representation of a statement. This is our hacky version of the sqlite front-end.)
次に、入力行を命令文の内部表現に変換するステップを追加します。 これははsqliteフロントエンドの改造版です。

[comment]: <> (Lastly, we pass the prepared statement to `execute_statement`. This function will eventually become our virtual machine.)
最後に、プリペアドステートメントを `execute_statement`に渡します。この関数は、最終的には仮想マシンとなります。

[comment]: <> (Notice that two of our new functions return enums indicating success or failure:)
2つの新しい関数が成功または失敗を示す列挙型を返すことに注意してください。

```c
typedef enum {
  META_COMMAND_SUCCESS,
  META_COMMAND_UNRECOGNIZED_COMMAND
} MetaCommandResult;

typedef enum { PREPARE_SUCCESS, PREPARE_UNRECOGNIZED_STATEMENT } PrepareResult;
```

[comment]: <> ("Unrecognized statement"? That seems a bit like an exception. But [exceptions are bad]&#40;https://www.youtube.com/watch?v=EVhCUSgNbzo&#41; &#40;and C doesn't even support them&#41;, so I'm using enum result codes wherever practical. The C compiler will complain if my switch statement doesn't handle a member of the enum, so we can feel a little more confident we handle every result of a function. Expect more result codes to be added in the future.)
"認識されないステートメント"？ それは例外のように思えます。 ただし、[例外は悪い]（https://www.youtube.com/watch?v=EVhCUSgNbzo）(Cはそれらをサポートしていません)ので、実用的な場合は常に列挙型の結果コードを使用しています。 switch文が列挙型のメンバーをhandleしない場合、Cコンパイラは文句を言います。よって関数のすべての結果をhandleすることで自信を持てるようになります。 今後、さらに多くの結果コードが追加される予定です。

[comment]: <> (`do_meta_command` is just a wrapper for existing functionality that leaves room for more commands:)
`do_meta_command`は、既存の機能の単なるラッパーであり、より多くのコマンドを実装するための拡張性をもたらします。

```c
MetaCommandResult do_meta_command(InputBuffer* input_buffer) {
  if (strcmp(input_buffer->buffer, ".exit") == 0) {
    exit(EXIT_SUCCESS);
  } else {
    return META_COMMAND_UNRECOGNIZED_COMMAND;
  }
}
```

[comment]: <> (Our "prepared statement" right now just contains an enum with two possible values. It will contain more data as we allow parameters in statements:)
現在の「プリペアドステートメント」には、考えられる2つの値を持つ列挙型が含まれています。 ステートメントでパラメーターを許すと、より多くのデータを含められます。

```c
typedef enum { STATEMENT_INSERT, STATEMENT_SELECT } StatementType;

typedef struct {
  StatementType type;
} Statement;
```

[comment]: <> (`prepare_statement` &#40;our "SQL Compiler"&#41; does not understand SQL right now. In fact, it only understands two words:)
`prepare_statement`(私たちの"SQLコンパイラ")は今のところSQLを理解していません。実際2つの単語しか理解しません。
```c
PrepareResult prepare_statement(InputBuffer* input_buffer,
                                Statement* statement) {
  if (strncmp(input_buffer->buffer, "insert", 6) == 0) {
    statement->type = STATEMENT_INSERT;
    return PREPARE_SUCCESS;
  }
  if (strcmp(input_buffer->buffer, "select") == 0) {
    statement->type = STATEMENT_SELECT;
    return PREPARE_SUCCESS;
  }

  return PREPARE_UNRECOGNIZED_STATEMENT;
}
```

[comment]: <> (Note that we use `strncmp` for "insert" since the "insert" keyword will be followed by data. &#40;e.g. `insert 1 cstack foo@bar.com`&#41;)
「insert」キーワードの後にデータが続くため、insert処理には「strncmp」を使用することに注意してください。 (例： `insert 1 cstack foo @ bar.com`)

[comment]: <> (Lastly, `execute_statement` contains a few stubs:)
最後に、 `execute_statement`にはいくつかのスタブが含まれています。
```c
void execute_statement(Statement* statement) {
  switch (statement->type) {
    case (STATEMENT_INSERT):
      printf("This is where we would do an insert.\n");
      break;
    case (STATEMENT_SELECT):
      printf("This is where we would do a select.\n");
      break;
  }
}
```

[comment]: <> (Note that it doesn't return any error codes because there's nothing that could go wrong yet.)
まだ問題が発生する可能性はないため、エラーコードは返されません。

[comment]: <> (With these refactors, we now recognize two new keywords!)
これらのリファクタにより、2つの新しいキーワードが認識されるようになりました。
```command-line
~ ./db
db > insert foo bar
This is where we would do an insert.
Executed.
db > delete foo
Unrecognized keyword at start of 'delete foo'.
db > select
This is where we would do a select.
Executed.
db > .tables
Unrecognized command '.tables'
db > .exit
~
```

[comment]: <> (The skeleton of our database is taking shape... wouldn't it be nice if it stored data? In the next part, we'll implement `insert` and `select`, creating the world's worst data store. In the mean time, here's the entire diff from this part:)
私たちのデータベースの骨格は形になりつつあります...それがデータを保存していたら素晴らしいと思いませんか？次のパートでは、`insert`と` select`を実装して、世界最悪のデータストアを作成します。そうこうしているうちに、このpartと差分全体は次のとおりです。

```diff
@@ -10,6 +10,23 @@ struct InputBuffer_t {
 } InputBuffer;
 
+typedef enum {
+  META_COMMAND_SUCCESS,
+  META_COMMAND_UNRECOGNIZED_COMMAND
+} MetaCommandResult;
+
+typedef enum { PREPARE_SUCCESS, PREPARE_UNRECOGNIZED_STATEMENT } PrepareResult;
+
+typedef enum { STATEMENT_INSERT, STATEMENT_SELECT } StatementType;
+
+typedef struct {
+  StatementType type;
+} Statement;
+
 InputBuffer* new_input_buffer() {
   InputBuffer* input_buffer = malloc(sizeof(InputBuffer));
   input_buffer->buffer = NULL;
@@ -40,17 +57,67 @@ void close_input_buffer(InputBuffer* input_buffer) {
     free(input_buffer);
 }
 
+MetaCommandResult do_meta_command(InputBuffer* input_buffer) {
+  if (strcmp(input_buffer->buffer, ".exit") == 0) {
+    close_input_buffer(input_buffer);
+    exit(EXIT_SUCCESS);
+  } else {
+    return META_COMMAND_UNRECOGNIZED_COMMAND;
+  }
+}
+
+PrepareResult prepare_statement(InputBuffer* input_buffer,
+                                Statement* statement) {
+  if (strncmp(input_buffer->buffer, "insert", 6) == 0) {
+    statement->type = STATEMENT_INSERT;
+    return PREPARE_SUCCESS;
+  }
+  if (strcmp(input_buffer->buffer, "select") == 0) {
+    statement->type = STATEMENT_SELECT;
+    return PREPARE_SUCCESS;
+  }
+
+  return PREPARE_UNRECOGNIZED_STATEMENT;
+}
+
+void execute_statement(Statement* statement) {
+  switch (statement->type) {
+    case (STATEMENT_INSERT):
+      printf("This is where we would do an insert.\n");
+      break;
+    case (STATEMENT_SELECT):
+      printf("This is where we would do a select.\n");
+      break;
+  }
+}
+
 int main(int argc, char* argv[]) {
   InputBuffer* input_buffer = new_input_buffer();
   while (true) {
     print_prompt();
     read_input(input_buffer);
 
-    if (strcmp(input_buffer->buffer, ".exit") == 0) {
-      close_input_buffer(input_buffer);
-      exit(EXIT_SUCCESS);
-    } else {
-      printf("Unrecognized command '%s'.\n", input_buffer->buffer);
+    if (input_buffer->buffer[0] == '.') {
+      switch (do_meta_command(input_buffer)) {
+        case (META_COMMAND_SUCCESS):
+          continue;
+        case (META_COMMAND_UNRECOGNIZED_COMMAND):
+          printf("Unrecognized command '%s'\n", input_buffer->buffer);
+          continue;
+      }
     }
+
+    Statement statement;
+    switch (prepare_statement(input_buffer, &statement)) {
+      case (PREPARE_SUCCESS):
+        break;
+      case (PREPARE_UNRECOGNIZED_STATEMENT):
+        printf("Unrecognized keyword at start of '%s'.\n",
+               input_buffer->buffer);
+        continue;
+    }
+
+    execute_statement(&statement);
+    printf("Executed.\n");
   }
 }
```
