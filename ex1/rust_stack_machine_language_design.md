よっしゃー。
Codexに渡すための **引き継ぎ仕様書** として、そのまま `README.md` や `docs/design.md` に貼れる形でまとめます。

---

# Rustスタックマシン型スクリプト言語 開発引き継ぎメモ

## 目的

Rustで、行列演算用途に拡張できる小さなスタックマシン型スクリプト言語を開発する。

最終的には、以下を目指す。

```text
スタックマシン型スクリプト言語
↓
行列演算DSL
↓
計算グラフ記述
↓
仮想メモリモデル
↓
GPU風の仮想演算機
↓
CPU上で高速実行
```

ただし、最初から全部は作らない。
まずは **Rust初心者でも理解できる最小VM** を作る。

---

# 開発方針

## 最初のMVP

最初は以下を動かす。

```text
1 2 3 * + print
```

これは次の計算を意味する。

```text
1 + 2 * 3
```

期待結果：

```text
F64(7.0)
```

---

# 言語仕様 Phase 1

## スクリプト形式

Forth風の空白区切り言語にする。

```text
1 2 + print
```

## 意味

```text
1      -> スタックに 1.0 を積む
2      -> スタックに 2.0 を積む
+      -> スタックから2つ取り出して加算する
print  -> スタックトップを表示する
```

---

# 命令セット Phase 1

最初に実装する命令はこれだけ。

```rust
enum OpCode {
    PushF64(f64),
    Add,
    Sub,
    Mul,
    Div,
    Print,
    Halt,
}
```

---

# 値の型 Phase 1

最初は `f64` だけ扱う。

```rust
#[derive(Debug, Clone)]
enum Value {
    F64(f64),
}
```

将来的には以下に拡張する。

```rust
#[derive(Debug, Clone)]
enum Value {
    F64(f64),
    Array(Vec<f64>),
    Matrix(Matrix),
}
```

---

# VM構造

```rust
struct VM {
    stack: Vec<Value>,
    code: Vec<OpCode>,
    ip: usize,
}
```

## 各フィールドの意味

```text
stack : 実行時スタック
code  : バイトコード列
ip    : instruction pointer。現在実行中の命令位置
```

---

# 実行モデル

VMは `code[ip]` を1命令ずつ実行する。

```text
命令を取得
↓
ipを1進める
↓
命令に応じてstackを操作
↓
Haltで終了
```

---

# Phase 1で必要な関数

## VM::new

```rust
fn new(code: Vec<OpCode>) -> Self
```

VMを初期化する。

## VM::run

```rust
fn run(&mut self) -> Result<(), String>
```

命令列を実行する。

## VM::pop_f64

```rust
fn pop_f64(&mut self) -> Result<f64, String>
```

スタックから `Value::F64` を取り出す。
型が違う場合やスタックが空の場合はエラーにする。

---

# コンパイラ Phase 1

最初のコンパイラは、空白区切りの文字列を `OpCode` に変換するだけでよい。

```rust
fn compile(source: &str) -> Result<Vec<OpCode>, String>
```

## 対応トークン

```text
+      -> OpCode::Add
-      -> OpCode::Sub
*      -> OpCode::Mul
/      -> OpCode::Div
print  -> OpCode::Print
数値   -> OpCode::PushF64(x)
```

最後に必ず `OpCode::Halt` を追加する。

---

# Phase 1の完成イメージ

## 入力

```text
1 2 3 * + print
```

## コンパイル後

```rust
vec![
    OpCode::PushF64(1.0),
    OpCode::PushF64(2.0),
    OpCode::PushF64(3.0),
    OpCode::Mul,
    OpCode::Add,
    OpCode::Print,
    OpCode::Halt,
]
```

## 実行結果

```text
F64(7.0)
```

---

# 推奨ディレクトリ構成

最初は単純にしてよい。

```text
stackmat/
├── Cargo.toml
└── src/
    └── main.rs
```

Phase 1では `main.rs` だけでよい。

---

# Cargoプロジェクト作成

```bash
cargo new stackmat
cd stackmat
cargo run
```

---

# Phase 1 実装コード案

Codexには、まず以下のようなコードを生成・整理してもらう。

```rust
#[derive(Debug, Clone)]
enum Value {
    F64(f64),
}

#[derive(Debug, Clone)]
enum OpCode {
    PushF64(f64),
    Add,
    Sub,
    Mul,
    Div,
    Print,
    Halt,
}

struct VM {
    stack: Vec<Value>,
    code: Vec<OpCode>,
    ip: usize,
}

impl VM {
    fn new(code: Vec<OpCode>) -> Self {
        Self {
            stack: Vec::new(),
            code,
            ip: 0,
        }
    }

    fn run(&mut self) -> Result<(), String> {
        loop {
            let op = self
                .code
                .get(self.ip)
                .ok_or("instruction pointer out of range")?
                .clone();

            self.ip += 1;

            match op {
                OpCode::PushF64(x) => {
                    self.stack.push(Value::F64(x));
                }

                OpCode::Add => {
                    let b = self.pop_f64()?;
                    let a = self.pop_f64()?;
                    self.stack.push(Value::F64(a + b));
                }

                OpCode::Sub => {
                    let b = self.pop_f64()?;
                    let a = self.pop_f64()?;
                    self.stack.push(Value::F64(a - b));
                }

                OpCode::Mul => {
                    let b = self.pop_f64()?;
                    let a = self.pop_f64()?;
                    self.stack.push(Value::F64(a * b));
                }

                OpCode::Div => {
                    let b = self.pop_f64()?;
                    let a = self.pop_f64()?;

                    if b == 0.0 {
                        return Err("division by zero".to_string());
                    }

                    self.stack.push(Value::F64(a / b));
                }

                OpCode::Print => {
                    let v = self.stack.last().ok_or("stack is empty")?;
                    println!("{:?}", v);
                }

                OpCode::Halt => {
                    break;
                }
            }
        }

        Ok(())
    }

    fn pop_f64(&mut self) -> Result<f64, String> {
        match self.stack.pop() {
            Some(Value::F64(x)) => Ok(x),
            None => Err("stack underflow".to_string()),
        }
    }
}

fn compile(source: &str) -> Result<Vec<OpCode>, String> {
    let mut code = Vec::new();

    for token in source.split_whitespace() {
        match token {
            "+" => code.push(OpCode::Add),
            "-" => code.push(OpCode::Sub),
            "*" => code.push(OpCode::Mul),
            "/" => code.push(OpCode::Div),
            "print" => code.push(OpCode::Print),
            _ => {
                let x: f64 = token
                    .parse()
                    .map_err(|_| format!("unknown token: {}", token))?;

                code.push(OpCode::PushF64(x));
            }
        }
    }

    code.push(OpCode::Halt);

    Ok(code)
}

fn main() -> Result<(), String> {
    let source = "1 2 3 * + print";

    let code = compile(source)?;
    let mut vm = VM::new(code);

    vm.run()?;

    Ok(())
}
```

---

# Codexへの依頼文

Codexには、まずこのように依頼するとよい。

```text
Rust初心者向けに、スタックマシン型スクリプト言語の最小実装を作りたいです。

要件:
- RustのCargoプロジェクトとして作成する
- 最初はsrc/main.rsだけでよい
- ValueはF64(f64)のみ
- OpCodeは PushF64, Add, Sub, Mul, Div, Print, Halt
- VMは stack, code, ip を持つ
- source文字列を空白区切りでcompileしてVec<OpCode>に変換する
- "1 2 3 * + print" が F64(7.0) を出力する
- エラーはResult<(), String>で扱う
- Rust初心者でも読めるようにコメントを多めに入れる
- まずは動くことを優先し、最適化はしない
```

---

# Phase 2: 配列リテラル

Phase 1が動いたら、次に配列を追加する。

## 目標スクリプト

```text
[1 2 3 4] print
```

## Value追加

```rust
enum Value {
    F64(f64),
    Array(Vec<f64>),
}
```

## OpCode追加

```rust
enum OpCode {
    PushF64(f64),
    PushArray(Vec<f64>),
    Add,
    Sub,
    Mul,
    Div,
    Print,
    Halt,
}
```

## 注意

`split_whitespace()` だけだと、

```text
[1 2 3 4]
```

の扱いが難しくなる。

そのため Phase 2 では簡易lexerを作る。

---

# Phase 3: 行列型

## Matrix構造体

```rust
#[derive(Debug, Clone)]
struct Matrix {
    rows: usize,
    cols: usize,
    data: Vec<f64>,
}
```

## Value追加

```rust
enum Value {
    F64(f64),
    Array(Vec<f64>),
    Matrix(Matrix),
}
```

## 目標スクリプト

```text
[1 2 3 4] 2 2 mat print
```

## 意味

```text
[1 2 3 4] 2 2 mat
```

は、2x2行列を作る。

```text
Matrix {
    rows: 2,
    cols: 2,
    data: [1.0, 2.0, 3.0, 4.0]
}
```

---

# Phase 4: 行列加算

## 目標スクリプト

```text
[1 2 3 4] 2 2 mat
[10 20 30 40] 2 2 mat
matadd
print
```

## 期待結果

```text
Matrix {
    rows: 2,
    cols: 2,
    data: [11.0, 22.0, 33.0, 44.0]
}
```

---

# Phase 5: 行列積

## 目標スクリプト

```text
[1 2 3 4] 2 2 mat
[5 6 7 8] 2 2 mat
matmul
print
```

## 期待結果

```text
Matrix {
    rows: 2,
    cols: 2,
    data: [19.0, 22.0, 43.0, 50.0]
}
```

---

# Phase 6: 変数

## 目標スクリプト

```text
[1 2 3 4] 2 2 mat store A
[5 6 7 8] 2 2 mat store B

load A
load B
matmul
print
```

## VMに追加するもの

```rust
use std::collections::HashMap;

struct VM {
    stack: Vec<Value>,
    code: Vec<OpCode>,
    ip: usize,
    vars: HashMap<String, Value>,
}
```

## OpCode追加

```rust
Store(String),
Load(String),
```

---

# Phase 7: 計算グラフ化

ここはまだ実装しない。
将来構想として残す。

現在のVMは即時実行型。

```text
命令を読む
↓
すぐ計算する
```

将来的には、

```text
命令を読む
↓
計算ノードを作る
↓
グラフとして保持する
↓
まとめて最適化して実行する
```

に拡張したい。

---

# 今回の重要方針

## 1. 最初は小さく作る

いきなり構文解析器や高度な型システムを作らない。

## 2. Forth風にする

スタックマシンとの相性がよい。

```text
1 2 + print
```

## 3. Rust初心者向けにする

最初は `main.rs` に全部書いてよい。
モジュール分割はあとでよい。

## 4. cloneは許容する

最初から所有権・借用・ゼロコピー最適化で詰まらない。
まず動くVMを作る。

## 5. 行列はrow-major

行列データは `Vec<f64>` で持つ。

```text
2x3 matrix

[1 2 3
 4 5 6]

data = [1, 2, 3, 4, 5, 6]
```

アクセス：

```rust
data[row * cols + col]
```

---

# 開発ロードマップ

```text
Phase 1: 数値スタックVM
Phase 2: 配列リテラル
Phase 3: 行列生成
Phase 4: 行列加算
Phase 5: 行列積
Phase 6: 変数 store/load
Phase 7: 計算グラフ
Phase 8: 仮想メモリモデル
Phase 9: 仮想GPU風演算器
```

---

# 今やるべきこと

まずはここだけやる。

```text
Phase 1: 数値スタックVM
```

完成条件：

```text
cargo run
```

で、

```text
F64(7.0)
```

が表示されること。

---

# Codexに最初にやらせる作業

```text
1. Cargoプロジェクト stackmat を作成
2. src/main.rs に最小VMを実装
3. "1 2 3 * + print" を実行
4. F64(7.0) が表示されることを確認
5. Rust初心者向けにコメントを追加
```

---

この引き継ぎで十分です。
最初は **Phase 1だけを確実に動かす** のが正解です。
行列、変数、計算グラフはそのあとでええです。
