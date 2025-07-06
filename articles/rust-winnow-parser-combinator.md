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

その柔軟性の代償として、多くののtraitとstructが組み合わさった一見複雑な設計になっているのは仕方がない(nomやcombineも似たようなものです)

## 主要な型

### `trait Parser<I, O, E>`

パーサを表現するtrait。だいたい `FnMut(&mut I) -> Result<O, E>` 相当。

コンビネータで組み合わせることができる。例: `(p1, p2)` から「`p1`でパースしてから`p2`でパースし、結果をタプルで返す」パーサを作れる。

### `trait Stream`

```rust
trait Stream {
    type Token: Debug;
    type Slice: Debug;
    type IterOffsets: Iterator<Item = (usize, Self::Token)>;
    type Checkpoint: Offset + Clone + Debug;
}
```

`Token`の列を表現するtrait。Winnowは任意のトークン列を入力できるよう設計されており、`&str`(`char`の列)や`&[u8]`(`u8`の列)、あるいはトーカナイズした結果などをパース対象にできる。

パース時には内部的なイテレータを進めるが、`fn checkpoint(&self) -> Checkpoint`および`fn reset(&mut self, cp: &Checkpoint)`を使用して以前の位置に戻ることもできる。

ストリームの能力に応じて、位置を扱うための`Location`や、トークン列を`T`型の値とマッチさせる `Compare<T>`トレイトなどが追加で実装される。

### `trait ParserError`

エラーを扱うtrait。

バックトラックを扱うためには、追加で`trait ModalError`が必要。

## 基本的な使い方

### 一番シンプルなやつ

カンマ区切りの数字2つをパースしたい。

```rust
fn parse<'a>(input: &'a str) -> Result<(i32, i32), ()> {
    // "num1,num2"を入力とする例
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
fn parse<'a>(input: &'a str) -> ModalResult<(i32, i32), ()> {
    // "(num1,num2)" もしくは "num1,num2" を入力とする例
}
```

### 位置を扱いたい

入力が`impl Location`である必要がある。典型的には、入力を`struct LocatintSlice`でラップする。

```rust
fn parse<'a>(input: &'a mut LocatingSlice<&str>) -> Result<((i32, usize), (i32, usize)), ()> {
    // "num1,num2" を入力として、(読んだ数字, 位置) を2つ返すパーサ
}

fn main() {
    let input = "123,456";
    let mut input = LocatingSlice::new(input);
    dbg!(parse(&mut input));
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
fn parse<'a>(input: &'a mut LocatingSlice<&str>) -> Result<Vec<(i32, usize), ()> {
    // "num1,num2,..." を入力として、(読んだ数字, 連番) 返すパーサ
}

fn parse_str(&str) -> Result<((i32, usize), (i32, usize))> {
    let mut input = LocatingSlice::new(input);
    parse(&mut input)
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
fn parse<'a>(input: &'a mut Partial<&str>) -> ModalResult<i32, ()> {
    // 数字をパースするパーサ
}

fn parse_str(input: &str) -> ModalResult<i32, ()> {
    let mut input = Partial::new(input);
    parse(&mut input)
}
```
