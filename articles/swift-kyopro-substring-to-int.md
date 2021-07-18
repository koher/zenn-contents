---
title: "競プロerはSwiftでSubstringを直接Intに変換してはいけない"
emoji: "🔢️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["swift", "競技プログラミング", "atcoder"]
published: true
---

次のようなコード、書いてませんか？

```swift
// ⛔ 遅い（入力読み込みで TLE になることあり）
let numbers: [Int] = readLine()!
    .split(separator: " ")
    .map { Int($0)! }
```

これをやってはいけません。↓のように書きましょう。

```swift
// ✅ 速い
let numbers: [Int] = readLine()!
    .split(separator: " ")
    .map { Int(String($0))! } // 一度 String に変換してから Int に変換する
```

`split` は `[Substring]` を返しますが、前者では `Substring` を直接 `Int` に変換します。しかし、これはパフォーマンス的に劣るため、後者のように一度 `String` に変換してから `Int` に変換する必要があります。

この問題が [ABC 210 D - National Railway](https://atcoder.jp/contests/abc210/tasks/abc210_d) で顕在化し、入力を読み込むだけで TLE になる現象が発生しました。

# ABC 210 D - National Railway で起こった問題

[ABC 210 D - National Railway](https://atcoder.jp/contests/abc210/tasks/abc210_d) の入力とその制約は次の通りです。

---

![](https://storage.googleapis.com/zenn-user-upload/ff7f08b8fd376b39867543a1.png)

---

高々 1000 × 1000 の整数を読み込むだけです。一見、 TLE になる理由は見当たりません。しかし、 `Substring` を直接 `Int` に変換した場合、この[入力を読み込むだけで TLE になります](https://atcoder.jp/contests/abc210/submissions/24341958)[^1]。

```swift
let HWC: [Int] = readLine()!
    .split(separator: " ")
    .map { Int($0)! }
let H = HWC[0]

var A: [[Int]] = []
for _ in 0 ..< H {
    let a: [Int] = readLine()!
        .split(separator: " ")
        .map { Int($0)! }
        A.append(a)
}

print(A.count)
```

![](https://storage.googleapis.com/zenn-user-upload/4dad96f0b6464cf2766a9622.png)


なんと、このコードの [`.map { Int($0)! }` の部分を `.map { Int(String($0))! }` に変えるだけで TLE が消えます](https://atcoder.jp/contests/abc210/submissions/24342045)。しかも実行時間はわずか 579 ms 。

本番中にこんなことに気付くのは困難です。 **入力文字列から整数値を読み込むときは、必ず `Substring` を `String` に変換してから `Int` に変換する** ようにしましょう。

# なぜ問題が起こるのか？

Swift において [`String` と `Substring` は似たようなパフォーマンスで動作する](https://zenn.dev/koher/articles/swift-array-slice)はずです。なぜこのような問題が起こるのでしょうか。順を追って見てみましょう。

## `radix` が原因？

まず、 `String` と `Substring` を `Int` のイニシャライザに渡したときには異なるイニシャライザが呼ばれます。

```swift
// String
let s: String = "42"
print(Int(s)!) // Int.init(_: String) が呼ばれる
```

```swift
// Substring
let s: Substring = "42"
print(Int(s)!) // Int.init<S: StringProtocol>(_: S, radix: Int) が呼ばれる
```

前者では [`Int.init(_: String)`](https://developer.apple.com/documentation/swift/int/2927504-init) が、後者では [`Int.init<S: StringProtocol>(_: S, radix: Int)`](https://developer.apple.com/documentation/swift/int/2924481-init) が呼ばれます。前者は文字列を 10 進数の整数として、後者は `radix` に指定された任意進数の整数としてパースします。

僕がこの違いに気付いたとき、初めは `Substring` を渡したときは任意進数をパースする処理が入っていて重いのではないかと思いました。 `radix` のデフォルト引数があるからわかりづらいですが、後者では `radix: 10` が省略された引数として渡されていたのです。

しかし、[両者の実装](https://github.com/apple/swift/blob/swift-5.2.1-RELEASE/stdlib/public/core/IntegerParsing.swift)を確認したところ、前者は単に後者を呼び出しているだけでした。

```swift
extension FixedWidthInteger {
  ...
  public init?(_ description: String) {
    self.init(description, radix: 10)
  }
}
```

任意進数の整数をパースするために余分な処理が行われているわけではないようです。

## `String` と `Substring` のパフォーマンスの違い

では、なぜパフォーマンスの違いが生まれるのでしょうか？ `String` と `Substring` に大きなパフォーマンスの違いが存在するあるのでしょうか？次のようなコードで計測してみます。

```swift
// string-substring-measure.swift 
import Foundation

func measure(_ body: @escaping () -> Void) {
    let start = Date.timeIntervalSinceReferenceDate
    for _ in 0 ..< 10 {
        body()
    }
    let end = Date.timeIntervalSinceReferenceDate
    print((end - start) / 10, "sec")
}

let s1: String = (1 ... 1_000_000).map { $0.description }.joined(separator: " ")
let s2: Substring = .init(s1)

// 最適化で除去されないように sum に値を足し込んで最後に表示する
var sum: UInt8 = 0

// String の場合
measure {
    for c in s1 {
        sum &+= c.asciiValue!
    }
}

// Substring の場合
measure {
    for c in s2 {
        sum &+= c.asciiValue!
    }
}

print(sum)
```

```
$ swift -Ounchecked string-substring-measure.swift 
0.23251150846481322 sec
0.38350720405578614 sec
148
```

計算量のオーダーは変わらないとはいえ、 65% ほど `Substring` が遅いようです。しかし、先の AtCoder における入力読み込みの実験では 3 〜 4 倍ほどもパフォーマンスが異なりました。他にも原因があるかもしれません。

## Specialization

他の原因として考えられるものに [Specialization](https://github.com/apple/swift/blob/7123d2614b5f222d03b3762cb110d27a9dd98e24/docs/OptimizationTips.rst#id26) が挙げられます。

今回のケースでは、 `String` を渡した場合は文字列が `String` そのものとして扱われるのに対して、 `Substring` を渡した場合には `StringProtocol` の [Existential として扱われている](https://heart-of-swift.github.io/protocol-oriented-programming/#existential-type-%E3%81%A8-existential-container)可能性があります。 Existential を介して値を扱うと、その分だけオーバーヘッドが発生してパフォーマンスが低下します。

:::message
`Int.init(_: String)` を呼び出した場合には、　`Int.init<S: StringProtocol>(_: S, radix: Int)` を呼ぶ箇所で `S == String` が確定しているので、標準ライブラリのビルド時に（ `S == String` として） Specialization が実行されます。

一方で、 `Int.init<S: StringProtocol>(_: S, radix: Int)` を直接呼び出すケースでは、標準ライブラリのビルド時に型パラメータ `S` を決定できません。 `Int.init<S: StringProtocol>(_: S, radix: Int)` を呼び出すのは僕らが書いたコードで、標準ライブラリの外側に存在するからです。このため、 `Int.init<S: StringProtocol>(_: S, radix: Int)` の第 1 引数に `Substring` を渡す場合、その Specialization は標準ライブラリビルド時には行なえません。

このように、モジュールをまたいだ Specialization （ Cross-module Specialization ）はライブラリ側をビルドするときには実行できないという問題があります。 [Cross-module Specialization を行うには `@inlinable` などを適切に利用する必要があります](https://github.com/apple/swift-evolution/blob/main/proposals/0193-cross-module-inlining-and-specialization.md)。

標準ライブラリのジェネリック API は通常 Cross-module Specialization ができるようになっていますが、今回は、 `String` と `Substring` を渡した場合のパフォーマンスに大きな違いがあったことから、後者で正しく Specialize されていないのではないかと疑いました。
:::

先程の二つのコードの [SIL (Swift Intermediate Language)](https://github.com/apple/swift/blob/main/docs/SIL.rst) 出力を確認してみましょう。確認しやすいように興味のある箇所だけを `convert` 関数の中に閉じ込めます。

```swift
// string-to-int.swift
let s: String = "42"
print(convert(s))

@inline(never)
func convert(_ s: String) -> Int {
    Int(s)! // Int.init(_: String) が呼ばれる
}
```

```swift
// substring-to-int.swift
let s: Substring = "42"
print(convert(s))

@inline(never)
func convert(_ s: Substring) -> Int {
    Int(s)! // Int.init<S: StringProtocol>(_: S, radix: Int) が呼ばれる
}
```

次のように `-emit-sil` オプションで SIL を出力します。 SIL 出力全体を掲載すると長いので、結果から `convert` 関数の部分だけを抜き出しました。

```
$ swiftc -Ounchecked -emit-sil string-to-int.swift
（略）
// convert(_:)
sil hidden [noinline] @$s4main7convertySiSSF : $@convention(thin) (@guaranteed String) -> Int {
// %0 "s"                                         // users: %6, %10, %1
bb0(%0 : $String):
  debug_value %0 : $String, let, name "s", argno 1 // id: %1
  %2 = metatype $@thick Int.Type                  // user: %10
  %3 = integer_literal $Builtin.Int64, 10         // user: %4
  %4 = struct $Int (%3 : $Builtin.Int64)          // user: %10
  // function_ref specialized FixedWidthInteger.init<A>(_:radix:)
  %5 = function_ref @$ss17FixedWidthIntegerPsE_5radixxSgqd___SitcSyRd__lufCSi_SSTg5 : $@convention(method) (@owned String, Int, @thick Int.Type) -> Optional<Int> // user: %10
  %6 = struct_extract %0 : $String, #String._guts // user: %7
  %7 = struct_extract %6 : $_StringGuts, #_StringGuts._object // user: %8
  %8 = struct_extract %7 : $_StringObject, #_StringObject._object // user: %9
  strong_retain %8 : $Builtin.BridgeObject        // id: %9
  %10 = apply %5(%0, %4, %2) : $@convention(method) (@owned String, Int, @thick Int.Type) -> Optional<Int> // user: %11
  %11 = unchecked_enum_data %10 : $Optional<Int>, #Optional.some!enumelt // user: %12
  return %11 : $Int                               // id: %12
} // end sil function '$s4main7convertySiSSF'
```

```
$ swiftc -Ounchecked -emit-sil substring-to-int.swift
（略）
// convert(_:)
sil hidden [noinline] @$s4main7convertySiSsF : $@convention(thin) (@guaranteed Substring) -> Int {
// %0 "s"                                         // users: %3, %1
bb0(%0 : $Substring):
  debug_value %0 : $Substring, let, name "s", argno 1 // id: %1
  %2 = metatype $@thick Int.Type                  // user: %23
  %3 = struct_extract %0 : $Substring, #Substring._slice // users: %10, %11, %4
  %4 = struct_extract %3 : $Slice<String>, #Slice._base // user: %5
  %5 = struct_extract %4 : $String, #String._guts // users: %20, %9
  %6 = integer_literal $Builtin.Int64, 10         // user: %7
  %7 = struct $Int (%6 : $Builtin.Int64)          // user: %18
  %8 = alloc_stack $IndexingIterator<Substring.UTF8View> // users: %35, %24, %23, %15
  %9 = struct $String.UTF8View (%5 : $_StringGuts) // user: %12
  %10 = struct_extract %3 : $Slice<String>, #Slice._startIndex // users: %14, %12
  %11 = struct_extract %3 : $Slice<String>, #Slice._endIndex // user: %12
  %12 = struct $Slice<String.UTF8View> (%10 : $String.Index, %11 : $String.Index, %9 : $String.UTF8View) // user: %13
  %13 = struct $Substring.UTF8View (%12 : $Slice<String.UTF8View>) // user: %14
  %14 = struct $IndexingIterator<Substring.UTF8View> (%13 : $Substring.UTF8View, %10 : $String.Index) // user: %15
  store %14 to %8 : $*IndexingIterator<Substring.UTF8View> // id: %15
  %16 = alloc_stack $Optional<Int>                // users: %34, %33, %23
  %17 = alloc_stack $Int                          // users: %32, %23, %18
  store %7 to %17 : $*Int                         // id: %18
  // function_ref static FixedWidthInteger._parseASCIISlowPath<A, B>(codeUnits:radix:)
  %19 = function_ref @$ss17FixedWidthIntegerPsE19_parseASCIISlowPath9codeUnits5radixqd_0_Sgqd__z_qd_0_tStRd__sAARd_0_SU7ElementRpd__r0_lFZ : $@convention(method) <τ_0_0 where τ_0_0 : FixedWidthInteger><τ_1_0, τ_1_1 where τ_1_0 : IteratorProtocol, τ_1_1 : FixedWidthInteger, τ_1_0.Element : UnsignedInteger> (@inout τ_1_0, @in_guaranteed τ_1_1, @thick τ_0_0.Type) -> @out Optional<τ_1_1> // user: %23
  %20 = struct_extract %5 : $_StringGuts, #_StringGuts._object // user: %21
  %21 = struct_extract %20 : $_StringObject, #_StringObject._object // user: %22
  strong_retain %21 : $Builtin.BridgeObject       // id: %22
  %23 = apply %19<Int, IndexingIterator<Substring.UTF8View>, Int>(%16, %8, %17, %2) : $@convention(method) <τ_0_0 where τ_0_0 : FixedWidthInteger><τ_1_0, τ_1_1 where τ_1_0 : IteratorProtocol, τ_1_1 : FixedWidthInteger, τ_1_0.Element : UnsignedInteger> (@inout τ_1_0, @in_guaranteed τ_1_1, @thick τ_0_0.Type) -> @out Optional<τ_1_1>
  %24 = struct_element_addr %8 : $*IndexingIterator<Substring.UTF8View>, #IndexingIterator._elements // user: %25
  %25 = struct_element_addr %24 : $*Substring.UTF8View, #Substring.UTF8View._slice // user: %26
  %26 = struct_element_addr %25 : $*Slice<String.UTF8View>, #Slice._base // user: %27
  %27 = struct_element_addr %26 : $*String.UTF8View, #String.UTF8View._guts // user: %28
  %28 = struct_element_addr %27 : $*_StringGuts, #_StringGuts._object // user: %29
  %29 = struct_element_addr %28 : $*_StringObject, #_StringObject._object // user: %30
  %30 = load %29 : $*Builtin.BridgeObject         // user: %31
  strong_release %30 : $Builtin.BridgeObject      // id: %31
  dealloc_stack %17 : $*Int                       // id: %32
  %33 = load %16 : $*Optional<Int>                // user: %36
  dealloc_stack %16 : $*Optional<Int>             // id: %34
  dealloc_stack %8 : $*IndexingIterator<Substring.UTF8View> // id: %35
  %36 = unchecked_enum_data %33 : $Optional<Int>, #Optional.some!enumelt // user: %37
  return %36 : $Int                               // id: %37
} // end sil function '$s4main7convertySiSsF'
```

後者（ `Substring` ）の場合、 `Int.init` がインライン化されているようでずいぶんと長さが異なります。しかし予想に反して、渡された文字列は `Substring` として扱われている（ `StringProtocol` の Existential として扱われていない）ようです。たとえば、↓などに `Substring` のまま扱われている様子が見られます。

```
%24 = struct_element_addr %8 : $*IndexingIterator<Substring.UTF8View>, #IndexingIterator._elements // user: %25
```

どうやら、 `Substring` を渡した場合でも Specialization は適切に行われているようです。パフォーマンス低下の原因は、 Specialization の有無に起因した Existential のオーバーヘッドによるものではないようです。

## やっぱり `Substring` が遅い

Specialization の有無による違いではないとすると、やはり `Substring` が遅いのでしょうか。

今度は次のような実験をしてみましょう。 `String`, `Substring` どちらの場合でも、 `radix` を与えることで `Int.init<S: StringProtocol>(_: S, radix: Int)` を呼び出します。

```swift
// convert-time.swift
import Foundation

func measure(_ body: @escaping () -> Void) {
    let start = Date.timeIntervalSinceReferenceDate
    for _ in 0 ..< 10 {
        body()
    }
    let end = Date.timeIntervalSinceReferenceDate
    print((end - start) / 10, "sec")
}

let line = (1 ... 1_000_000).map { $0.description }.joined(separator: " ")

// 最適化で除去されないように sum に値を足し込んで最後に表示する
var sum: Int = 0

// String の場合
measure {
    // Int.init<S: StringProtocol>(_: S, radix: Int) が呼ばれる
    let numbers = line.split(separator: " ").map { Int($0, radix: 10)! }
    sum += numbers.count
}

// Substring の場合
measure {
    // Int.init<S: StringProtocol>(_: S, radix: Int) が呼ばれる
    let numbers = line.split(separator: " ").map { Int(String($0), radix: 10)! }
    sum += numbers.count
}

print(sum)
```

```
$ swift -Ounchecked convert-time.swift 
1.2229918003082276 sec
0.3543915987014771 sec
20000000
```

見事に差が現れました。同じ条件で試しているので、やはり **`Substring` が遅い** ようです。 65% ほど `Substring` が遅かった実験では、処理が単純すぎて（呼び出される API も異なり）違いが現れづらかったのだと思います。

:::message
この実験は先程の Specialization の実験と合わせて意味を持つことに注意して下さい。この実験単体では、 `String` の方が速い理由が Specialization であることを否定できません。

標準ライブラリの API である `Int.init(_: String)` が `Int.init<S: StringProtocol>(_: S, radix: Int)` を呼び出しているため、標準ライブラリビルド時に `Int.init<S: StringProtocol>(_: S, radix: Int)` は `S == String` について Specialize されます。そのため、この実験単体では `String` の場合だけ Specialize された可能性を排除できません。

先の実験で `Int.init<S: StringProtocol>(_: S, radix: Int)` が Cross-module で Specialize されることを確認しているので、それと合わせて `Substring` が遅いのだろうと結論付けられます。
:::

# なぜこれまで問題が顕在化しなかったか

1000 × 1000 の整数の読み込みは、これまでも何度もあったはずです。なぜこれまで問題が顕在化しなかったのでしょうか。

僕の推測ですが、これは $1 \leq C \leq 10^9$ という制約によるものだと思います。つまり、桁数が大きすぎて処理のオーバーヘッドが大きかったのではないかということです。パフォーマンスの悪い `Substring` の API にアクセスする回数は、読み込む整数文字列の桁数に比例します。

1000 × 1000 のデータを読み込むことは何度もあったと思いますが、これほど桁数の大きい整数を 1000 × 1000 も読み込んだ記憶は（僕が解いた問題の中では）ありません。今回、桁数が大きい整数を 1000 × 1000 も `Substring` から `Int` に直接変換しようとしたことで問題が顕在化したのではないかと考えられます。

# 結論

`Substring` は遅いです。 1000 × 1000 の整数を `Substring` から `Int` に変換しようとすると、それだけで TLE になることがあります。必ず次のように `String` を介して変換するようにしましょう。

```swift
let numbers: [Int] = readLine()!
    .split(separator: " ")
    .map { Int(String($0))! }
```

[^1]: たまに TLE にならず、ぎりぎりで通ります。