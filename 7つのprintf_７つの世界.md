---
marp: true
theme: gaia
---

# 7つの printf 7 つの世界

磯部 創一郎

---

## この発表の経緯

- 雑談をしていたときに C 言語の `printf` の仕様で盛り上がった
- 言語ごとに `printf` 的な文字列のフォーマット処理に対するアプローチが違っていて、まとめたら結構面白いのでは？
- そういえば 7 つの言語 7 つの世界っていう本あったから、それ文字ってしまう？
- という感じで、タイトル先行で資料を作成した

---

## 7 つの printf たち

- C 言語の `printf`
- Go の `fmt.Printf`
- Rust の `println!`
- OCaml の `printf`
- (鋭意作成中)Common Lisp の format
- (鋭意作成中)Idris で printf の型を実現するには？
- (鋭意作成中)TypeScript で printf の型を実現するには？

---

![bg 70% left](c_dall.png)

## C 言語

（このキャラクターは DALL-E 3 に生成させたもので実在しない）

---

### C 言語の printf

```c
printf("%d + %d = %d", 1, 1, 2); // => 1 + 1 = 2
```

- C 言語の printf の文字列には `%d` や `%s` といった format specifier を指定して、第二引数以降の値を元に文字列を出力する機能がある
- つまり引数の数は第一引数の文字列で format specifier をどれだけ使用したかで変わる
- どのように実現するのだろうか？

---

### C 言語の可変長引数

- C 言語では可変長引数（Variable-Length Parameter Lists）がサポートされており、`printf` はそれによって実現している

---

```c
#include <stdarg.h>

void my_printf(const char *format, ...)
{
    va_list argptr;
    
    va_start(argptr, format);
    va_arg(argptr, int); // => 1 として使用できる
    va_arg(argptr, int); // => 1 として使用できる
    va_arg(argptr, int); // => 2 として使用できる
    va_end(argptr);
}

int main(void)
{
    my_printf("%d + %d = %d", 1, 1, 2); 
    return 0;
}
```

参考: https://ja.wikibooks.org/wiki/C%E8%A8%80%E8%AA%9E/%E6%A8%99%E6%BA%96%E3%83%A9%E3%82%A4%E3%83%96%E3%83%A9%E3%83%AA/stdarg.h

---

### 可変長引数の問題

- これでめでたしめでたしとはいかない
- というのも C 言語の可変長引数の数が適切かは _コンパイル時でも実行時においても決定できない_
- ゆえに引数なしで `%d` を使うこともできてしまう

---

- 例えば以下のようにすると

```c
#include <stdio.h>

int main(void)
{
    printf("%d");
    return 0;
}
```

意味不明な値が表示される :innocent: （クラッシュすらしない）

```shell
$ ./a.out
335738696
```

---

### 何が起きたのか？

- 大前提、関数呼び出しをする場合、引数の情報をスタックメモリにつめて関数を呼び出している
- C 言語の可変長引数でやっていることは関数の引数がスタックされていると思われるメモリを読みにいくこと
  - 参考: https://qiita.com/subaruf/items/657c67a1809515589a7c
- なのでそこにある値が適切なのかすらわからず、正しく実装しないと出鱈目な値として解釈しようとしてしまう（デバッグしんどいね）
- もちろん型情報もないので、型が間違っていてもコンパイルでは気がつけない

---

### C言語の printf にまつわる余談

- C言語のコンパイラの努力
  - 実装依存ではあるが、`printf` の文字列リテラルをコンパイラが解析して Warning を出したりはしてくれるよう
  - Hello, world を `printf` で実装するのはあるあるだが、gcc とかでは最適化の過程で単なる標準出力である `puts` に変換されたりする

---

### C言語の printf にまつわる余談 2

- ブラウザ開発で SDK のバージョンをあげた後、特定の URL でクラッシュするようになった
  - 調査していくと URL のデバッグ出力に使っていた SDK の関数の仕様が `printf` と同等になったことで、URL エンコーディングの % を format specifier と誤認していたことが発覚
  - 出鱈目なメモリを読みに行きクラッシュという現象であった :innocent:
  - エスケープも考えたが、以下の対応を思いついていい感じに直せた
    ```c
    // sdkPrintf(url);
    sdkPrintf("%s", url);
    ```

---

![bg 70% left](gopher.png)

## Go 言語

(Gopher くんはネズミ)

---

### Go 言語の fmt.Printf

```go
fmt.Printf("%d + %d = %d\n", 1, 1, 2)
// => 1 + 1 = 2

fmt.Printf("%d + %d = %d\n", 1, 1, "2")
// => panic: fmt.Printf format %d has arg "2" of wrong type string
```

- C 言語の printf 同様 `%d` や `%s` といった format specifier を指定して、第二引数以降の値を元に文字列を出力する機能をもつ
    - `%v` といった独自の拡張ももつ（構造体で主に使うがそれ以外の primitive な型にも対応）
- C 言語と違い無理やりポインタを読むのではなく、ちゃんと型として意図しているかはチェックされる

---

### Go の可変長引数

```go
func myPrintf(format string, nums ...int) {
	fmt.Print(nums, " ")
}

func main() {
	myPrintf("%d + %d = %d", 1, 1, 2) // => [1 1 2]
}
```

- 同じ型をスライスとして受け取れる仕組み
- C 言語のようなポインタ操作と違い `fmt.Printf` の引数の数が正しいかわかる
- 同じ型しか受け取れないが、 `fmt.Printf` の format specifier によって色々な型を使い分けたい

---

### Go のリフレクション

- Go には `interface` という、TypeScript でいう any のような型が存在する
- ただコンパイル時に型情報を失っておらず、どの型だったのか `.(type)` を呼び出すことで元に戻すことができる

```go
switch f := arg.(type) {
case bool:
    ...
```

参考: https://cs.opensource.google/go/go/+/refs/tags/go1.23.4:src/fmt/print.go;l=707

---

### Go のリフレクションで実現される fmt.Printf

- リフレクションにより、任意の型、任意の数の引数を受け取れるため `printf` を実現できる
- さらに `%v` といった format specifier 側で型を意識しなくても出力できるようなことも実現（引数の型をリフレクションで判定すればいいので）

---

## Go の fmt.Printf は万能か？

- 残念ながら、すべての問題を解決はできない
- 例えば以下の点
    - コンパイル時にエラーとならないこと
    - リフレクションの機構は、実行時に型情報をもつためオーバーヘッドとなること
        - C 言語のようなシステムプログラミングとなると無視できないレベルにはなりそう

---

![bg 70% left](ferris.png)

## Rust

(このキャラクターの名前は Ferris 、非公式)

---

### Rust の println!

```rust
println!("{} + {} = {}", 1, 1, 2); // => 1 + 1 = 2
```

- `println!` が C言語の `printf` と同等の機能を備えている
- `printf` 同様、可変長引数を利用できるのだが、 Rust 自体は可変長引数をサポートしていない
- どう実現しているのか？

---

### Rust のマクロ

- Rust にはマクロという仕組みで、コンパイル時の抽象構文木（以下、AST） に作用して AST を作り出せる仕組みを持っている
    - 参考: https://doc.rust-jp.rs/rust-by-example-ja/macros.html
- `println!` の `!` はマクロであることを示しており、 `println!` は型をチェックする前にコードが展開される

---

### Rust のマクロによる可変長引数

```rust
macro_rules! my_printf {
    ( $( $x:expr ),* ) => {
        {
            $(println!("{}", $x);)* // ここが引数の数だけ展開される
        }
    };
}

fn main() {
    my_printf!("%d + %d = %d", 1, 1, 2);
}
```
参考: https://note.com/webdrawer/n/n8f6d7a871424

---

### println! のマクロ展開後

```rust
let args = format_args!("Hello, {0} and {1}!\n", name1, name2);

// => 以下のように展開される
let args = ::std::fmt::Arguments::new_v1(
    &["Hello, ", " and ", "!\n"],
    // 略
);
```

- `println!` の内部では `format_args!` という別のマクロが使われている
- このマクロは例えば `"Hello, {0} and {1}!"` のような文字列を指定されると、 `["Hello,", " and ", "!"]` のリストに変換される
    - 参考: https://ubnt-intrepid.netlify.app/rust-format-args/

---

### 他の言語のマクロとの比較

- C 言語にもマクロが存在するが、こちらは単なる文字列処理で AST として処理されない
- AST レベルでマクロの処理があるのは有名どころだと Lisp
    - Lisp は S 式ベースゆえに AST レベルのマクロのイメージしやすい気はする
- Elixir という Erlang VM 上で動作する言語にも同等のマクロが存在する
    - 前職の先輩がマクロを使って、Elixir の文法を便利な形に改良していてマクロとしての発想の参考になる(https://github.com/skirino/croma)

---

### Rust の println! は万能か？

- コンパイル時に型のチェックもできる点でかなり強力だが、C, Go にできて Rust でできないことがある

```rust
fn main() {
    let s = "Hello, world!";
    println!(s); // error: format argument must be a string literal
}
```

- `println!` はマクロで展開する都合、文字列リテラルでないと使用できない(format_args! 使えばある程度はいけるかも？)
- あとマクロはコンパイルがとても遅くなる（それはそう）
- `println!` は実装を読まないでよいが、マクロを読むのは難しい

---

![bg left 70%](ocaml.png)

## OCaml

（Ocaml ではない）

---

### OCaml の printf

```ocaml
let () = printf "%d + %d = %d\n" 1 1 2;;
```

- OCaml には printf という C 言語とかなり近い IF の組み込み関数が存在している
- OCaml は型の制約が強い言語なので Rust 同様マクロが欲しくなるところ
- だが ppx というプラグインのマクロはあるものの、標準ではサポートされていない
- もちろん可変長引数もサポートされていない
- どう実現しているのか？

---

### OCaml の format 型

- 結論 OCaml において `printf` のために format 型という特殊な型が用意されている
- `%d` といった文字列を読み取って int が必要とコンパイル時に判断してくれる
- `("%d\n" : (_,_,_) format)` とかで format に変換すると `(int -> 'a, 'b, 'a) format = <abstr>` のように %d から int を期待する関数として解釈される
    - 参考: https://camlspotter.hatenablog.com/entry/20091102/1257099984

---

### OCaml の REPL

```ocaml
"hello" ^ " world";;
- : string = "hello world"
```

- ocaml でコードを実行すると、最終的な型と値を返してくれる

---

### OCaml の REPL で string を format 型にキャストする

```ocaml
("%d + %d = %d\n" : (_,_,_) format);;
- : (int -> int -> int -> 'a, 'b, 'a) format =
CamlinternalFormatBasics.Format
 (CamlinternalFormatBasics.Int (CamlinternalFormatBasics.Int_d,
   CamlinternalFormatBasics.No_padding,
   CamlinternalFormatBasics.No_precision,
   CamlinternalFormatBasics.String_literal (" + ",
    CamlinternalFormatBasics.Int (CamlinternalFormatBasics.Int_d,
     CamlinternalFormatBasics.No_padding,
     CamlinternalFormatBasics.No_precision,
     CamlinternalFormatBasics.String_literal (" = ",
      CamlinternalFormatBasics.Int (CamlinternalFormatBasics.Int_d,
       CamlinternalFormatBasics.No_padding,
       CamlinternalFormatBasics.No_precision,
       CamlinternalFormatBasics.Char_literal ('\n',
        CamlinternalFormatBasics.End_of_format)))))),
 "%d + %d = %d\n")
```

---

### OCaml のカリー化

```ocaml
let a = printf "%d + %d = %d" 1 1;;
val a : int -> unit = <fun>
```

```ocaml
let () = printf "%d + %d = %d" 1 1;;
Error: This expression has type int -> unit but an expression was expected of type
  unit
```

- ちなみにデフォルトでカリー化されているため、引数が足りないということは起きえない
- ただ let とするときに () として何も値を持たない書き方をすることがほとんどなので、そこでコンパイルエラーになることがほとんどと思われる

---

### OCaml の printf は万能か？

```rust
open Printf
let s = ("hello %s" : (_,_,_) format);;
let () = printf s;;
```

- Rust と違い、format 型にすれば文字列リテラルでなくても一応いける
- が、 format 型自体特殊なので適用される条件は複雑っぽい

---

### OCaml の型推論

- OCaml は型推論が強力なのでほとんど型を書かずに実装できる
- ゆえにカリー化で正しくない型が来たとしても、型推論に任せて予期せぬコンパイルエラーになったりでデバッグが難しい側面がある（カリー化便利なんだけどね）

---

## CommonLisp

---

### CommonLisp の format

  - Common Lisp には format 関数が printf としての役割を持っている
  - 動的型付け言語なので、引数が足りない場合は No more argumens のエラーで実行時エラー
  - 型自体は面白いわけではないが、機能が多すぎる

---

### なんと loop がいける

```lisp
(format t "~{~10,'0d~%~}" '(1 2 3 4 5))
```

```
0000000001
0000000002
0000000003
0000000004
0000000005
```

---

### クワインもいけてしまう

```lisp
(format t "(format t ~s~:* ~s)" "(format t ~s~:* ~s)")
```

```
(format t "(format t ~s~:* ~s)" "(format t ~s~:* ~s)")
```

- 参考: https://qiita.com/t-sin/items/d88887b595728b3a6b97

---

### Idris

- 参考: https://qiita.com/SekiT/items/d4692f6e47ace97b8f1e

---

### awk
- 参考: https://qiita.com/ko1nksm/items/16ca4f4546c1ea8e97d7#awk-%E3%81%AE%E8%A8%80%E8%AA%9E%E4%BB%95%E6%A7%98

---

### TypeScript

- 参考: https://www.hacklewayne.com/a-truly-strongly-typed-printf-in-typescript