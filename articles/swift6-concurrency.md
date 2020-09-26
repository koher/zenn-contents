---
title: "Swift 6で来たる並行処理の大型アップデート近況"
emoji: "🌀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["swift"]
published: true
---

**最近、 [Swift リポジトリ](https://github.com/apple/swift)に並行処理関係の Pull Request (PR) が続々とマージされています。** たとえば、次のような PR があります。

- [Add `async` to the Swift type system. #33147](https://github.com/apple/swift/pull/33147)
- [Add @asyncHandler attribute. #33476](https://github.com/apple/swift/pull/33476)
- [Import "did" delegate methods as @asyncHandler. #34065](https://github.com/apple/swift/pull/34065)
- [Import Objective-C methods with completion handlers as async #33674](https://github.com/apple/swift/pull/33674)
- [Basic support for actor classes and actor isolation #33906](https://github.com/apple/swift/pull/33906)

Swift の並行処理（ Concurrency ）関連の機能については、 2020 年 1 月に発表された "[On the road to Swift 6](https://forums.swift.org/t/on-the-road-to-swift-6/32862)" という公式アナウンスの中で特に重要な分野の一つとして挙げられていました。

> - Provide excellent solutions for major language features such as memory ownership and **concurrency**

前述の PR はそれに沿った動きだと考えられます。

さらに、 Swift Core Team の一人である John McCall さんが 2020 年 9 月 17 日に [Swift Forums で並行処理のデザイン案が数週間内に提供されると述べています](https://forums.swift.org/t/refine-dispatchqueue-main-to-return-a-os-dispatch-queue-main/40345/13) 。

> The concurrency design (coming in a few weeks, I promise!) will likely demote the importance of using this API directly

ちなみに上記の文中の "this API" とは `DispatchQueue.main` のことで、これを直接利用する重要性は下がると述べています。 `DispatchQueue.main` は Swift プログラマにとって非常に馴染み深い API だと思いますが、これを使う機会がほとんどなくなる（？）ほどの劇的な変化がありそうだと推測できます。

Swift の並行処理については現状では正式な手順でプロポーザルが作られていないものの、上記の経緯から考えてこれから急速に議論が進み、 Swift の次期メジャーバージョンである **Swift 6 （おそらく 2021 年リリース）には並行処理に関する大型のアップデートが含まれる** ことになりそうです。

昨日開催された[わいわいswiftc #22](https://iosdiscord.connpass.com/event/188624/) にて上記が話題になったので、その内容を本記事にまとめてみました。

# Swift の並行処理

Swift の並行処理については、 2017 年 8 月に（ Swift の生みの親であり Core Team のメンバーでもある） Chris Lattner によってマニフェストが示されました。

- [Swift Concurrency Manifesto](https://gist.github.com/lattner/31ed37682ef1576b16bca1432ea9f782)

このマニフェストは五つのパートからなるのですが、 Part 1 が `async/await` について、 Part 2 以降が `actor` についてのものとなっています。

`async/await` についてはプロポーザルのドラフトとして↓に切り出されています。

- [Async/Await for Swift](https://gist.github.com/lattner/429b9070918248274f25b714dcfc7619)

これらは Swift コミュニティに大きな衝撃を与え、半ば規定路線として扱われていますが、 3 年経った 2020 年 9 月 26 日現在でも正式なマニフェストやプロポーザルは作られていません。その証拠に上記のドキュメントはただの Gist で、 Swift および [Swift Evolution リポジトリ](https://github.com/apple/swift-evolution)に取り入れられていません。

しかし前述したように、このところ並行処理関連の動きが加速しており、今後正式に議論が進められていくものと思われます。

# Swift の `async/await`

`async/await` と言えば JavaScript / TypeScript や C# が有名です。一見すると、それらの `async/await` と Swift の `async/await` は同じもののように見えます。

```js
// JavaScript
async function foo() {
    const a = await bar();
    const b = await baz();
    return a + b;
}
```

```swift
// Swift
func foo() async -> Int {
    let a = await bar()
    let b = await baz()
    return a + b
}
```

`async` の位置が違う他はそっくりです。しかし、 Swift の `async/await` はそれらの言語の `async/await` と少し異なります。

たとえば、 JavaScript / TypeScript では `async` はその関数の中で `await` を使えるという意味ですし、 `await` は `Promise` を剥がす（値が得られるまで非同期的に待つ）ためのものです。そして、 `async` 関数の戻り値は `Promise` です。

しかし、 Swift の `async/await` には `Promise` に相当するものは登場しません。 `async` はその関数が非同期であることを示し、 `await` は `async` な関数をコールするときに必要なマークに過ぎません。 `async` 関数の戻り値の型も `Int` のような素の型です。

Swift の `async` と `await` はそれぞれ `throws` と `try` にとても良く似ています。 `throws` が付与された関数を呼び出すときには `try` を付けることが求められます。

```swift
func foo() throws -> Int { ... }

let a: Int = try foo() // try がないとコンパイルエラー
```

同様に、 `async` が付与された関数を呼び出すときには `await` を付けることが求められます。

```swift
func foo() async -> Int { ... }

let a: Int = await foo() // await がないとコンパイルエラー
```

また、 `throws` 関数を呼び出す場合（つまり `try` を書く場合）、（ `catch` しない限りは） `throws` 関数の中でないといけません。

```swift
func main() throws { // throws がないとコンパイルエラー
    print(try foo())
}
```

同様に、 `async` 関数を呼び出す場合（ `await` を書く場合）、 `async` 関数の中でないとコンパイルエラーになります。

```swift
func main() async { // async がないとコンパイルエラー
    print(await foo())
}
```

このように、 Swift では `throws` と `async` が、 `try` と `await` が対になるように設計されており、 `Promise` は介在しません。 Swift の `async/await` は JavaScript 等の `async/await` よりも Kotlin の `suspend` に近く、機能的にも Kotlin と同じくコルーチンをサポートするためのものです。

Swift の `async/await` そのものや `throws/try` との関係については、↓の記事および try! Swift Tokyo 2016 の発表で説明しているのでそちらを御覧下さい。

- [Proposalには載っていないSwift 5のasync/awaitが素晴らしいと思う理論的背景](https://qiita.com/koher/items/29357b5e00aec1962601)
- [Three Stories about Error Handling in Swift | try! Swift Tokyo 2016 （英語・動画）](https://github.com/koher/three-stories-about-error-handling-in-swift)

![](https://storage.googleapis.com/zenn-user-upload/heohk6fqd0m4cd59eyqjpbsce4mm)

# `@asyncHandler`

`async` 関数を使おうと思っても、 `async` 関数は `async` 関数の中でしか呼べません。 `async` 関数を呼び出すエントリーポイントはどうすれば良いでしょうか。

前述のプロポーザルでは、非同期エントリーポイントとして標準ライブラリに `beginAsync` 関数を追加することが提案されていました。

`beginAsync` の利用例は次のようになります。これは、ボタンが押されたときに非同期でデータをダウンロードするコードです。

```swift
func onButtonPressed(_ sender: UIButton) {
    beginAsync {
        let data: Data = await download(from: url)
        ...
    }
}
```

`beginAsync` は渡されたクロージャを非同期的に実行します（正確には `await` 以降を suspend します）。そのため、 `onButtonPressed` は `download` の完了を待たずに即座に終了します。これによって、 `download` を UI スレッドがブロックされることなく実行できます。

`beginAsync` に渡すクロージャの中で `async` 関数である `download` を呼び出せるのは、 `beginAsync` のシグネチャが次のようになっているからです。

```swift
func beginAsync(_ body: () async throws -> Void) rethrows -> Void
```

`body` に `async` が付与されているのがポイントです。そのため、 `body` として渡されるクロージャ式の中で `async` 関数を呼び出せるわけです。

しかし、↓の PR を見る限り、これとは異なったアプローチが採用されることになりそうです。

- [Add @asyncHandler attribute. #33476](https://github.com/apple/swift/pull/33476)

この PR は、 Swift に `@asyncHandler` という Attribute を追加するものです。

`@asyncHandler` は `beginAsync` と同じ役割を果たします。 `@asyncHandler` を使って `onButtonPressed` を実装すると次のようになります。

```swift
@asyncHandler
func onButtonPressed(_ sender: UIButton) {
    let data: Data = await download(from: url)
    ...
}
```

これは、関数全体を `beginAsync` で包んだ場合と等価です。

[わいわいswiftc #22](https://iosdiscord.connpass.com/event/188624/) では、 `beginAsync` をやめて `@asyncHandler` にした理由として、次のようなものが推測されていました。

- ネストを減らせる
- 専用の Attribute を用意することでより細かくコンパイルエラーの原因を診断することが可能となり、適切なエラーメッセージを提示できる

僕が個人的に気に入っているのは、 `@asyncHandler` を付与した関数は `throws` が禁止されている点です。これは、 `beginAsync` から `throws` と `rethrows` を取り除き、↓のように変更したことに相当します。

```swift
func beginAsync(_ body: () async -> Void) -> Void
```

僕は以前からそうするべきだと主張していた（↓）のですが、その通りの形にまとまりそうでよかったです。

- [SwiftのbeginAsyncにthrowsが要らない理由](https://qiita.com/koher/items/4e38dca62932022fbbb3)

`@asyncHandler` で `throws` が禁止されていることは [PR に含まれるテストコード](https://github.com/DougGregor/swift/blob/8cd892aba8edaf1c4155430186e4843926e14f57/test/attr/asynchandler.swift#L22-L24)からわかります。

```swift
@asyncHandler
func asyncHandlerBad3() throws { }
// expected-error@-1{{'@asyncHandler' function cannot throw}}{{25-32=}}
```

その他にも、テストコードからは `@asyncHandler` の戻り値は `Void` でないといけない、 `mutating func` には `@asyncHandler` を付けられないなどの（ `beginAsync` と比較してみると自明な）ルールを読み取ることができておもしろいです。

# デリゲートと `@asyncHandler`

- [Import "did" delegate methods as @asyncHandler. #34065](https://github.com/apple/swift/pull/34065)

この PR はちょっと変わっていて、 `did` を単語として含む Obj-C プロトコルのメソッドのうち、 `@asyncHandler` の条件（戻り値が `Void` など）を満たすものについて、自動的に `@asyncHandler` を付与しようというものです。

テストコードでは[次のようなプロトコルが挙げられています](https://github.com/DougGregor/swift/blob/a000375881ce37668c136fdaed71c997b194cebe/test/Inputs/clang-importer-sdk/usr/include/ObjCConcurrency.h#L16-L22)。

```objc
// Objective-C
@protocol RefrigeratorDelegate<NSObject>
- (void)someoneDidOpenRefrigerator:(id)fridge;
- (void)refrigerator:(id)fridge didGetFilledWithItems:(NSArray *)items;
- (void)refrigerator:(id)fridge didGetFilledWithIntegers:(NSInteger *)items count:(NSInteger)count;
- (void)refrigerator:(id)fridge willAddItem:(id)item;
- (BOOL)refrigerator:(id)fridge didRemoveItem:(id)item;
@end
```

[テストコード](https://github.com/DougGregor/swift/blob/a000375881ce37668c136fdaed71c997b194cebe/test/IDE/print_clang_objc_async.swift#L23-L27)によると、これらのデリゲートメソッドの最初の二つには `@asyncHandler` が付与され、後の三つには付与されないようです。

```swift
// CHECK-NEXT: @asyncHandler func someoneDidOpenRefrigerator(_ fridge: Any)
// CHECK-NEXT: @asyncHandler func refrigerator(_ fridge: Any, didGetFilledWithItems items: [Any])
// CHECK-NEXT: {{^}}  func refrigerator(_ fridge: Any, didGetFilledWithIntegers items: UnsafeMutablePointer<Int>, count: Int)
// CHECK-NEXT: {{^}}  func refrigerator(_ fridge: Any, willAddItem item: Any)
// CHECK-NEXT: {{^}}  func refrigerator(_ fridge: Any, didRemoveItem item: Any) -> Bool
```

`refrigerator(_:didGetFilledWithIntegers:count:)` については（おそらく） `items` が `inout` 引数相当なこと、 `refrigerator(_:willAddItem:)` は `did` を含まないこと、 `refrigerator(_:didRemoveItem:)` は戻り値の型が `Void` でないことが理由だと思われます。

この PR の意図するところは、 `did` のデリゲートメソッドの中で非同期処理を発火することが多いので、 `@asyncHandler` が付与されていると `async` 関数が使えて便利でしょ、ということだと思われます。

しかし、命名を元にこのような処理を施して本当に良いのか、 `did` だけで `will` には必要ないのかなど、わいわいswiftc #22 では様々な議論が交わされました。個人的に興味深かった意見をまとめたものが↓です。

- `will` については非同期処理を待たずに後続（の `did` などの）処理が行われるので適さないのではないか。
- でも `did` でも[繰り返し呼ばれるデリゲートメソッド](https://developer.apple.com/documentation/foundation/urlsessiondatadelegate/1411528-urlsession)もあるけど良いのか。
- `@asyncHandler` が付与された関数は通常の関数のスーパータイプになるので、 `async` 関数を呼ばないといけないわけではない。なので、雑に付与されても良いのではないか。

# completion ハンドラーを持つ Obj-C メソッドを `async` に

- [Import Objective-C methods with completion handlers as async #33674](https://github.com/apple/swift/pull/33674)

この PR は、 completion ハンドラーを持つ Obj-C メソッドを `async` メソッドに変換するというものです。たとえば、 `UIView` の `animate` メソッドは現在 completion ハンドラーを使って次のように使います。

```swift
// BEFORE
func onButtonPressed(_ sender: UIButton) {
    UIView.animate(withDuration: 0.5) {
        image.alpha = 1.0
    } completion: { isFinished in
        ... // 完了時の処理
    }
}
```

この `animate` メソッドが `async` に変換されると、それを利用するコードは次のようになります。

```swift
// AFTER
@asyncHandler
func onButtonPressed(_ sender: UIButton) {
    let isFinsihed = await UIView.animate(withDuration: 0.5) {
        image.alpha = 1.0
    } 
    ... // 完了時の処理
}
```

シグネチャで言うと次のような変化になります。

```swift
// BEFORE
class func animate(withDuration duration: TimeInterval, 
        animations: @escaping () -> Void, 
        completion: ((Bool) -> Void)? = nil)
```

```swift
// AFTER
class func animate(withDuration duration: TimeInterval, 
        animations: @escaping () -> Void) async -> Bool
```

[テストコード](https://github.com/DougGregor/swift/blob/6ad2757bef212510785eef8a9755d09140b54978/test/IDE/print_clang_objc_async.swift)には色々なパターンが挙げられていておもしろいです。

`UIView.animate` はエラーを発生させない非同期処理でしたが、大抵の非同期処理はエラーを発生させる可能性があります。その場合、たとえば↓のように結果（ `String` ）とエラーをそれぞれ `Optional` で受け取る API が一般的です。

```swift
// BEFORE
func doSomethingDangerous(_ operation: String,
    completionHandler handler: ((String?, Error?) -> Void)? = nil)
```

このような場合は、 `async throws` が付与された関数に変換してくれるようです。

```swift
// AFTER
func doSomethingDangerous(_ operation: String) async throws -> String?
```

この変換前後で、この関数の利用側のコードは次のように変化します。

```swift
// BEFORE
doSomethingDangerous("ABC") { value, error in
    if let error = error {
        // エラーハンドリング
    }
    use(value!)
}
```

```swift
// AFTER
do {
    let value = try await doSomethingDangerous("ABC")
    use(value)
} catch {
    // エラーハンドリング
}
```

`if` や `for` などの制御構文との組み合わせも簡単になり、便利になりそうです！

わいわいswiftc #22 では他にも次のようなことが話されていました。

- `Result` が導入されたときに、このようなエラーを伴う非同期 API のハンドラーは `(Value?, Error?)` ではなく `Result<Value, Error>` を受け取る形に変換されなかったのか？（そうすれば Forced Unwrapping が必要ない。）
    - `Result` の導入時にはすでに `async/await` が論じられていたので、それを見越して（ `Result` が不要な） `async/await` を待ったのではないか。
- キャンセラを返すような非同期 API では `async` にできない。
    - `Result` の[プロポーザルで論じられた](https://github.com/apple/swift-evolution/blob/master/proposals/0235-add-result.md#asynchronous-apis) `URLSession.dataTask` （ [API リファレンス](https://developer.apple.com/documentation/foundation/urlsession/1407613-datatask)）は結局 `(Data?, URLResponse?, Error?)` を受け取るままだが、この API は（キャンセラに相当する） `URLSessionDataTask` を返すので結局 `async` にできない。それなら `Result<(URLResponse, Data), Error>` を受け取る形に修正されても良いのでは？

# `actor`

[マニフェスト](https://gist.github.com/lattner/31ed37682ef1576b16bca1432ea9f782)の Part 1 に当たる `async/await` だけでなく、 Part 2 以降の `actor` に関する PR もマージされています。

- [Basic support for actor classes and actor isolation #33906](https://github.com/apple/swift/pull/33906)

このことから、 Swift 6 での並行処理は `async/await` に留まらず、 `actor` まで含めた広範囲のものになると予想されます。

この `actor` は[アクターモデル](https://ja.wikipedia.org/wiki/%E3%82%A2%E3%82%AF%E3%82%BF%E3%83%BC%E3%83%A2%E3%83%87%E3%83%AB)の `actor` で、マニフェストの中でも次のように述べられています。

> We propose the introduction of a first-class actor model

アクターモデルについては僕も勉強中なので詳しく説明できませんが、アクターモデルとは多数のアクター同士が非同期のメッセージパッシングによって通信し処理を実行するモデルのことです。

アクターモデルを採用した最も有名なプログラミング言語は [Erlang](https://www.erlang.org/) ではないかと思います。ここでは、実際に Erlang を使っているエンジニアから聞いた、 Erlang を学ぶのにオススメの書籍を二冊挙げておきます。前者は Erlang の設計者自身が書いたもので、後者は数年前に発売されて人気を博したものです。

- [プログラミングErlang](https://www.amazon.co.jp/dp/B079YZC22H)
- [すごいErlangゆかいに学ぼう！](https://www.amazon.co.jp/dp/B00MLUGZIS)

:::message
本節の以下の内容はあやふやな知識によって書かれています。
:::

間違いがあるかもしれませんが、僕は Erlang のアクターは次のような特徴を持っていると理解しています。

- Erlang プロセスという軽量なプロセス（ OS のプロセスとは別物）がアクターとして振る舞う
- Erlang プロセスはメッセージボックスを持っており、 Erlang プロセス同士がメッセージを送り合うことで通信する
- Erlang プロセスは自身のメッセージボックスから順番にメッセージを取り出し処理を実行する（キューに似ている？）
- Erlang プロセスは共有メモリを持たない（ので共有メモリに由来する問題を引き起こさない）
- Erlang プロセスがクラッシュしても OS のプロセスがクラッシュするわけではない

僕は特に最後の点が興味深く、 Erlang ではエラーハンドリングとしてプロセスをクラッシュさせることが普通に行われると聞いたことがあります。

これに関連することがマニフェストの "[Part 3: Reliability through fault isolation](https://gist.github.com/lattner/31ed37682ef1576b16bca1432ea9f782)" で述べられています。

これまで Swift は Forced Unwrapping の失敗や `Array` の index out of bounds のエラー、 `preconditoon` の失敗などの [Logic Failure](https://github.com/apple/swift/blob/master/docs/ErrorHandlingRationale.rst#logic-failures) をハンドリングすることができませんでした。 `actor` の導入によって、これらのクラッシュをプロセスのクラッシュではなく `actor` のクラッシュとして隔離し、 Logic Failure をハンドリングすることが可能となります。

僕もまだきちんとマニフェストを読めていないですし、 "a few weeks" で正式なドキュメントが出てきそうなのでそれを待ちたいと思いますが、 Swift への `actor` 導入が本格的に動きそうなので、そろそろちゃんと勉強したいと考えています。前述の Erlang 本も途中まで読んで積読になってるので、それを再開してみるのも良さそうです。

# その他の PR

軽く調べて見たところ、昨日わいわい swiftc #22 で話題になった以外にも次のような PR が見つかりました。広範囲の構文が実装されつつあることがわかり、期待が膨らみます。

## `async/await` 関連

- [Implement parsing and semantic analysis of await operator #33199](https://github.com/apple/swift/pull/33199)
- [Implement restrictions on calls to 'async' functions. #33399](https://github.com/apple/swift/pull/33399)
- [Add support for 'async' closures. #33408](https://github.com/apple/swift/pull/33408)
- [async autoclosures are only legal on async functions. #33446](https://github.com/apple/swift/pull/33446)
- [Treat 'await' as a contextual keyword. #33457](https://github.com/apple/swift/pull/33457)
- [Infer @asyncHandler from protocol requirements. #33488](https://github.com/apple/swift/pull/33488)
- [Fix nested await/try parsing and effects checking. #33807](https://github.com/apple/swift/pull/33807)
- [Enable support for @objc async methods #33837](https://github.com/apple/swift/pull/33837)
- [Allow overload 'async' with non-async and disambiguate uses. #33862](https://github.com/apple/swift/pull/33862)

## `actor` 関連

- [Actor-isolated members cannot satisfy protocol requirements #33982](https://github.com/apple/swift/pull/33982)
- [Introduce `@actorIndependent` attribute. #33998](https://github.com/apple/swift/pull/33998)
- [Handle partial application of actor-isolated methods. #34051](https://github.com/apple/swift/pull/34051)

# まとめ

最近、 Swift リポジトリに `async/await` や `actor` といった並行処理関連の PR が大量にマージされています。 Swift Core Team メンバーの発言からも、今後数週間以内に Swift の並行処理に関する正式なドキュメントが出てきそうで、今後急速に話が進むと思われます。 Concurrency Menifesto が出てから 3 年、待ち侘びていた Swift の並行処理関連機能が Swift 6 でついに導入されそうです。

`async/await` はともかく、 `actor` は多くの Swift プログラマにとって馴染みの薄い概念だと思います。来年に向けて勉強を進めることが求められるでしょう。しかし、このアップデートによって、 Swift で非同期処理や並行処理を扱うのが劇的に楽になると予想されます。それらを使える未来を考えるとわくわくしますね。 Swift 6 のリリースが待ち遠しいです！
