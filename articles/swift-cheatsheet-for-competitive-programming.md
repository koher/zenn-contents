---
title: "競技プログラマのためのSwiftチートシート"
emoji: "💯"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["swift", "競技プログラミング"]
published: true
---

**Swift 未経験者が Swift で競技プログラミングに挑戦してみるための**、 Swift の基本構文や標準ライブラリの**チートシートです**。競技プログラミングで必要そうなものに絞って掲載しています。より詳しい情報は公式ドキュメント ["The Swift Programming Language"](https://docs.swift.org/swift-book/) を御覧下さい。

本チートシートの各項目に素早くアクセスするには、ページ右下の "Contents" をご活用下さい。

# Hello World

```swift
print("Hello, World!")
```

Swift では `main` 関数は必要ありません。 Python や Ruby のように↑の 1 行で完成されたコードです。

`print` の後はデフォルトで改行します。改行をなくすには次のように書きます。

```swift
print("Hello,", terminator: "") // 改行なし
print(" World!")
```

# 変数・定数

## 変数（ `var` ）

変数宣言には `var` を使います。

```swift
var a = 42
```

変数には再代入可能です。

```swift
// 変数は再代入可
var a = 2
a = 3 // ✅ OK
```

Swift には型推論があるので型の記述を省略できますが、明記する場合は次のように `:` に続けて型を書きます。

```swift
var a: Int = 42
```

## 定数（ `let` ）

定数宣言には `let` を使います。

```swift
let a = 42
```

定数には再代入**できません**。

```swift
// 定数は再代入不可
let a = 2
a = 3 // ⛔ コンパイルエラー
```

定数宣言と初期化は別々にすることができ、条件分岐と組合せて初期化することも可能です。

```swift
// 条件分岐と定数の遅延初期化
let a: Int
if b < 100 {
    a = 2
} else {
    a = 3
}
```

# 演算子

基本的に C や Java などと同じです。ただし、**インクリメントおよびデクリメントのための演算子 `++`, `--` は廃止されて利用できない**ので注意して下さい。

## 四則 & 剰余演算子（ `+`, `-`, `*`, `/`, `%` ）

| 演算子 | 意味 | 例 |
|:--|:--|:--|
| `+` | たし算 | `2 + 3` |
| `-` | ひき算 | `3 - 2` |
| `*` | かけ算 | `2 * 3` |
| `/` | わり算 | `6 / 2` |
| `%` | 剰余演算 | `8 % 3` |

整数（ `Int` ）同士のわり算の結果は `Int` に丸められる（切り捨て）ので注意して下さい。

```swift
7 / 2     // 3
7.0 / 2.0 // 3.5
```

Swift の演算子は関数なので、次のようにして高階関数の引数に渡すことができます。

```swift
// 1 から 100 までの合計（演算子を関数として利用）
let sum = (1 ... 100).reduce(0, +) // 5050
```

## 比較演算子（ `==`, `!=`, `<`, `>`, `<=`, `>=` ）

| 演算子 | 意味 | 例 |
|:--|:--|:--|
| `==` | 等しい | `a == 2` |
| `!=` | 等しくない | `a != 2` |
| `<` | より小さい | `a < 2` |
| `>` | より大きい | `a > 2` |
| `<=` | 以下 | `a <= 2` |
| `>=` | 以上 | `a >= 2` |

Swift の演算子は関数なので、次のようにして高階関数の引数に渡すことができます。

```swift
// 演算子を関数として利用（降順にソート）
let descending = [2, 5, 3].sorted(by: >) // [5, 2, 3]
```

## 論理演算子（ `&&`, `||`, `!` ）

| 演算子 | 意味 | 例 |
|:--|:--|:--|
| `&&` | かつ（ AND ） | `0 <= a && a < 100` |
| `||` | または（ OR ） | `a < 0 || 100 <= a` |
| `!` | 否定（ NOT ） | `!(0 <= a && a < 100)` |

`&&` および `||` は短絡評価されます。

```swift
// 短絡評価
1 < 2 && isFoo() // 1 < 2 が true なので isFoo() は実行されない
```

## ビット演算子（ `&`, `|`, `~`, `^` ）

| 演算子 | 意味 | 例 |
|:--|:--|:--|
| `&` | ビット単位 AND | `a & 0xff` |
| `|` | ビット単位 OR | `a | 0xff` |
| `~` | ビット単位 NOT | `~a` |
| `^` | ビット単位 XOR | `a ^ b` |

## シフト演算子（ `<<`, `>>` ）

| 演算子 | 意味 | 例 |
|:--|:--|:--|
| `<<` | 左シフト | `1 << 16` |
| `>>` | 右シフト | `1 << 16` |

## 複合代入演算子（ `+=`, `-=`, `*=`, `/=` など）

| 演算子 | 意味 | 例 |
|:--|:--|:--|
| `+=` | `a += b` は `a = a + b` | `a += 100` |
| `-=` | `a -= b` は `a = a - b` | `a -= 100` |
| `*=` | `a *= b` は `a = a * b` | `a *= 100` |
| `/=` | `a /= b` は `a = a / b` | `a /= 100` |

その他にも `%=`, `&=`, `<<=` など多様な複合代入演算子があります。

### インクリメント（ `+= 1` ）、デクリメント（ `-= 1` ）

Swift には `++` や　`--` のようなインクリメントおよびデクリメントのための演算子はないので、 `+=` や `-=` を利用します。

```swift
a += 1 // インクリメント
a -= 1 // デクリメント
```

## 三項演算子（ `a ? b : c` ）

```swift
// a < 100 が真なら "A" 、そうでなければ "B"
let result: String = a < 100 ? "A" : "B"
```

## オーバーフロー（ `&+`, `&-` など）

Swift の `+` や `-` にはオーバーフローが許されていません。オーバーフローした場合は実行時エラーになります。意図的にオーバーフローさせたい場合には `&` を付けた `&+`, `&-` などの演算子を利用します。

```swift
let a: UInt8 = 0xFF
a &+ 1 // 0
```

# 基本的な型

## 整数（ `Int` など）

```swift
let a: Int = 42
```

Swift の `Int` は実行環境によって 32 bit か 64 bit か変化しますが、 **2020 年現在で Swift が動作する環境では基本的に 64 bit** と考えて良いです。

今の環境でどちらかを調べるには次のようなコードを実行します。

```swift
// Int が 32 bit か 64 bit か確認
// 4 と表示されれば 32 bit
// 8 と表示されれば 64 bit
print(MemoryLayout<Int>.size)
```

環境によらず固定サイズの整数型もあり、標準で `Int8`, `Int16`, `Int32`, `Int64` が用意されています。また、非負整数を表す `UInt` や、固定サイズ非負整数 `UInt8`, `UInt16`, `UInt32`, `UInt64` もあります。 1 バイトを表すときは `UInt8` を使います。

### 整数型の最大・最小値（ `.max`, `.min` ）

`static` プロパティ `max`, `min` を使うことで、最大・最小値を取得できます。

```swift
// 最大・最小値の取得
Int.max    // 9223372036854775807
UInt16.max // 65535
Int8.min   // -128
```

### 整数型への変換

Swift は基本的に暗黙の型変換を許さないので、整数型同士であったとしても型変換が必要です。

```swift
// Int32 → Int の変換
let a: Int32 = 42
let b: Int = Int(a)
```

この `Int16(a)` は**キャストではなく**、単に `Int16` 型のイニシャライザ（コンストラクタのようなもの）を呼び出しているだけです。整数型や浮動小数点数型は互いに変換できるように、変換のためのイニシャライザがそれぞれの型に用意されています。

`String` からの変換に失敗した場合（整数に変換できない文字列（ `"abc"` など）が与えられた場合）、 `nil` が返されます。そのため戻り値の型は `Int` ではなく `Int?` （ `Optional<Int>` ）になります。競プロでは失敗するような変換を行わないことが多いので、ほとんどのケースは `!` を付けて `nil` を無視し、 `Int` に変換することができます。

```swift
// String → Int の変換
let s: String = "42"
let a: Int = Int(s)! // ! で nil を無視して Int? を Int に変換
```

### 整数リテラル（ `42` ）

```swift
let a = 0b101010 // 2 進数
let b = 0o52     // 8 進数
let c = 42       // 10 進数
let d = 0x2a     // 16 進数
```

桁数が多いときには、リテラル中に `_` を挿入して可読性を向上することもできます。

```swift
let a = 1_000_000 // 100 万
```

**Swift ではリテラル自体に型はありません。** `ExpressibleByIntegerLiteral` というプロトコルに適合した型であれば何でも整数リテラルで表すことができます。

```swift
// この場合、 42 というリテラルは UInt 型
let a: UInt = 42
```

これはリテラル `42` が `Int` 型で、それが `UInt` 型に暗黙の型変換されているのではありません。このリテラルが `UInt` 型の定数 `a` に代入されることから、コンパイラがリテラルの型を `UInt` に決定しています。

型の記述が省略された場合は、整数リテラルは `Int` 型と推論されます。

```swift
// この場合、 42 というリテラルは Int 型
let a = 42
```

## 浮動小数点数（ `Double` など）

```swift
let a: Double = 42.0
```

`Double` は倍精度（ 64 bit ）浮動小数点数型です。単精度（ 32 bit ）浮動小数点数型の `Float` もありますが、競プロにおいて利用機会は少ないでしょう。

### 浮動小数点数型への変換

基本的に整数型と同じです。数値型同士でも明示的な型変換が必要です。

```swift
// Int → Double の変換
let a: Int = 42
let b: Double = Double(a)
```

`String` からの変換は失敗し `nil` が返されるおそれがあるので `Double?` （ `Optional<Double>` ）となります。変換失敗が考えられない場合は `!` を付けて `nil` を無視します。

```swift
// String → Double の変換
let s: String = "42.0"
let a: Double = Double(s)! //  ! で nil を無視して Double? を Double に変換
```

### 無限大（ `.infinity` ）と NaN （ `.nan` ）

```swift
Double.infinity  // +Inf
-Double.infinity // -Inf
Double.nan       // NaN
```

NaN のチェックには `isNaN` プロパティを用います。

```swift
// NaN のチェック
if a.isNaN {
    ...
}
```

### π（ `.pi` ）と e （ `M_E` ）

```swift
Double.pi // 3.1415926535897931
```

Swift 5.3 現在 `Double.e` はないので、 libc の `M_E` を↓のようにして利用します。

```swift
import Foundation
M_E // 2.7182818284590451
```

### 浮動小数点数リテラル（ `42.0` ）

```swift
let a = 42.0
let b = 4.2e1 // これも 42.0
```

`Double` や `Float` は `ExpressibleByIntegerLiteral` プロトコルに適合しているので、浮動小数点数リテラルの他に整数リテラルを用いることもできます。

```swift
let a: Double = 42 // ✅ OK
```

ただし、**暗黙の型変換ではない**ので↓はできません。

```swift
let a = 42        // a は Int 型
let b: Double = a // ⛔ コンパイルエラー
```

## 真偽値（ `Bool` ）

```swift
let a: Bool = true
let b: Bool = false
```

`toggle` メソッドで値を反転させることができます。

```swift
// toggle メソッドによる Bool 値の反転
var f = true
f.toggle()
print(f) // false
```

`toggle` メソッドを利用するためにはその真偽値が可変であること（ `var` に代入されているなど）が必要です。

## 文字列（ `String` ）

```swift
let s: String = "abc"
```

### 文字列の文字数（ `count`, `isEmpty` ）

```swift
"abc".count // 3
```

Swift では次のような絵文字も 1 文字としてカウントできます。

```swift
"👨‍👩‍👧‍👦".count // 1
```

空文字列かどうかは `isEmpty` で確認できます。

```swift
"abc".isEmpty // false
"".isEmpty    // true
```

### 文字列の分割（ `split` ）

```swift
"A B C".split(separator: " ") // ["A", "B", "C"]
```

競プロでは入力文字列を分割して整数に変換することが多いと思いますが、次のように書けます。

```swift
"2 3 5".split(separator: " ").map { Int($0)! } // [2, 3, 5]
```

### 文字列の結合（ `joined` ）

```swift
let ss: [String] = ["A", "B", "C"]
ss.joined(separator: " ") // "A B C"
```

`Int` の `Array` （ `[Int]` ）を結合する場合などは次のように書けます。

```swift
let numbers: [Int] = [2, 3, 5]
numbers.map(String.init).joined(separator: " ") // "2 3 5"
```

### 文字列のフォーマット（ "String(format:_:)" ）

C 形式のフォーマットが可能です。

```swift
let y = 2020
let m = 10
let d = 24
let s = String(format: "%04d-%02d-%02d", y, m, d)
print(s) // 2020-10-24
```

### 文字列型への変換

```swift
// Int → String の変換
let a: Int = 42
let s: String = String(a) 
```

また、 `CustomStringConvertible` に適合した型であれば `description` プロパティで `String` に変換することもできます。

```swift
// Int → String の変換（ description の利用）
let a: Int = 42
let s: String = a.description
```

さらに、すべての式は文字列補間を使って文字列リテラルに埋め込めます。

```swift
// Int → String の変換（文字列補間の利用）
let a: Int = 42
let s: String = "\(a)"
```

### 文字列リテラル（ `"abc"` ）

```swift
let s = "abc"
```

`"""` を使って複数行の文字列リテラルを書くこともできます。

```swift
// 複数行の文字列リテラル（ " も使える）
let s = """
<ul id="foo">
    <li>Python</li>
    <li>JavaScript</li>
    <li>Swift</li>
</ul>
"""
```

#### 文字列補間（ `"\(a)"`）

`\()` を使って文字列リテラルに式を埋め込めます。

```swift
print("2 + 3 = \(2 + 3)") // 2 + 3 = 5
```

競プロにおいては、実行結果として複数値を出力しなければならない場合などに便利です。

```swift
// スペース区切りで a, b, c の値を出力
print("\(a) \(b) \(c)")
```

### コレクションとしての文字列

Swift の `String` は `Character` （文字）のコレクションです。 `Array` などと同じく `for-in` 文でイテレーションできます。

```swift
// 文字列とイテレーション
let s = "abc"
for c in s {
    print(c)
}
```

```
a
b
c
```

ただし、インデックスを指定して文字にランダムアクセスすることはできません（絵文字の文字数を上手く数えられることから推測できるように `Character` のサイズは固定でないので、 i 文字目のアドレスを知るには前からトラバースしなければならないため）。繰り返しランダムアクセスが必要な場合は一度 `Array` に変換するのが良いでしょう。

```swift
// 文字列中の文字へのランダムアクセス
let s: String = "abcdefg"
let cs: [Character] = Array(s)
print(cs[2]) // c
```

### 部分文字列（ `Substring` ）

```swift
let s: Substring = "abcdefg".dropFirst(3) // "defg"
```

Swift では部分文字列型 `Substring` は `String` と型として区別されます。 `String` に変換する必要がある場合は↓のようにします。

```swift
let a: Substring = "abcdefg".prefix(3) // "abc" ( Substring )
let s: String = String(a)              // "abc" ( String )
```

なぜ `String` と `Substring` の型が分かれているかについては[こちら](https://www.youtube.com/watch?v=D6exg8DCNC0&t=36)。

## 文字（ `Character` ）

```swift
let c: Character = "a"
```

`String` は `Character` のコレクションなので、 `String` の要素は `Character` です。

```swift
// String の最初の要素を取得
let c: Character = "abc".first! // "a"
```

↑で `!` が必要なのは `nil` が返される可能性があるからです。空文字列の場合 `first` で最初の文字を取得することができないため `nil` が返されます。そのため、 `first` の戻り値の型は `Character?` （ `Optional<Character>` ）です。空文字列でないことがわかっている場合、 `!` で `nil` を無視し、 `Character?` を `Character` に変換できます。

### 文字リテラル（ `"a"` ）

```swift
let c: Character = "a"
```

文字列リテラルと同じに見えますが、これは文字（正確には書記素クラスタ）リテラルです。詳細は[こちら](https://developer.apple.com/documentation/swift/expressiblebyextendedgraphemeclusterliteral)。

## 配列（ `Array` ）

```swift
let a: [Int] = [2, 3, 5]
print(a[0]) // 2
```

### 配列の型（ `[Int]`, `Array<Int>` など）

`[Int]` は `Array<Int>` のシンタックスシュガーです。↓の二つはまったく同じ意味です。

```swift
let a: [Int] = [2, 3, 5]
```

```swift
let a: Array<Int> = [2, 3, 5]
```

通常は `[Int]` の形式が用いられることが多いです。

### 配列の要素数（ `count`, `isEmpty` ）

```swift
[2, 3, 5].count // 3
```

```swift
[2, 3, 5].isEmpty // false
[].isEmpty        // true
```

### 配列の変更

Swift の `Array` は `class` ではなく `struct` なことに注意が必要です。そのため、 `let` （定数）ではなく `var` （変数）に格納されてないと変更することができません。

```swift
// let だと Array を変更できない
let a: [Int] = [2, 3, 5]
a[2] = 4 // ⛔ コンパイルエラー
```

```swift
// var だと Array を変更できる
var a: [Int] = [2, 3, 5]
a[2] = 4 // ✅　OK
```

また、 `struct` のインスタンスは代入時にコピーされ共有されることがないため、 `Array` の代入後に一方を変更してももう一方には影響を与えません。

```swift
// Array の一方を変更してももう一方に影響はない
var a: [Int] = [2, 3, 5]
var b: [Int] = a
a[2] = 4
print(b[2]) // 5
```

代入時に毎回コピーされるとパフォーマンスに悪影響を与えそうですが、 [Copy-on-Write](https://ja.wikipedia.org/wiki/%E3%82%B3%E3%83%94%E3%83%BC%E3%82%AA%E3%83%B3%E3%83%A9%E3%82%A4%E3%83%88) という仕組みを使い、パフォーマンスへの影響を回避しています。代入時にはバッファをコピーせず共有して、実際に必要になるまでコピーを行いません。

```swift
// Copy-on-Write の例（コピーが発生する場合）
var a: [Int] = [2, 3, 5]
var b: [Int] = a // ここではコピーされない
a.append(7)      // バッファが共有されているので変更前にコピーされる
```

```swift
// Copy-on-Write の例（コピーが発生しない場合）
var a: [Int] = [2, 3, 5]
a.append(7)      // バッファが共有されていないので変更前のコピーは不要
```

### 配列への要素の追加（ `append`, `insert` ）

```swift
// 末尾への要素の追加
var a = [2, 3, 5]
a.append(7)
print(a) // [2, 3, 5, 7]
```

```swift
// 末尾以外への要素の追加
var a = [1, 1, 2, 3, 5]
a.insert(0, at: 0)
print(a) // [0, 1, 1, 2, 3, 5]
```

`append` は $O(1)$ 、 `insert` は $O(n)$ です。

### 配列からの要素の削除（ `removeLast`, `remove`, `popLast` ）

```swift
// 末尾からの要素の削除
var a = [2, 3, 5]
let r: Int = a.removeLast()
print(a) // [2, 3]
print(r) // 5
```

```swift
// 末尾以外からの要素の削除
var a = [0, 1, 1, 2, 3, 5]
let r: Int = a.remove(at: 0)
print(a) // [1, 1, 2, 3, 5]
print(r) // 0
```

空の `Array` に対して `removeLast` を呼び出すと実行時エラーになりますが、 `popLast` の場合は `nil` を返します。そのため、 `popLast` の戻り値の型は `Optional` です（要素の型が `Int` なら `Int?` ）。

```swift
// 末尾からの要素の削除
var a = [2, 3, 5]
let r: Int? = a.popLast()
print(a) // [2, 3]
print(r) // Optional(5)
```

これを利用して、 `popLast` を使えば要素がなくなるまでループすることなどが可能です。

```swift
// 配列の要素がなくなるまでループ
var a = [2, 3, 5]
while let x = a.popLast() {
    print(x)
}
```

```
5
3
2
```

`removeLast` および `popLast` は $O(1)$ 、 `remove` は $O(n)$ です。

### 配列のイテレーション

```swift
let a: [Int] = [2, 3, 5]
for x in a {
    print(x)
}
```

```
2
3
5
```

### 配列のネスト

```swift
let a: [[Int]] = [
    [2, 3],
    [5, 7, 11],
]

print(a[0][1]) // 3
```

### 配列リテラル

```swift
let a = [2, 3, 5]
let b: [Int] = [] // 空の配列リテラル、型推論できないので型の記述が必須
```

配列リテラル自体は型を持たないので、必ずしも `Array` を表すとは限りません。 `ExpressibleByArrayLiteral` プロトコルに適合した型であれば何でも配列リテラルで表すことができます。 `Array` の他に `ArraySlice` や `Set` なども `ExpressibleByArrayLiteral` に適合しています。

```swift
// Array 以外の型を配列リテラルで表す
let a: ArraySlice<Int> = [2, 3, 5]
let b: Set<Int> = [2, 3, 5]
```

これは暗黙の型変換ではありません。左辺の型からリテラルの型が決定されています。

### 部分配列（ `ArraySlice` ）

```swift
let a: [Int] = [0, 1, 1, 2, 3, 5]
let b: ArraySlice<Int> = a[1 ..< 4]
print(b) // [1, 1, 2]
```

`a[start ..< end]` のようにすると `end` は含まれず、 `a[start ... end]` のようにすると `end` も含まれます。また、 `a[start...]` や `a[..<end]` のように、 `start` と `end` の片方だけを指定することもできます（この `start ..< end` などは特殊な構文ではなく、すべて `Range` とその亜種です）。

`array[start ..< end]` で `ArraySlice` を取得する処理は $O(1)$ です。

Swift では部分配列型 `ArraySlice` は `Array` と型として区別されます。 `Array` に変換する必要がある場合は↓のようにします。

```swift
let a: [Int] = [0, 1, 1, 2, 3, 5]
let b: ArraySlice<Int> = a[1 ..< 4] // [1, 1, 2] ( ArraySlice<Int> )
let c: [Int] = Array(b)             // [1, 1, 2] ( [Int] )
```

`Array(arraySlice)` で `Array` を生成する処理は $O(n)$ です。

なぜ `Array` と `ArraySlice` の型が分かれているかについては[こちら](https://www.youtube.com/watch?v=D6exg8DCNC0&t=36)。

## 辞書（連想配列）（ `Dictionary` ）

```swift
let a: [String: Int] = ["x": 2, "y": 3, "z": 5]
print(a["x"])  // Optional(2)
print(a["x"]!) // 2
```

ハッシュテーブルで実装されているため、基本的な操作は $O(1)$ です。

`Array` と違い、 `Dictionary` をキーで引く場合、キーに対応した値が登録されていないと `nil` が返されます。そのため、 `a["x"]` の戻り値の型は `Int` ではなく `Int?` （ `Optional<Int>` ）になります。キーに対応した値が存在することがわかっている場合は `a["x"]!` のように `!` を付けることで `nil` を無視し、（このケースであれば） `Int?` を `Int` に変換することができます。

### 辞書の型（ `[String: Int]`, `Dictionary<String, Int>` など）

`[String: Int]` は `Dictionary<String, Int>` のシンタックスシュガーです。↓の二つはまったく同じ意味です。

```swift
let a: [String: Int] = ["x": 2, "y": 3, "z": 5]
```

```swift
let a: Dictionary<String, Int> = ["x": 2, "y": 3, "z": 5]
```

通常は `[String: Int]` の形式が用いられることが多いです。

### 辞書のエントリー数（ `count`, `isEmpty` ）

```swift
["x": 2, "y": 3, "z": 5].count // 3
```

```swift
["x": 2, "y": 3, "z": 5].isEmpty // false
[:].isEmpty                      // true
```

`count` および `isEmpty` は $O(1)$ です。

### 辞書の変更

`Dictionary` も `Array` と同じく `struct` で値型です。変更するためには `let` （定数）ではなく `var` （変数）に格納されている必要があります。

```swift
// let だと Dictionary を変更できない
let a: [String: Int] = ["x": 2, "y": 3, "z": 5]
a["w"] = 7 // ⛔ コンパイルエラー
```

```swift
// var だと Dictionary を変更できる
var a: [String: Int] = ["x": 2, "y": 3, "z": 5]
a["w"] = 7 // ✅ OK
```

Copy-on-Write についても `Array` と同様です。

値の変更・追加は $O(1)$ です。

### 辞書のエントリーの削除

```swift
var a: [String: Int] = ["x": 2, "y": 3, "z": 5]
a.removeValue(forKey: "z")
print(a) // ["x": 2, "y":, 3] （順不同）
```

`removeValue` は $O(1)$ です。

### 辞書のイテレーション

```swift
let a: [String: Int] = ["x": 2, "y": 3, "z": 5]
for (key, value) in a {
    print("\(key) -> \(value)")
}
```

```
x -> 2
y -> 3
z -> 5
```

（順不同）

### 辞書のキーの存在確認（ `keys.contains` ）

```swift
["x": 2, "y": 3, "z": 5].keys.contains("x") // true
["x": 2, "y": 3, "z": 5].keys.contains("w") // false
```

`keys.contains` は $O(1)$ です。

### 辞書リテラル（ `["x": 2, "y": 3]`, `[:]` など）

```swift
let a = ["x": 2, "y": 3, "z": 5]
let b: [String: Int] = [:] // 空の辞書リテラル、型推論できないので型の記述が必須
```

## 集合（ `Set` ）

```swift
let a: Set<Int> = [2, 3, 5]
print(a.contains(3)) // true
print(a.contains(4)) // false
```

イテレーションや要素数の取得など、基本的に `Array` と同じです。

ハッシュテーブルで実装されているため、基本的な操作は $O(1)$ です。

### 集合への要素の追加（ `insert` ）

```swift
var a: Set<Int> = [2, 3, 5]
a.insert(7)
print(a) // [2, 3, 5, 7] （順不同）
```

`insert` は $O(1)$ です。

### 集合からの要素の削除（ `remove` ）

```swift
var a: Set<Int> = [2, 3, 5]
a.remove(5)
print(a) // [2, 3] （順不同）
```

`remove` は $O(1)$ です。

## レンジ（ `Range`, `ClosedRange` など）

```swift
let a: Range<Int>       = 0 ..< 100 // 100 を含まない
let b: ClosedRange<Int> = 0 ... 100 // 100 を含む
```

その他にも、上界・下界だけを定めたレンジを作ることもできます。

```swift
let c: PartialRangeUpTo<Int>    = ..<100
let d: PartialRangeThrough<Int> = ...100
let e: PartialRangeFrom<Int>    = 100...
```

レンジの生成は $O(1)$ です。

### レンジに含まれるか（ `contains` ）

```swift
let a: Range<Int> = 0 ..< 100
print(a.contains(0))   // true
print(a.contains(50))  // true
print(a.contains(100)) // false
```

### レンジのイテレーション

```swift
for i in 1 ... 3 {
    print(i)
}
```

```
1
2
3
```

`Comparable` に適合した任意の型について `Range` を作ることができますが、 `Range<String>` などはイテレートすることができません。イテレートできるのは `Range` の `Bound` （ `Range<Int>` なら `Int` ）が `Strideable` プロトコルに適合し、かつ `Bound.Stride` が `SignedInteger` プロトコルに適合する場合だけです。

## タプル（ `(Int, String)` など）

```swift
let t: (Int, String) = (42, "abc")
print(t.0) // 42
print(t.1) // abc
```

Swift のタプルはコレクションではないのでイテレートすることはできません（ `Int` と `String` など型を混ぜられるのでイテレートできるのは不自然です）。

**Swift のタプルは値型であり、値変数はスタック領域に確保されるので、タプルを使っても（オブジェクトに包まれるなどの）オーバーヘッドがありません。**

### タプルの分解

```swift
let t = (42, "abc")
let (a, b) = t
print(a) // 42
print(b) // abc
```

### ラベル付きタプル

```swift
let p: (x: Double, y: Double) = (2.0, 3.0)
print(p.x) // 2.0
print(p.y) // 3.0
```

または型推論を用いて↓のようにも書けます。

```swift
let p = (x: 2.0, y: 3.0)
print(p.x) // 2.0
print(p.y) // 3.0
```

### 空のタプル

空のタプルの型は `()` です。 `Void` は空のタプルの型 `()` のエイリアスです。↓の二つは全く同じ意味になります。

```swift
let a: Void = ()
```

```swift
let a: () = ()
```

### 1 要素のタプル

`Int` などの普通の型はすべて 1 要素のタプル `(Int)` などと等価です。

```swift
let a: (Int) = 42 // ✅ OK
let b: Int = a    // ✅ OK
```

## オプショナル（ `Optional` ）

Swift では普通の型に `nil` （ `null` 相当のもの）を代入することはできません。 `nil` を代入したい場合は `Optional` 型にします。 

```swift
let a: Int = nil // ⛔ コンパイルエラー
```

```swift
let a: Int? = nil // ✅ OK
```

`Int?` は `Int` または `nil` を表す型なので、そのままでは `Int` のように使うことはできません。

```swift
let a: Int? = 2
print(a + 1) // ⛔ コンパイルエラー
```

↑の例では `a` が `Int` ではなく `Int?` 型なので `a + 1` ができません。

### オプショナルの型（ `Int?`, `Optional<Int>` など）

`Int?` は `Optional<Int>` のシンタックスシュガーです。↓の二つはまったく同じ意味になります。

```swift
let a: Int? = nil
```

```swift
let a: Optional<Int> = nil
```

通常は `Int?` の形式が用いられます。

### 非オプショナル型への変換（ `if let` など ）

`Int?` を `Int` として使うためには `a` が `nil` でないことをチェックする必要があります。

```swift
let a: Int? = 2
if let a = a { // a が nil でなかったときだけ新しい定数 a を作って代入
    print(a + 1) // ✅ OK
}
```

この `if let a = a` のような代入を Optional Binding といい、新しい `a` の型は `Int` になります。この新しい `a` は `if` 文の範囲でのみ有効です。

Optional Binding の右辺には式を書くこともできます。そのため、次のようなコードを書く機会は多いです。

```swift
if let a = array.first {
    // array が空でない場合の処理
}
```

`if` 以外に `guard` や `while` でも Optional Binding が使えます。

### 非オプショナル型への変換（ `!` ）

もしオプショナル型が `nil` でないことがわかっている場合には `!` 演算子を使って強制的に非オプショナル型に変換（ Forced Unwrapping ）することができます。

```swift
let a: Int? = 2
print(a! + 1) // 3
```

Forced Unwrapping は適切に使えば役に立ちますが、非オプショナル型への変換と分岐を同時にしたい場合には Optional Binding を使いましょう。

```swift
// 👎 Swift に不慣れな人の書いたコード
if !array.isEmpty {
    let a = array.first!
    print(a + 1)
}
```

```swift
// 👍 Swifty なコード
if let a = array.first {
    print(a + 1)
}
```

### `nil` の代替値（ `??` ）

```swift
let a: Int = array.first ?? 0
```

`a ?? b` は `a` が `nil` だった場合に `b` を返します。 `a` の型が `Foo?` だった場合、 `b` と `a ?? b` の型は `Foo` です。

# 制御構文

## 条件分岐

### `if`

```swift
if a < 42 {
    print("真")
}
```

```swift
if a < 42 {
    print("真")
} else {
    print("偽")
}
```

```swift
if a < 42 {
    print("小")
} else if a > 42 {
    print("大")
} else {
    print("等")
}
```

`if` の後ろの条件式に `()` は不要です。 `{}` は必須です。ただし、条件式に Trailing Closure を書く場合はパースの問題で　`()` が必要です。

```swift
// 条件式に () が必要な場合
if (array.reduce(true) { $0 && $1 > 0 }) {
    // ...
}
```

### `switch`

```swift
switch a {
case 0:
    print("零")
case 1:
    print("壱")
case 2:
    print("弐")
case 3:
    print("参")
default:
    print("他")
}
```

`case` ごとの `break` は不要です。ただし、 `case` の後に書くコードが空の場合は `break` を書きます。 C や Java のように（ `break` せず） fallthrough したい場合には `fallthrough` を書きます。

Swift の `switch` は網羅的な分岐が必須なので、↑の例では `default` が必要です。 `default` がないとコンパイルエラーになります。

#### パターンマッチング

`switch` を使ってパターンマッチングすることもできます。

```swift
let a: Bool = true
let b: Bool = false
switch (a, b) {
case (true, true):
    print("真真")
case (true, false):
    print("真偽")
case (false, true):
    print("偽真")
case (false, false):
    print("真真")
}
```

↑のように分岐が網羅的な場合には `default` は不要です。

### `guard`

`guard` は満たすべき条件を記述し、条件を満たさないケースで早期脱出（ `return`, `break` など）させるためのものです。

```swift
guard score >= 60 else {
    return // 早期脱出
}
// score >= を満たす場合の処理
```

Optional Binding と組合せて利用するケースも多いです。

```swift
// guard let
guard let first = array.first else {
    return // 早期脱出
}
// first を使う処理
```

## ループ

### `while`

```swift
var a = 10
while a >= 0 {
    print(a)
    a -= 1
}
```

Optional Binding と組合せて利用するケースも多いです。

```swift
// while let
var a = [2, 3, 5]
while let x = a.popLast() {
    print(x)
}
```

```
5
3
2
```

### `repeat-while`

多くの言語で `do-while` と呼ばれるものです。 Swift では `do` は無名スコープに用いられるので、代わりに `repeat-while` になっています。

```swift
var a = 0
repeat {
    print(a)
    a += 1
} while a < 10
```

### `for-in`

```swift
let a = [2, 3, 5]
for x in a {
    print(x)
}
```

Swift には `for (int i = 0; i < n; i ++)` 形式の `for` はありません。そのようなケースでは `for` とレンジを組合せます。

```swift
// 0 から 10 までのループ（ 10 は含まず）
for i in 0 ..< 10 {
    print(i)
}
```

`for (int i = 0; i < n; i += 2)` のようにステップを設定したい場合は `stride` 関数を使います。

```swift
// ステップ 2 のループ
for i in stride(from: 0, to: 10, by: 2) {
    print(i)
}
```

## 無名スコープ

クロージャ式と区別するため、 `{ }` ではなく `do { }` が用いられます。

```swift
let a = 2
print(a) // 2
do {
    print(a) // 2
    let a = 3
    print(a) // 3
}
print(a) // 2
```

# 関数

```swift
func square(of number: Int) -> Int {
    number * number
}
```

`square` が関数名、 `of` がラベル、 `number` が引数名、その直後の `: Int` が引数の型、最後の `->` の右の `Int` が戻り値の型を表します。

この関数は↓のように呼び出せます。 `of:` がラベルとして現れていることに注意して下さい。

```swift
square(of: 3) // 9
```

ラベルと引数名が同じ場合には一つにまとめることができます。

```swift
// ラベルと引数名が同じ場合
func triangleAreaWith(base: Double, height: Double) -> Double {
    base * height / 2
}
```

↑は↓と同じ意味です。

```swift
// ラベルと引数名を別々に書いた場合
func triangleAreaWith(base base: Double, height height: Double) -> Double {
    base * height / 2
}
```

ラベルが不要な場合には `_` を書きます。

```swift
// ラベルが不要な場合
func tan(_ x: Double) -> Double {
    sin(x) / cos(x)
}
```

関数の `{ }` の中が単一の式でない場合には `return` が必要です。

```swift
// { } の中が単一の式でないので return が必要
func sum(of array: [Int]) -> Int {
    var sum = 0
    for element in array {
            sum += element
    }
    return sum // return が必須
}
```

## 関数の型パラメータ

```swift
// 型パラメータ T を使って恒等関数 id を実装
func id<T>(_ value: T) -> T {
    value
}
```

型パラメータに制約を付けることもできます。↓は型パラメータ `T` に `AdditiveArithmetic` プロトコルに適合しているという制約を付けています。

```swift
func sum<T>(of array: [T]) -> T where T: AdditiveArithmetic {
    var sum = T.zero
    for element in array {
            sum += element
    }
    return sum
}
```

制約のおかげで `T.zero` や `sum += element` における `T` 同士のたし算を記述できます。

## 関数の型

`(第 1 引数の型, 第 2 引数の型, ...) -> 戻り値の型` で関数の型を表します。

```swift
func triangleAreaWith(base: Double, height: Double) -> Double {
    base * height / 2
}

// f は (Double, Double) -> Double 型
let f: (Double, Double) -> Double = triangleAreaWith(base:height:)
f(10, 4) // 20
```

## クロージャ式（ラムダ式）

**多くの言語でラムダ式と呼ばれるものを Swift ではクロージャ式と呼びます。**

```swift
let f: (Double, Double) -> Double = { base, height in
    base * height / 2
}
```

クロージャ式の引数名を省略することもできます。その場合、順番に `$0`, `$1`, ... が割り振られます。

```swift
let f: (Double, Double) -> Double = {
    $0 * $1 / 2
}
```

## 高階関数

```swift
let a = [1, -2, 3]
a.map(abs) // [1, 2, 3]
```

Swift の演算子は関数なので高階関数に渡すこともできます。

```swift
// 演算子を関数として高階関数に渡す
[2, 3, 5].reduce(0, +) // 10 ( [2, 3, 5] の合計 )
```

クロージャ式を直接渡すこともできます。

```swift
// 高階関数にクロージャ式を渡す
let a = [2, 3, 5]
a.map({ $0 * $0 }) // [4, 9, 25]
```

最後の引数がクロージャ式の場合、クロージャ式を関数の `()` の外側に出すことができます。これを Trailing Closure と呼びます。

```swift
// Trailing Closure
let a = [2, 3, 5]
a.map() { $0 * $0 } // [4, 9, 25]
```

さらに、 Trailing Closure にした場合、 `()` の中が空だと `()` を省略することができます。

```swift
// () の省略
let a = [2, 3, 5]
a.map { $0 * $0 } // [4, 9, 25]
```

### 標準ライブラリの高階関数

```swift
[2, 3, 5].map { $0 * $0 } // [4, 9, 25]
[0, 1, 1, 2, 3, 5].filter { $0.isMultiple(of: 2) } // [0, 2]
(1 ... 100).reduce(0, +) // 5050
[[2], [3, 5], [7, 11, 13]].flatMap { $0 } // [2, 3, 5, 7, 11, 13]
["2", "3", "four", "5", "six", "7"].compactMap { Int($0) } // [2, 3, 5, 7]
[2, 3, 5, 7, 11, 13].prefix(while: { $0 < 10 }) // [2, 3, 5, 7]
[2, 3, 5, 7, 11, 13].drop(while: { $0 < 10 }) // [11, 13]
[3, 2, 4, 5, 1].sorted(by: >) // [5, 4, 3, 2, 1]
```

## `inout` 引数

`inout` 引数によって参照渡し（のようなこと）ができます。 `inout` 引数には関数の中から結果を書き込むことができ、それが呼び出し元に反映されます。

```swift
func swap<T>(_ a: inout T, _ b: inout T) {
    let temp = a
    a = b
    b = temp
}
```

`inout` 引数に値を渡すときには `&` を付けます。 `inout` 引数に渡せるのは左辺値（ l-value ）のみです。

```swift
var x = 2
var y = 3
swap(&x, &y)

print(x) // 3
print(y) // 2
```

# 標準入出力

## 標準入力

```swift
// 単一行を読み込み
let line: String = readLine()!
```

`readLine` はこれ以上読み込める行がない場合 `nil` を返します。そのため、 `readLine` の戻り値の型は `String?` （ `Optional<String>` ）です。 `nil` にならないことがわかっているなら `!` で `nil` を無視し、 `String?` を `String` に変換します。

↓のような入力を `[Int]` に変換したい場合、

```
2 3 5
```

`map` 等と組み合わせて次のように書けます。

```swift
let a: [Int] = readLine()!.map { Int($0)! }
```

## 標準出力

```swift
print(42)
```

`print` はデフォルトで出力後改行します。改行が不要の場合は `terminator` に `""` を渡します。

```swift
print("A", terminator: "")
print("B", terminator: "")
print("C")
```

```
ABC
```