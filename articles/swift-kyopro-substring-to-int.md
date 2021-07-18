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
    .map { Int(String($0))! }
```

`split` は `[Substring]` を返しますが、前者では `Substring` を直接 `Int` に変換します。しかし、これはパフォーマンス的に劣るため、後者のように一度 `String` に変換してから `Int` に変換する必要があります。

この問題が [ABC 210 D - National Railway](https://atcoder.jp/contests/abc210/tasks/abc210_d) で顕在化し、入力を読み込むだけで TLE になる問題が発生しました。

# ABC 210 D - National Railway で起こった問題

[ABC 210 D - National Railway](https://atcoder.jp/contests/abc210/tasks/abc210_d) の入力とその制約は次の通りです。

---

![](https://storage.googleapis.com/zenn-user-upload/ff7f08b8fd376b39867543a1.png)

---

高々 1000 × 1000 の整数を読み込むだけです。 TLE になる理由はないように思えます。しかし、この[入力を読み込むだけで TLE になります](https://atcoder.jp/contests/abc210/submissions/24341958)。

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


しかし、このコードの [`.map { Int($0)! }` の部分を `.map { Int(String($0))! }` に変えるだけで TLE は消えます](https://atcoder.jp/contests/abc210/submissions/24342045)。

こんなのに本番中に気付くのは難しいと思います。 **入力の読み込み時は、必ず `Substring` を一度 `String` に変換してから `Int` に変換する** ようにしましょう。

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

前者では [`Int.init(_: String)`](https://developer.apple.com/documentation/swift/int/2927504-init) が、後者では [`Int.init<S: StringProtocol>(_: S, radix: Int)`](https://developer.apple.com/documentation/swift/int/2924481-init) が呼ばれます。

この違いに気付いて初めは、 `Substring` を渡したときは任意進数をパースする処理が入っていて重いのではないかと思いました。 `radix` のデフォルト引数があるからわかりづらいですが、後者では `radix: 10` が省略された引数として渡されていたのです。

しかし、[両者の実装](https://github.com/apple/swift/blob/swift-5.2.1-RELEASE/stdlib/public/core/IntegerParsing.swift)を確認したところ、前者は単に後者を呼び出しているだけでした。任意進数を処理するための余分な処理が行われていることがパフォーマンス低下の原因ではないようです。

```swift
extension FixedWidthInteger {
  ...
  public init?(_ description: String) {
    self.init(description, radix: 10)
  }
}
```

## `String` と `Substring` のパフォーマンス

では、なぜパフォーマンスの差が生まれるのでしょうか？ `String` と `Substring` にパフォーマンスの差があるのでしょうか？次のようなコードで計測してみます。

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
measure {
    for c in s1 {
        sum &+= c.asciiValue!
    }
}
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

計算量のオーダーは変わらないとはいえ、 65% ほど `Substring` が遅いようです。しかし、先の AtCoder における入力読み込みの実験では 3 〜 4 倍ほどもパフォーマンスが異なりました。

## Specialization

他に差を生む原因として考えられるものに [Specialization](https://github.com/apple/swift/blob/7123d2614b5f222d03b3762cb110d27a9dd98e24/docs/OptimizationTips.rst#id26) が挙げられます。つまり、 `String` の場合は `String` そのものとして扱われるのに対して、 `Substring` の場合には `StringProtocol` の [Existential として扱われている](https://heart-of-swift.github.io/protocol-oriented-programming/#existential-type-%E3%81%A8-existential-container)可能性があります。 Existential を介して値を扱うと、その分だけオーバーヘッドが発生してパフォーマンスが低下します。

:::message
`Int.init(_: String)` を呼び出した場合には、　`Int.init<S: StringProtocol>(_: S, radix: Int)` を呼ぶ箇所で `S == String` が確定しているので、標準ライブラリのビルド時に Specialization が実行されます。しかし、 `Int.init<S: StringProtocol>(_: S, radix: Int)` を直接呼び出すケースでは、標準ライブラリのビルド時に型パラメータ `S` が決定されないので Specialization できません。このように、モジュールをまたいで別々にビルドされるケースでは、一般的に型パラメータは Specialize されません。

しかし、それだと標準ライブラリの多くのジェネリック API が Specialize できないということになってしまいます。モジュールをまたいでも Specialize する方法はいくつか存在し、通常標準ライブラリの API は大抵モジュールの外から呼び出した場合でも Specialize されるようになっています。

今回は、 `String` と `Substring` を渡した場合のパフォーマンスに差があったことから、正しく Specialize されていないのではないかと疑っています。
:::

先程の二つのコードの [SIL](https://github.com/apple/swift/blob/main/docs/SIL.rst) を確認してみましょう。確認しやすいように興味のある箇所だけを `convert` 関数の中に閉じ込めます。

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

次のようにして SIL を出力し、結果から `convert` の部分だけ抜き出したのが↓です。

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

後者（ `Substring` の場合）では `Int.init` がインライン化されているようで、ずいぶんと長さが異なります。しかし予想に反して、インライン化された先で `Substring` として扱われている（ `StringProtocol` の Existential として扱われていない）様子が見られ（↓）、どうやら Specialization は働いているようです。

```
%24 = struct_element_addr %8 : $*IndexingIterator<Substring.UTF8View>, #IndexingIterator._elements // user: %25
```

`Substring` を直接 `Int` に変換した場合のパフォーマンス低下は、 Specialization の有無に起因した Existential のオーバーヘッドによるものではなさそうです。

## やっぱり `Substring` が遅い

Specialization の有無による違いではないとすると、やはり `Substring` が遅いのでしょうか。今度は次のような実験をしてみましょう。

`Substring` から直接 `Int` に変換する場合でも、 `String` を介して変換する場合でも、どちらも明示的に `Int.init<S: StringProtocol>(_: S, radix: Int)` を呼び出します。

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

measure {
    // Int.init<S: StringProtocol>(_: S, radix: Int) が呼ばれる
    let numbers = line.split(separator: " ").map { Int($0, radix: 10)! }
    sum += numbers.count
}
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

見事に差が現れました。同じ条件で試しているので、やはり **`Substring` が遅い** ようです。 65% ほど `Substring` が遅かった実験では、処理が単純すぎて（呼び出される API も異なり）違いが現れづらかったのかもしれません。

:::message
ただし、この実験だけでは Specialization の差による可能性は残されています。標準ライブラリ内で `Int.init(_: String)` 経由で呼び出されることで、 `Int.init<S: StringProtocol>(_: S, radix: Int)` は `S == String` については Specialize されることが確定しています。そのため、この実験単体では `String` を渡したケースだけ Specialize されたものが呼び出されている可能性を排除できません。

先の実験で `Int.init<S: StringProtocol>(_: S, radix: Int)` が `S == Substring` に対しても Specialize されることを確認しているので、それと合わせて考えると `Substring` が遅いのだろうと考えられます。
:::

# なぜこれまで問題が顕在化しなかったか

1000 × 1000 の整数の読み込みなどこれまでも何度もあったはずです。なぜ今回問題が顕在化したのでしょうか。

これは $1 \leq C \leq 10^9$ という制約によるものかもしれません。つまり、桁数が大きすぎて処理のオーバーヘッドが大きかったのではないかというのが僕の推測です。 1000 × 1000 のデータを読み込むことは何度もあったと思いますが、これほど桁数の大きい整数を 1000 × 1000 も読み込んだ記憶は（僕が解いた問題の中では）ありません。今回、桁数が大きい整数を 1000 × 1000 も `Substring` から `Int` に直接変換しようとしたことで問題が顕在化したのではないでしょうか。

# 結論

`Substring` は遅いです。 1000 × 1000 の整数を `Substring` から `Int` に変換しようとしただけで TLE になることがあります。必ず次のように `String` を介して変換するようにしましょう。

```swift
let numbers: [Int] = readLine()!
    .split(separator: " ")
    .map { Int(String($0))! }
```