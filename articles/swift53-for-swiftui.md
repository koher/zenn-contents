---
title: "SwiftUIを楽にするSwift 5.3の新機能"
emoji: "🦅"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["swift", "swiftui"]
published: true
---

昨日 [Swift 5.3 がリリースされました](https://swift.org/blog/)（ Xcode 12 に含まれます）。マイナーバージョンアップということもあり、言語自体に大きな変化があるわけではありませんが、 **SwiftUI にとっては大きなアップデート** です。なぜなら、

- クロージャ式の中のほとんどの `self.` を省略できるようになる
- `@ViewBuilder` で `if let` や `switch` が使えるようになる

からです。

# `self.` の省略

```swift
// BEFORE (Swift 5.2)
Button("OK") {
    self.isEnabled = true
}
```

```swift
// AFTER (Swift 5.3)
Button("OK") {
    isEnabled = true
}
```

Swift ではこれまで `@escaping` な引数にクロージャ式を渡す場合、クロージャ式の中から `self` のメンバにアクセスしようとすると、明示的に `self.` を書く必要がありました。しかし、 Swift 5.3 では SwiftUI のコードを書く多くのケースで `self.` を省略できるようになります。

- [SE-0269: Increase availability of implicit `self` in `@escaping` closures when reference cycles are unlikely to occur](https://github.com/apple/swift-evolution/blob/master/proposals/0269-implicit-self-explicit-capture.md)

## そもそも何のために `self.` を書かないといけなかったのか

そもそも `self.` が強制されていた理由は何でしょうか。それは **循環参照によるメモリリーク** を避けるためです。

たとえば、次の `Clock` クラスは、 `Clock` と `Timer` が互いに参照しあっているため、他から参照されていなくても解放されません。お互いの参照をカウントするため、参照カウントが 0 にならないからです。

```swift
// 循環参照の例

// 1 秒ごとに現在の秒数を表示するクラス
final class Clock {
    private var timer: Timer!
    private var seconds: Int = 0
    init() {
        timer = Timer.scheduledTimer(withTimeInterval: 1.0,
                repeats: true) { _ in
            print(self.seconds)
            self.seconds += 1
        }
    }
    deinit { timer.invalidate() }
}
```

↓のように、たとえ生成した `Clock` インスタンスを即座に捨てても、どこからも参照されていない状態でこの `Clock` と `Timer` インスタンスは生き続けます。これはメモリリークです。

```swift
_ = Clock() // これでも `Clock` と `Timer` は解放されない
```

この場合の正しい実装は、 `[weak self]` を使って `self` を `weak` キャプチャ（弱参照）することです。

```swift
// 循環参照を回避する実装

// 1 秒ごとに現在の秒数を表示するクラス
final class Clock {
    private var timer: Timer!
    private var seconds: Int = 0
    init() {
        timer = Timer.scheduledTimer(withTimeInterval: 1.0,
                repeats: true) { [weak self] _ in
            guard let self = self else { return }
            print(self.seconds)
            self.seconds += 1
        }
    }
    deinit { timer.invalidate() }
}
```

この場合 `Timer` に渡したクロージャ式は `self` を `weak` キャプチャしているので、 `self` の参照カウントを増やしません。そのため、外部から `Clock` インスタンスへの参照がなくなれば `Clock` インスタンスは解放され、そうすると `Clock` インスタンスが保持していた `Timer` への参照もなくなるので `Timer` インスタンスも解放されます。

もしこのケースで `self.` が強制されなかった場合、どうなるでしょうか。たとえば、次のコードの中には一つも `self` が現れていません。注意深くコードを書かなければ、 `self` をキャプチャしていることを見逃し、循環参照によるメモリリークを生んでしまうかもしれません。

```swift
// 意図せず self をキャプチャする例

// 1 秒ごとに現在の秒数を表示するクラス
final class Clock {
    private var timer: Timer!
    private var seconds: Int = 0
    init() {
        timer = Timer.scheduledTimer(withTimeInterval: 1.0,
                repeats: true) { _ in
            print(seconds) // ここで self をキャプチャ
            seconds += 1 // ここで self をキャプチャ
        }
    }
    deinit { timer.invalidate() }
}
```

`self.` が強制されることで、この `self` は循環参照によるメモリリークを引き起こさないかと、一歩立ち止まって考えることができるわけです。

## メモリリークしないケース: `@nonescaping`

`@escaping` **でない** （ `@nonescaping` な）引数にクロージャ式を渡す場合、循環参照によるメモリリークを気にする必要はありません。

たとえば、次のコードでは `forEach` に渡すクロージャ式の中で `Kuku` （九九）クラスの `dan` （段）プロパティにアクセスしていますが、 `self.dan` のように `self.` を書く必要はありません。

```swift
// self. が不要な例

// 与えられた九九の段（ dan ）を表示するクラス
final class Kuku {
    let dan: Int
    init(dan: Int) { self.dan = dan }
    func show() {
        (1 ... 9).forEach { print(dan * $0) } // self は不要
    }
}
```

なぜなら、 `forEach` の引数は `@escaping` **でない** からです。関数やメソッドの `@escaping` でない引数にクロージャが渡された場合、その関数やメソッドは渡されたクロージャをプロパティ等にとっておくことはできません。渡されたクロージャへの参照は、その関数やメソッドの実行が終わるとただちに破棄されます。

そのため、たとえクロージャ式が `self` をキャプチャしていたとしても、関数やメソッドを抜けたらそのクロージャ式は即時解放されるため、クロージャ式から `self` への参照も破棄されます。循環参照が残ってメモリリークすることはありません。

そのような理由から、これまでも `@escaping` **でない** 引数にクロージャ式を渡す場合は、そのクロージャ式中での `self.` を省略することができました。

## メモリリークしないケース: 値型

`@escaping` なクロージャ式を書く場合でも、循環参照によるメモリリークの恐れがないことがあります。それは `self` が値型の場合です。

値型は参照型と違ってそもそも参照することができないので、循環参照が起こるおそれはありません。しかしそれにも関わらず、たとえ `self` が値型の場合でも、これまでは `@escaping` なら `self.` を書くことが強制されていました。

```swift
// Swift 5.2
struct ContentView: View {
    @State var isEnabled: Bool = false
    
    var body: some View {
        Button("OK") {
            self.isEnabled = true // self が必要
        }
    }
}
```

Swift 5.3 ではこの制約が取り払われ、 `self` が値型の場合には `self.` を書く必要がなくなりました。

```swift
// Swift 5.3
struct ContentView: View {
    @State var isEnabled: Bool = false
    
    var body: some View {
        Button("OK") {
            isEnabled = true // self は不要
        }
    }
}
```

**SwiftUI では通常 `View` を `struct` （値型）として実装します。そのため、クロージャ式の中の `self.` はほとんどのケースで省略できるようになります。**

# `@ViewBuilder` での `if let` や `swifch` の利用

```swift
// BEFORE (Swift 5.2)
VStack {
    Text("Result:")
    if value != nil {
        Text("\(value!)")
    }
}
```

```swift
// AFTER (Swift 5.3)
VStack {
    Text("Result:")
    if let value = self.value {
        Text("\(value)")
    }
}
```

SwiftUI では `body` の中に宣言的に View を記述することができます。これは `body` に `@ViewBuilder` が付与されているからです。 `@ViewBuilder` は Function Builder という言語仕様を用いて実装されています。

Swift 5.2 までの Function Builder では `if let` や `switch` を扱うことができませんでした。 Swift 5.3 ではこれが可能になります。

## Function Builder とは

Function Builder は宣言的に値を組み立てるための仕組みです。 `@ViewBuilder` の他にも、 Function Builder を使えば様々なものを宣言的に記述できるようになります。

たとえば、↓は `Dictionary` を宣言的に組み立てる例です。通常の Dictionary リテラルでは `if` で分岐することができませんが、この例では `isFoo` が `true` の場合だけ `"c": 5` というエントリーが含まれるように分岐しています。

```swift
// Dictionary のイニシャライザに Function Builder を用いた例
let dictionary: [String: Int] =  .init {
    [
        "a": 2,
        "b": 3,
    ]
    if isFoo { ["c": 5] } // 分岐することが可能
}
```

こんなことができるのは、 `Dictionary` のイニシャライザに `@DictionaryBuilder` という Function Builder を付与しているからです。この `@DictionaryBuilder` は標準ライブラリに含まれるものではなく、僕が実装してライブラリにしたものです。誰でも Function Builder を使ってこの `@DictionaryBuilder` のようなものを作ることができます。

- [DictionaryBuilder - GitHub / koher](https://github.com/koher/dictionary-builder)

実はこの Function Builder はまだ（ 2020 年 9 月 18 日時点では）正式には Swift の言語仕様として採用されていません。現時点で最新の Proposal でまさに今議論が行われ、 Swift Core Team が結論を下そうとしているところです。

- [SE-0289: Function builders](https://github.com/apple/swift-evolution/blob/master/proposals/0289-function-builders.md)

最終的に承認されるのは間違いないでしょうが、正式に組み込まれるのは Swift 5.4 か Swift 6 になるでしょう。

まだ承認されていないにも関わらず、 Function Builder は Swift 5.1 から使うことができます。これは、 SwiftUI のために非公式機能として組み込まれたからです。 [Function Builder 自体は 2019 年 1 月から議論されており](https://forums.swift.org/t/function-builders/25167)、昨年 SwiftUI のリリースと同時に Swift に組み込まれました。

実験的に組み込まれたため Function Builder として提案されている一部の機能しか提供されておらず、これまでは通常の（ `if let` でない） `if` 文しか使うことができませんでした。 Swift 5.3 では、それに加えて `if let` と `switch` が使えるようになりました。正式に Function Builder が実装されれば、最終的には `for` 文なども利用できるようになる予定です。そうすれば、 SwiftUI で `ForEach` を `for` 文で書けるようになるかもしれません。

## `@ViewBuilder` で `if let` と `switch` を使う

せっかく Swift には `Optional` があるのに、これまでは `@ViewBuilder` の中で `if let` が使えず、次のように `!` で Forced Unwrapping しないといけませんでした。

```swift
// Swift 5.2
struct ContentView: View {
    @State var value: Int?

    var body: some View {
        VStack {
            Text("Result:")
            if value != nil { // if let にできない
                Text("\(value!)") // ! が必要
            }
        }
        .onAppear {
            fetchValue(completion: { self.value = $0 })
        }
    }
}
```

Swift 5.3 では `@ViewBuilder` で `if let` が使えるので次のように書けます。

```swift
// Swift 5.3
struct ContentView: View {
    @State var value: Int?

    var body: some View {
        VStack {
            Text("Result:")
            if let value = self.value { // if let
                Text("\(value)") // ! は不要
            }
        }
        .onAppear {
            fetchValue(completion: { value = $0 })
        }
    }
}
```

また、 `switch` を使うこともできるようになりました。

↑の例では `fetchValue` は必ず成功しますが、 `fetchValue` のような非同期 API はエラーを返すことも多いでしょう。 `fetchValue` の `completion` ハンドラーが `Int` の代わりに `Result<Int, Error>` を受け取るケースでは、 `switch` を使って次のように分岐できます。

```swift
// Swift 5.3
struct ContentView: View {
    @State private var value: Result<Int, Error>?
    
    var body: some View {
        VStack {
            Text("Result:")
            if let value = self.value {
                switch value { // switch も使える
                case .success(let value):
                    Text("\(value)")
                case .failure(_):
                    Text("Error")
                }
            }
        }
        .onAppear {
            fetchValue(completion: { value = $0 })
        }
    }
}
```

`Optional` と `if let` はもちろん、 Swift では `enum` を使うことが多く、 `switch` の利用機会も少なくありません。 **`@ViewBuilder` で `if let` と `switch` が使えないことは SwiftUI のコードを書く上で大きなストレスでした。 Swift 5.3 でそれらが解消されたことで、快適な SwiftUI ライフを送れるようになるでしょう。**

## まとめ

Swift 5.3 では、 SwiftUI のコードを書く際に↓が実現されます。

- クロージャ式の中のほとんどの `self.` を省略できる
- `@ViewBuilder` で `if let` や `switch` が使える

Swift 5.3 で快適な SwiftUI ライフを楽しんで下さい！