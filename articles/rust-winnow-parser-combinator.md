---
title: "パーサコンビネータWinnow入門"
emoji: "🦀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["rust", "parser", "winnow"]
published: false
---

RustのパーサコンビネータWinnowの基本的な使い方について紹介する。

## 設計

Rustの他のパーサコンビネータと同じく、文字列やバイナリなど任意のトークン列をパース可能な設計になっている。
エラーはカスタマイズ可能で、速度を重視して最低限の情報だけ返すこともできるし、詳細な情報を返すこともできるようになっている。

その柔軟性の代償として、多くのtraitとstructが組み合わさった一見複雑な設計になっているのは仕方がない(nomやcombineも似たようなものである)

## 主要な型

### `trait Parser<I, O, E>`

パーサを表現する中心的なtrait。概念的には `FnMut(&mut I) -> Result<O, E>` に相当し、入力`&mut I`から値`O`をパースするか、エラー`E`を返す。

```rust
pub trait Parser<I, O, E> {
    fn parse_next(&mut self, input: &mut I) -> PResult<O, E>;
    
    // 多数のコンビネータメソッドが提供される
    fn by_ref(&mut self) -> ByRef<'_, Self> { ... }
    fn map<G, O2>(self, g: G) -> Map<Self, G> { ... }
    fn and_then<G, O2>(self, g: G) -> AndThen<Self, G> { ... }
    // ... など
}
```

このtraitの実装により、パーサ同士を組み合わせて複雑なパーサを構築できる：

```rust
// タプルでの連結
(parser1, parser2)  // parser1の後にparser2を実行

// 配列での繰り返し
[parser; 3]  // parserを3回実行

// Optionでの省略可能
Some(parser)  // 0回または1回実行
```

### `trait Stream`

```rust
pub trait Stream: Offset + Clone + Debug {
    type Token: Debug;
    type Slice: Debug;
    type IterOffsets: Iterator<Item = (usize, Self::Token)>;
    type Checkpoint: Offset + Clone + Debug;
    
    // 基本操作
    fn iter_offsets(&self) -> Self::IterOffsets;
    fn eof_offset(&self) -> usize;
    fn next_token(&mut self) -> Option<Self::Token>;
    fn peek_token(&self) -> Option<(Self, Self::Token)>;
    
    // チェックポイント機能
    fn checkpoint(&self) -> Self::Checkpoint;
    fn reset(&mut self, checkpoint: &Self::Checkpoint);
    
    // スライス操作
    fn peek_slice(&self, offset: usize) -> (Self, Self::Slice);
    fn take(&mut self, offset: usize) -> Self::Slice;
}
```

`Stream`は入力データの抽象化を提供する。標準的な実装：
- `&str` - 文字列のパース
- `&[u8]` - バイナリデータのパース
- `&[T]` - 任意の型のスライス
- `LocatingSlice<I>` - 位置情報付きストリーム
- `Partial<I>` - 不完全な入力の処理
- `Stateful<I, S>` - 状態付きストリーム

### `trait ParserError<I>`

```rust
pub trait ParserError<I>: Debug + Display {
    fn from_error_kind(input: &I, kind: ErrorKind) -> Self;
    fn append(self, input: &I, kind: ErrorKind) -> Self;
    
    // オプショナルなメソッド
    fn from_char(input: &I, char: char) -> Self { ... }
    fn from_external_error(input: &I, kind: ErrorKind, e: Box<dyn Error>) -> Self { ... }
}
```

エラー処理の柔軟性を提供する。主な実装：
- `()` - 最小限のエラー情報（高速）
- `ErrorKind` - 基本的なエラー種別
- `ContextError` - コンテキスト情報付きエラー
- `VerboseError` - デバッグ用の詳細エラー

### `enum ErrMode<E>`

```rust
pub enum ErrMode<E> {
    Incomplete(Needed),     // 入力が不足
    Backtrack(E),          // バックトラック可能なエラー
    Cut(E),                // バックトラック不可能なエラー
}
```

パーサの失敗モードを表現する。`Cut`を使うことで、無駄なバックトラックを防ぎ、より良いエラーメッセージを生成できる。

### `struct PResult<O, E = ErrorKind>`

```rust
pub type PResult<O, E = ErrorKind> = Result<O, ErrMode<E>>;
```

パーサの結果型。通常の`Result`に`ErrMode`でラップされたエラーを含む。

### ストリームラッパー型

#### `LocatingSlice<I>`
位置情報（行番号、列番号、バイトオフセット）を追跡する：

```rust
let input = LocatingSlice::new("hello\nworld");
// パース中に location() メソッドで現在位置を取得可能
```

#### `Partial<I>`
ストリーミングパースをサポート：

```rust
let mut input = Partial::new("incomplete dat");
// パーサは Incomplete エラーを返すことができる
```

#### `Stateful<I, S>`
パース中の状態管理：

```rust
let mut input = Stateful {
    input: "data",
    state: MyState::new(),
};
// パーサは input.state にアクセス可能
```

### 便利な型エイリアス

```rust
// よく使う組み合わせ
pub type IResult<I, O, E = ErrorKind> = Result<(I, O), ErrMode<E>>;
pub type ModalResult<O, E = ErrorKind> = Result<O, ErrMode<E>>;
```

## 基本的な使い方

### 一番シンプルなやつ

カンマ区切りの数字2つをパースしたい。

```rust
use winnow::prelude::*;
use winnow::ascii::dec_int;
use winnow::combinator::separated_pair;

fn parse(input: &mut &str) -> PResult<(i32, i32)> {
    separated_pair(dec_int, ',', dec_int).parse_next(input)
}

fn main() {
    let mut input = "123,456";
    let result = parse(&mut input);
    assert_eq!(result, Ok((123, 456)));
}
```

### バックトラックしたい

`ModalResult<T, E>`(実体は`Result<T, ErrMode<E>>`)を使用することになる。

```rust
pub enum ErrMode<E> {
    Incomplete(Needed),
    Backtrack(E),
    Cut(E),
}
```

パーサが`ErrMode::BackTrack`を返した場合、`alt()`等で別の選択肢が指定されている場合はそちらを試す。
`ErrMode::Cut(E)`を返した場合は、別の選択肢を試さずただちに失敗する。

```rust
use winnow::prelude::*;
use winnow::ascii::dec_int;
use winnow::combinator::{alt, delimited, separated_pair};

fn parse(input: &mut &str) -> PResult<(i32, i32)> {
    alt((
        // "(num1,num2)" パターン
        delimited('(', separated_pair(dec_int, ',', dec_int), ')'),
        // "num1,num2" パターン
        separated_pair(dec_int, ',', dec_int),
    )).parse_next(input)
}

fn main() {
    let mut input1 = "(123,456)";
    let mut input2 = "789,012";
    assert_eq!(parse(&mut input1), Ok((123, 456)));
    assert_eq!(parse(&mut input2), Ok((789, 12)));
}
```

### 位置を扱いたい

入力が`impl Location`である必要がある。典型的には、入力を`struct LocatingSlice`でラップする。

```rust
use winnow::prelude::*;
use winnow::ascii::dec_int;
use winnow::combinator::separated_pair;
use winnow::stream::{LocatingSlice, Location};

fn parse_with_location(input: &mut LocatingSlice<&str>) -> PResult<((i32, usize), (i32, usize))> {
    separated_pair(
        (dec_int, |input: &mut LocatingSlice<&str>| Ok(input.location())),
        ',',
        (dec_int, |input: &mut LocatingSlice<&str>| Ok(input.location()))
    ).parse_next(input)
}

fn main() {
    let input = "123,456";
    let mut input = LocatingSlice::new(input);
    let result = parse_with_location(&mut input);
    // 結果: ((123, 3), (456, 7)) - 各数値とその終了位置
    dbg!(result);
}
```

### 状態を扱いたい

`struct Stateful`で入力をラップする必要がある。

```rust
pub struct Stateful<I, S> {
    pub input: I,
    pub state: S,
}
```

ただし、バックトラックによって位置がリセットされても状態はリセットされないことに注意。そういうのが欲しければ自分で書く必要がある。

```rust
use winnow::prelude::*;
use winnow::ascii::dec_int;
use winnow::combinator::separated;
use winnow::stream::Stateful;

#[derive(Debug, Default)]
struct Counter {
    count: usize,
}

fn parse_with_counter(input: &mut Stateful<&str, Counter>) -> PResult<Vec<(i32, usize)>> {
    separated(
        1..,
        |input: &mut Stateful<&str, Counter>| {
            let num = dec_int.parse_next(&mut input.input)?;
            let count = input.state.count;
            input.state.count += 1;
            Ok((num, count))
        },
        ','
    ).parse_next(input)
}

fn main() {
    let input = "123,456,789";
    let mut input = Stateful {
        input,
        state: Counter::default(),
    };
    let result = parse_with_counter(&mut input);
    // 結果: [(123, 0), (456, 1), (789, 2)]
    dbg!(result);
}
```

### 不完全な入力を扱いたい

入力の最初のn文字しかわかっていないときに、パース可能かどうかチェックしたいという需要がある。たとえば 整数をパースするとき、入力が `12a`(以下不明)ならばパースエラーだと確定する。いっぽうで`123`(以下不明)ならば残りの入力を見ないと結果はわからない。

入力が不完全だと示すためには、`struct Partial`で入力をラップしてやる。


`struct ErrMode<E>` には「入力が不完全」というパース結果を表す値があるのでこれを見ればよい。

```rust
pub enum ErrMode<E> {
    Incomplete(Needed),
    Backtrack(E),
    Cut(E),
}
```

```rust
use winnow::prelude::*;
use winnow::ascii::dec_int;
use winnow::error::ErrMode;
use winnow::stream::Partial;

fn parse_partial(input: &mut Partial<&str>) -> PResult<i32> {
    dec_int.parse_next(input)
}

fn main() {
    // 完全な入力
    let mut complete = Partial::new("123 abc");
    assert!(matches!(parse_partial(&mut complete), Ok(123)));
    
    // 不完全かもしれない入力
    let mut incomplete = Partial::new("123");
    let result = parse_partial(&mut incomplete);
    // 入力が不完全なため、エラーになる
    assert!(matches!(result, Err(ErrMode::Incomplete(_))));
    
    // 明確にエラーとなる入力
    let mut error = Partial::new("abc");
    assert!(matches!(parse_partial(&mut error), Err(ErrMode::Backtrack(_))));
}
```

## よく使うコンビネータ

Winnowには多くの便利なコンビネータが用意されている。

### 基本的なコンビネータ

```rust
use winnow::prelude::*;
use winnow::token::take;
use winnow::combinator::{preceded, terminated, delimited};

// 前置詞付きパース: "> " の後の行を取得
fn quoted_line(input: &mut &str) -> PResult<&str> {
    preceded("> ", take_till(0.., '\n')).parse_next(input)
}

// 終端付きパース: セミコロンで終わる文
fn statement(input: &mut &str) -> PResult<&str> {
    terminated(take_till(1.., ';'), ';').parse_next(input)
}

// 囲まれたパース: 括弧の中身
fn parenthesized(input: &mut &str) -> PResult<&str> {
    delimited('(', take_till(0.., ')'), ')').parse_next(input)
}
```

### 繰り返しのコンビネータ

```rust
use winnow::prelude::*;
use winnow::ascii::{dec_int, space0};
use winnow::combinator::{separated, repeat};

// カンマ区切りの数値リスト
fn number_list(input: &mut &str) -> PResult<Vec<i32>> {
    separated(0.., dec_int, (',', space0)).parse_next(input)
}

// 固定回数の繰り返し
use winnow::ascii::dec_uint;

fn rgb_values(input: &mut &str) -> PResult<[u8; 3]> {
    repeat(3, preceded(space0, dec_uint::<_, u8, _>)).parse_next(input)
}
```

### カスタムパーサの作成

```rust
use winnow::prelude::*;
use winnow::token::{take_while, take_till, any};
use winnow::combinator::{alt, delimited, fold_repeat};
use winnow::ascii::alpha1;

// 識別子のパーサ
fn identifier(input: &mut &str) -> PResult<&str> {
    (
        alt(('_', alpha1)),
        take_while(0.., |c: char| c.is_alphanumeric() || c == '_')
    ).recognize().parse_next(input)
}

// 文字列リテラルのパーサ
fn string_literal(input: &mut &str) -> PResult<String> {
    delimited(
        '"',
        fold_repeat(
            0..,
            alt((
                take_till(1.., ['\\', '"']).map(|s: &str| s.to_owned()),
                preceded('\\', any).map(|c| format!("\\{}", c))
            )),
            String::new,
            |mut acc, item| {
                acc.push_str(&item);
                acc
            }
        ),
        '"'
    ).parse_next(input)
}
```

## エラーハンドリング

Winnowでは、用途に応じて複数のエラー型が用意されている。

```rust
use winnow::prelude::*;
use winnow::error::{ErrorKind, ContextError, ParseError};
use winnow::ascii::dec_int;

// 簡単なエラー
fn simple_parser(input: &mut &str) -> PResult<i32> {
    dec_int.parse_next(input)
}

// コンテキスト付きエラー
fn with_context(input: &mut &str) -> PResult<i32, ContextError> {
    dec_int.context("expected integer").parse_next(input)
}

// カスタムエラー型
#[derive(Debug, PartialEq)]
enum MyError {
    InvalidNumber,
    OutOfRange,
}

impl<I> ParseError<I> for MyError {
    fn from_error_kind(_input: &I, _kind: ErrorKind) -> Self {
        MyError::InvalidNumber
    }

    fn append(self, _input: &I, _kind: ErrorKind) -> Self {
        self
    }
}

fn custom_error_parser(input: &mut &str) -> PResult<u8, MyError> {
    let num = dec_int.parse_next(input)?;
    if num > 255 {
        Err(ErrMode::Cut(MyError::OutOfRange))
    } else {
        Ok(num as u8)
    }
}
```

## 実践的な例: 簡単な式のパーサ

```rust
use winnow::prelude::*;
use winnow::ascii::{dec_int, space0};
use winnow::combinator::{alt, delimited, preceded, fold_repeat};

#[derive(Debug, Clone, PartialEq)]
enum Expr {
    Number(i32),
    Add(Box<Expr>, Box<Expr>),
    Mul(Box<Expr>, Box<Expr>),
}

// 数値または括弧で囲まれた式
fn factor(input: &mut &str) -> PResult<Expr> {
    alt((
        dec_int.map(Expr::Number),
        delimited(
            ('(', space0),
            expr,
            (space0, ')')
        ),
    )).parse_next(input)
}

// 乗算
fn term(input: &mut &str) -> PResult<Expr> {
    let init = factor.parse_next(input)?;
    
    winnow::combinator::fold_repeat(
        0..,
        preceded((space0, '*', space0), factor),
        move || init.clone(),
        |acc, val| Expr::Mul(Box::new(acc), Box::new(val))
    ).parse_next(input)
}

// 加算を含む完全な式
fn expr(input: &mut &str) -> PResult<Expr> {
    let init = term.parse_next(input)?;
    
    winnow::combinator::fold_repeat(
        0..,
        preceded((space0, '+', space0), term),
        move || init.clone(),
        |acc, val| Expr::Add(Box::new(acc), Box::new(val))
    ).parse_next(input)
}

fn main() {
    let mut input = "1 + 2 * 3";
    let result = expr(&mut input);
    // 結果: Add(Number(1), Mul(Number(2), Number(3)))
    dbg!(result);
}
```

## まとめ

Winnowは型安全で柔軟なパーサコンビネータライブラリである。基本的なパーサから始めて、コンビネータを組み合わせることで複雑な構文解析も実装できる。ストリーミング、位置情報、状態管理など高度な機能も提供されており、様々なユースケースに対応できる。
