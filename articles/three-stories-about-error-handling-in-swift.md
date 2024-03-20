---
title: "Swiftのエラーハンドリングについての三つの話"
emoji: "🕊"
type: "tech"
topics: ["swift", "error"]
published: true
---

:::message
これは、2016年に開催された最初の[try! Swift](https://tryswift.jp/)で、僕が英語発表したときの原稿を和訳したものです（読みやすいように微修正はしています）。

try! Swiftは、コロナ禍による長い中断を経て、（フル開催としては）5年ぶりに明日から開催されます。僕は、今回は時間的余裕がなさそうだったのでProposalの提出を見送ったのですが、僕にとっては初登壇となった思い入れのあるカンファレンスなので、何かしたいと思い当時の発表内容を和訳して投稿することにしました。

![](/images/three-stories-about-error-handling-in-swift/try-swift-tokyo-2016.jpg)

この発表は英語で行われたことと、25分の発表時間にしては内容を詰め込みすぎたことなどから、がんばって準備したわりにあまり伝わらなかったんじゃないかと感じていました。8年前の発表なので今となっては古い話もありますが、2021年に採用されたのと同じ形の `async/await` （Swift独自の形なので当時存在していなかったもの）を2016年時点で提案していたりして、おもしろいんじゃないかと思います。また、当時は `@discardableResult` がなく `@warn_unused_result` を使っていたなど、当時を知る歴史的資料としてもおもしろいと思います。

また、この発表はせっかく英語で話すんだし、Appleの言語についてのカンファレンスだからと、ジョブズの有名なスピーチ[^8]のオマージュとして原稿を作りました。全体の流れから細かい表現までかなりジョブズのスピーチを意識しています。おそらく、細かい部分はほとんどの人に伝わらなかったので、その点についても補足します。
:::

世界で最も優れたSwiftカンファレンスの1つであるtry! Swiftに参加できて光栄です。実を言うと、私はSwiftのカンファレンスに参加したことはなく、これが初めてのSwiftに関するプレゼンテーションです。今日は、エラーハンドリングに関する3つの話をしたいと思います。それだけです。大したことではありません。ただの3つの話です。

:::message
冒頭部分を英語で見比べると、次のようにほとんどジョブズのスピーチのままです。

ジョブズのスピーチ

> I am honored to be with you today at your commencement from one of the finest universities in the world. I never graduated from college. Truth be told, this is the closest I’ve ever gotten to a college graduation. Today I want to tell you three stories from my life. That’s it. No big deal. Just three stories.

僕の発表

> I am honored to be with you today at one of the finest Swift conferences in the world. Truth be told, I've never attended a Swift conference and this is my first presentation about Swift. Today I want to tell you three stories about error handling. That's it. No big deal. Just three stories.
:::

## 1. Optionalとの出会い (Meeting the Optionals)

:::message
ジョブズのスピーチ

> Connecting the Dots

僕の発表

> Meeting the Optionals
:::

最初の話は `Optional` との出会いについてです。

私は、 `Optional` はSwiftの最高の機能の一つだと考えています。では、なぜ私はそう考えるようになったのでしょうか。それは、Swiftが生まれる前に始まりました。

:::message
ジョブズのスピーチ

> So why did I drop out? It started before I was born.

僕の発表

> So why did I get it? It started before Swift was born.
:::

### Cにおけるエラーハンドリング

私は子供の頃にBASICで遊んだことはありましたが、実質的に最初に使ったプログラミング言語はCでした。Cでのエラーハンドリングは、このようなものでした。

```c
// [ C ]
int *numbers = (int *)malloc(sizeof(int) * 42);
if (numbers == NULL) {
    // ここでエラーハンドリング
}
```

これは簡単に忘れてしまいがちです。

Cコンパイラは、エラーハンドリングを忘れても警告やエラーを出しません。これは安全ではありません。

### Javaにおけるエラーハンドリング

その後、私はJavaの***検査例外***を学びました。これは、プログラマにエラーハンドリングを強制するものです。

例えば、文字列から整数をパースする関数を考えてみましょう。この関数は、文字列が正しくパースできない場合に `FormatException` を `throw` します。

```java
// [ Java ]
static int toInt(String string) throws FormatException {
    ...
}
```

```java
// [ Java ]
toInt("42"); // 成功
toInt("Swift"); // 失敗
```

エラーハンドリングをしないと、コンパイルエラーになります。

```java
// [ Java ]
String string = ...;
int number = toInt(string); // コンパイルエラー
```

`try` と `catch` を使う必要があります。

```java
// [ Java ]
String string = ...;
try {
    int number = toInt(string);
...
} catch (FormatException e) {
    // エラーハンドリング
    ...
}
```

しかし、時にはエラーを無視したいこともあります。たとえば、テキストフィールドにユーザーが数字だけを入力できるようにしている場合などです。そのような場合でも、エラーを無視するために、意味のないエラーハンドリングを書かなければなりません。

```java
// [ Java ]
String string = ...;
try {
    int number = toInt(string);
...
} catch (FormatException e) {
    // エラーハンドリング
    throw new Error("ここには到達しない。");
}
```

私は、同僚とこの問題について議論し、次のような結論に達しました。エラーを無視するための明示的だが簡単な方法が必要だと。

メソッド呼び出しの後に`!`をつけるのは良い候補でした。1つのキーを打つだけで済み、明示的で、危険そうに見えます。

```java
// [ Java ]
String string = ...;
int number = toInt(string)!; // 例外を無視する
// この`!`が私たちが求めていたものでした。
```

### エラーハンドリングのための `Optional`

数年後、私はSwiftの `Optional` に出会いました。

Swiftは主に `NullPointerException` を排除するために `Optional` を提供しました。しかし、 `Optional` はエラーハンドリングにも使われていました。

`toInt` は、`Optional` を使ってこのように書かれます。

```swift
// [ Swift ]
func toInt(string: String) -> Int? {
    ...
}
```

エラーハンドリングをしないと、コンパイルエラーになります。

```swift
// [ Swift ]
let string: String = ...
let number: Int = toInt(string) // コンパイルエラー
```

***Optional Binding***を使ってエラーをハンドリングすることができます。

```swift
// [ Swift ]
let string: String = ...
if let number = toInt(string) {
    ...
} else {
    // エラーハンドリング
    ...
}
```

エラーを無視するにはどうすればよいでしょうか。

***Forced Unwrapping*** を知ったとき、私はとても驚きました。それは、まさに私たちが求めていたものだったからです。

```swift
// [ Swift ]
let string: String = ...
let number: Int = toInt(string)! // エラーを無視する
```

*検査例外*とは異なり、 `Optional` は副作用のある関数、特に戻り値のない関数ではうまく機能しません。しかし、 `@warn_unused_result` 属性がその解決策になると思います。

```swift
// [ Swift ]
@warn_unused_result
func updateBar(bar: Bar) -> ()? {
    ...
}
```

```swift
// [ Swift ]
foo.updateBar(bar) // 戻り値を使わないと警告
```

:::message
現在のSwift（5.10）では、普通に関数を宣言すると、戻り値を利用しない場合にコンパイラが警告を発します。しかし、当時のSwift 2では逆に、戻り値を利用しないと警告を発したい関数を宣言する場合に、明示的に `@warn_unused_result` を付与する必要がありました。[SE-0047](https://github.com/apple/swift-evolution/blob/main/proposals/0047-nonvoid-warn.md)によって `@discardableResult` が導入され、Swift 3から現在の挙動となりました。

また、現在は空のタプルではなく `Void` と書くのが一般的ですが、当時は `()` もよく使われていました。
:::

エラーのハンドリングと無視は、このように行うことができます。

```swift
// [ Swift ]
if let _ = foo.updateBar(bar) {
    ...
} else {
    // エラーハンドリング
    ...
}
```

```swift
// [ Swift ]
_ = foo.updateBar(bar) // エラーを無視する
```

もし`@error_unused_result`のような属性があれば、（エラーハンドリングするか、明示的に無視するかを強制できて）さらに良くなるでしょう。

そして、 `Optional` はエラーをハンドリングするためのより柔軟な方法を提供します。*例外*は `throw` された直後にハンドリングしなければなりませんが、 `Optional` は遅延してハンドリングすることができます。

`Optional` を変数に代入したり、関数に渡したり、プロパティに格納したりすることができます。

```swift
// [ Swift ]
let string: String = ...
let number: Int? = toInt(string)
...
// エラーは遅延してハンドリングできる
if let number = number {
    ...
} else {
    // エラーハンドリング
    ...
}
```

:::message alert
現在、僕はこのような `Optional` の使い方は望ましくないと考えています。 `Optional` はエラー情報を持たないため、エラー発生箇所から離れてハンドリングすると、原因を特定するのが難しくなるためです。 `Optional` によるエラーハンドリングは、結果を受け取ってすぐに行うべきであることは、Swiftにおけるエラーハンドリングの指針を示した[Error Handling Rationale and Proposal](https://github.com/apple/swift/blob/main/docs/ErrorHandlingRationale.md)でも述べられています。
:::

### `Optional` を使う難しさ

すべてがロマンチックだったわけではありません。私のコードはすぐに `Optional` だらけになりました。

:::message
これはジョブズのスピーチ中の"It wasn’t all romantic."を踏襲した表現です。
:::

`Optional` では、数値を2乗するのでさえ簡単ではありません。

```swift
// [ Swift ]
let a: Int? = ...
let square = a * a // コンパイルエラー
```

和を計算するのも同様です。

```swift
// [ Swift ]
let a: Int? = ...
let b: Int? = ...
let sum = a + b // コンパイルエラー
```

*Optional Binding*を使うと、それぞれ5行になります。ひどいですね。

```swift
// [ Swift ]
let a: Int? = ...
let square: Int?
if let a = a {
    square = a * a
} else {
    square = nil
}
```

```swift
// [ Swift ]
let a: Int? = ...
let b: Int? = ...
let sum: Int?
if let a = a, b = b {
    sum = a + b
} else {
    sum = nil
}
```

### `Optional` のための関数型言語的な操作

幸いなことに、Swiftはそのようなケースのための関数型言語的な方法を提供しています。

`square` には `map` が便利で、

```swift
// [ Swift ]
let a: Int? = ...
let square: Int? = a.map { $0 * $0 }
```

`sum` には、ネストした `Optional` をフラット化するために `flatMap` が使えます。

```swift
// [ Swift ]
let a: Int? = ...
let b: Int? = ...
let sum: Int? = a.flatMap { a in b.map { b in a + b } }
```

`Optional` 値が増えると、複雑になります。典型的なケースは、JSONからモデルをデコードする場合です。

:::message
JSONのデコードの話が度々出てきますが、当時は `Codable` がなかったため、JSONのデコードがよく話題に上がりました。
:::

SwiftyJSON[^1]のようなAPIがあるとします。

```swift
// [ Swift ]
let id: String? = json["id"].string
```

デコードのどのステップでも失敗する可能性があります。

```js
// [ JSON ]
// `json`が`Object`ではないかもしれない
[ "abc" ]
// `"id"`というキーがないかもしれない
{ "foo": "abc" }
// 値が`String`ではないかもしれない
{ "id": 42 }
```

そのため、すべての戻り値が `Optional` になります。これらの `Optional` 値を使って、この `Person` をどのように初期化すればよいでしょうか。

```swift
// [ Swift ]
struct Person {
    let id: String
    let firstName: String
    let lastName: String
    let age: Int
    let isAdmin: Bool
}

let id: String? = json["id"].string
let firstName: String? = json["firstName"].string
let lastName: String? = json["lastName"].string
let age: Int? = json["age"].int
let isAdmin: Bool? = json["isAdmin"].bool
```

`flatMap`を使うと、このひどいピラミッドができあがります。

```swift
// [ Swift ]
let person: Person? = id.flatMap { id in
    firstName.flatMap { firstName in
        lastName.flatMap { lastName in
            age.flatMap { age in
                isAdmin.flatMap { isAdmin in
                    Person(
                        id: id,
                        firstName: firstName,
                        lastName: lastName,
                        age: age,
                        isAdmin: isAdmin
                    )
                }
            }
        }
    }
}
```

*Optional Binding*の方がましに見えます。

```swift
// [ Swift ]
let person: Person?
if
    let id = id,
    let firstName = firstName,
    let lastName = lastName,
    let age = age,
    let isAdmin = isAdmin
{
    person = Person(
        id: id,
        firstName: firstName,
        lastName: lastName,
        age: age,
        isAdmin: isAdmin
    )
} else {
    person = nil
}
```

しかし、パラメータ名を何度も繰り返さなければなりません。

:::message
現在は*Optional Binding*の右辺を省略できるので少し楽になります。

```swift
if
    let id,
    let firstName,
    let lastName,
    let age,
    let isAdmin
{
    ...
}
```
:::

Haskellでよく使われるアプリカティブスタイルでは、もっとシンプルになります。

```swift
// [ Swift ]
let person: Person? = curry(Person.init)
    <^> id
    <*> firstName
    <*> lastName
    <*> age
    <*> isAdmin
```

アプリカティブスタイルは、サードパーティのライブラリ**thoughtbot/Runes**[^2]を使うことで、Swiftでも利用可能です。

:::message alert
現在僕は、Swiftにおいてこのようなスタイルでのコーディングを推奨していません。SwiftでのOptionalの利用については、その後、次の記事にまとめたので参考にしてください。

- [SwiftのOptionalのベストプラクティス](https://qiita.com/koher/items/8b6156c8263b9b23c43c)
:::

### `Optional`のためのシンタックシュガーと演算子

さらに、Swiftは次のようなのシンタックスシュガーと演算子を提供し、 `Optional` を使いやすくしています。

```swift
// [ Swift ]
//let foo: Optional<Foo> = ...
let foo: Foo? = ...

// let baz: Baz? = foo.flatMap { $0.bar }.flatMap { $0.baz }
let baz: Baz? = foo?.bar?.baz

// let quxOrNil: Qux? = ...
// let qux: Qux
// if let q = quxOrNil {
//     qux = q
// } else {
//     qux = Qux()
// }
let quxOrNil: Qux? = ...
let qux: Qux = quxOrNil ?? Qux()
```

### Swiftにおける `Optional`

- `Foo? == Optional<Foo>`
- *Forced Unwrapping*: `!`
- `map`, `flatMap`
- アプリカティブスタイル: `<^>`, `<*>`
- *Optional Chaining*: `foo?.bar?.baz`
- *Nil-Coalescing Operator*: `??`

一部の言語にはそれらのいくつかがありました。しかし、それらすべての組み合わせによって、Swiftは他の言語には実現できていない安全で実用的、理論的に洗練された言語になりました。私はそれに魅了されました。

*null参照*の発明者であるTony Hoareはこう言いました。

> 実装が非常に簡単だったので、null参照を入れたいという誘惑に抗えませんでした。

これはプログラミングのダークサイドだと思います。ダークサイドに落ちるのは簡単です。少しの安全性を犠牲にすれば、型の複雑さから解放されます。しかし、私はジェダイでいたいのです。それが進化につながると信じています。

`Optional` は進化でした。型安全でありながら実用的です。それが、私がSwiftの `Optional` が素晴らしいと考える理由です。

## 2. 成功か失敗か (Success or Failure)

:::message
ジョブズのスピーチ

> Love and Loss

僕の発表

> Success or Failure

andがorに変わってしまいましたが、名詞and/or名詞の形でそろえました。"Love and Loss"はLoveとLossの語感が似ているところも狙っているのではないかと思いますが、さすがに文脈を成り立たせながらそこまで再現することはできませんでした。
:::

私の二つ目の話は、成功か失敗かについてです。

### `Optional` の問題点

`Optional` は素晴らしかったのですが、エラーの原因を報告する方法がありませんでした。

それは主に二つの問題につながります。

1. デバッグが難しい
2. エラーの原因によって処理を分岐できない

たとえば、

```swift
// [ Swift ]
let a: Int? = toInt(aString)
let b: Int? = toInt(bString) 
let sum: Int? = a.flatMap { a in b.map { b in a + b } }
guard let sum = sum else {
    // `a`と`b`のどちらがパースに失敗したのか?
    // 入力された文字列は何だったのか?
    ...
}
```

このような単純な処理でさえ、 `a` と `b` のどちらのパースに失敗したのか、入力は何だったのかを知りたいところです。

もう一つはJSONの例です。JSONでキー `"isAdmin"` が省略されている場合に `false` としたい場合、 `Optional` ではどのようにすればよいでしょうか。

```swift
// [ Swift ]
let isAdmin: Bool
if let admin = json["isAdmin"].bool {
    // JSONが {"isAdmin": true} だった場合はここに入る
    isAdmin = admin
} else {
    // JSONが
    // 1. [true]
    // 2. {}
    // 3. {"isAdmin": 42}
    // のどれであってもここに入る
    isAdmin = ...
}
```

示したように、3通りの失敗の仕方があります。2番目のケースからのみ回復させたい（ `"isAdmin"` を `false` としてデコードしたい）のですが、他の二つのケースではエラーにしたいです。

1. `[ true ]`: error
2. `{}`: `false `
3. `{ "isAdmin": 42 }`: error

`nil` ではその違いを表現できません。

### `Optional` の代替案

私はそれらの問題に対して3つの解決策を見つけました。

1. タプル
2. Union型
3. Result型

それらはSwift Evolutionメーリングリストでも議論されています。

:::message
今では考えられないことですが、当時のSwift Evolutionはメーリングリストでした。
:::

### タプル

タプルを使うと、`toInt`は次のように書けます。

```swift
// [ Swift ]
func toInt(string: String) -> (Int?, FormatError?) {
    ...
}
```

`Int` 値に加えて `FormatError` を返します。Goのライブラリではこのスタイルを採用しているものがあります。

しかし、これでは結果が次の4通りになってしまいます。

- `(value, nil )`: 成功
- `(nil , error)`: 失敗 
- `(value, error)`: ???
- `(nil , nil )`: ???

最後の二つは不要です（型の上で起こらないものを含んでいて冗長です）。

### Union型

*Union型*はCeylon、TypeScript、*型ヒント*付きのPythonなどで提供されています。

*Union型*では、 `Int|String` は `Int` または `String` 型であることを意味します。そのため、 `Int|FormatError` で `Int` か `FormatError` を（戻り値として）直接返すことができます。

```swift
// [ Swift ]
func toInt(string: String) -> Int|FormatError {
    ...
}
```

```swift
// [ Swift ]
switch toInt(...) {
case let value as Int:
    ...
case let error as FormatError:  
    // エラーハンドリング
    ...
}
```

さらに興味深いのは、CeylonとPythonの `Optional` が*Union型*のシンタックスシュガーであることです。

```java
// [ Ceylon ]
Integer? a = 42;
Integer|Null a = 42;
```

```python
# [ Python ]
def foo() -> Optional[Foo]: ...
def foo() -> Union[Foo, None]: ...  
```

*Union型*は、それらの言語では `Optional` を一般化する自然な方法です。

しかし、Swiftの `Optional` は `enum` です。

```swift
// [ Swift ]
enum Optional<T> {
    case Some(T)
    case None
}
```

:::message
当時のSwiftでは `enum` の `case` 名はUpperCamelCaseでした。今はlowerCamelCaseが一般的です。

`Optional` の `case` も `.some` と `.none` にリネームされました。
:::

私は*Union型*で `Optional` を表すのはSwift的ではないと思いました。

### Result型

*Result型*はRustから来ました。

`Result` はこのように宣言できます。

```swift
// [ Swift ]
enum Result<T, E> {
    case Success(T)
    case Failure(E)
}
```

これはSwift的で、`Optional`の自然な拡張です。

```swift
// [ Swift ] 
enum Optional<T> {
    case Some(T)
    case None  
}
```

*Result型*を使えば、エラー情報を取得でき、

```swift
// [ Swift ]
let a: Result<Int, FormatError> = toInt(aString)
let b: Result<Int, FormatError> = toInt(bString)
let sum: Result<Int, FormatError> = a.flatMap { a in b.map { b in a + b } }
switch sum {
case let .Success(sum):
    ...
case let .Failure(error):
    // `error`から詳細なエラー情報を取得する
    ...
}
```

エラーの原因によって処理を分岐できます。

```swift
// [ Swift ]
let isAdmin: Bool
switch json["isAdmin"].bool {
case let .Success(admin):
    isAdmin = admin
case .Failure(.MissingKey):  
    // {} => false
    isAdmin = false
case .Failure(.TypeMismatch, .NotObject):
    // [ true ] => error 
    // { "isAdmin": 42 } => error
    ...
}
```

`Result` は `Optional` のように `map` や `flatMap` することもできます。

antitypical/Result[^3]は、そのような `Result` をSwiftに提供しています。

:::message
Swiftの標準ライブラリに `Result` が追加されたのは意外と最近で、[SE-0235](https://github.com/apple/swift-evolution/blob/main/proposals/0235-add-result.md)で提案され、Swift 5.0になってからです。
:::

`Result` に `Optional` のようなシンタックスシュガーがあれば便利でしょう。

*Unition型*は導入しないにしても、その `|` による表記は直感的で書きやすそうです。また、 `flatMap` のチェーンは*Optional Chaining*のように書けるべきです。

```swift
// [ Swift ]
let foo: Result<Foo, Error> = ...
let baz: Result<Foo, Error> = foo.flatMap { $0.bar }.flatMap { $0.baz }
```

```swift
// [ Swift ]
let foo: Foo|Error = ...
let baz: Baz|Error = foo?.bar?.baz
```

それらは `Result` をより強力にするでしょう。

### `Result` を使う難しさ

`Result` は良さそうでした。しかし、それらがうまく機能しないケースがすぐに見つかりました。これがその例です。

```swift
// [ Swift ]
let a: Result<Int, ErrorA> = ...
let b: Result<Int, ErrorB> = ...
let sum: Result<Int, ???> = a.flatMap { a in b.map { b in a + b } }  
```

2番目の型パラメータは何にすべきでしょうか。`ErrorA`と`ErrorB`の両方になり得ます。

1つの簡単な答えは、（ `ErrorA|ErrorB` の）*Union型*を使うことでした。しかし、*Union型*は（Swiftには適さないと）導入を見送りました。

```swift
// [ Swift ]
// ⛔️ Union型の導入を見送ったのでErrorA|ErrorBは使えない
let sum: Result<Int, ErrorA|ErrorB> = a.flatMap { a in b.map { b in a + b } }
```

次のアイデアは、ネストした `Result` でした。

```swift
// [ Swift ]
let sum: Result<Int, Result<ErrorA, ErrorB>> = a.flatMap { a in b.map { b in a + b } }
```

しかし、これはひどく見えます。

`|` 表記を使うと、少しマシになりました。

```swift
// [ Swift ]
let sum: Int|ErrorA|ErrorB = a.flatMap { a in b.map { b in a + b } }
```
しかし、 `Result` が増えるとまだひどい状態でした。ネストが深すぎて直感的ではありません。

```swift
// [ Swift ]
let id: String|ErrorA = ...
let firstName: String|ErrorB = ...
let lastName: String|ErrorC = ...
let age: Int|ErrorD = ...
let isAdmin: Bool| ErrorE = ...
let person: Person|(((ErrorA|ErrorB)|ErrorC)|ErrorD)|ErrorE
    = curry(Person.init) <^> id <*> firstName
        <*> lastName <*> age <*> isAdmin
```

このとき、エラーハンドリングはこのように行われます。

```swift
// [ Swift ]
switch person {
case let .Success(person):
...
case let .Failure(.Success(.Success(.Success(.Success(.Failure(errorA)))))):
...
case let .Failure(.Success(.Success(.Success(.Failure(errorB))))):
...
case let .Failure(.Success(.Success(.Failure(errorC)))):
...
case let .Failure(.Success(.Failure(errorD))):
...
case let .Failure(.Failure(errorD)):
...
}
```

あまりにも煩雑です。

### エラー型のないResult型

私はこの問題について長い間考えました。そして最終的に、 `Result` の2番目の型パラメータは実際には重要ではないと結論付けました。

もし `Result` が次のように宣言されていれば、エラーの型を失い、安全ではないように見えます。

```swift
// [ Swift ]
enum Result<T> {
    case Success(T)
    case Failure(ErrorType)
}
```

:::message
当時は `Error` プロトコルは `ErrorType` プロトコルでした。
:::

しかし、ほとんどの場合、すべての可能なエラーに対して処理を分岐する必要はありません。気にする必要があるのは、一つか二つの例外的なエラーだけです。

ネットワーク処理について考えてみましょう。それらはさまざまな方法で失敗します。タイムアウトになったときは処理をリトライしたいですが、ForbiddenやNot foundなどの場合はリトライしたくありません。

そこで、処理を成功・タイムアウト・その他のケースに分岐します。起こり得るすべてのエラーを列挙する必要はありません。

```swift
// [ Swift ]
downloadJson(url) { json: Result<Json> in
    switch json {
    case let .Success(json): // 成功
        ...
    case let .Failure(.Timeout): // タイムアウト
        // リトライ
        ...
    case let .Failure(error): // その他
        // エラー
        ...
    }
}
```

JSONの例でも同じことが言えます。`MissingKey`からのみ回復させ、その他のエラーではエラーとしてハンドリングしたいです。

```swift
// [ Swift ]
let isAdmin: Bool
switch json["isAdmin"].bool {
case let .Success(admin): // 成功
    isAdmin = admin
case .Failure(.MissingKey): // キーがない
    // {} => false
    isAdmin = false
case let .Failure(error): // その他
    // [ true ] => error
    // { "isAdmin": 42 } => error
    ...
}
```

起こり得るすべてのエラーに対して処理を分岐する必要があることは非常にまれです。そして、実際にそれが必要な場合は、Swiftがすでに提供している*Associated Value*付きの `enum` で行うことができます。

```swift
// [ Swift ]
enum Foo {
    case Bar(A)
    case Baz
    case Qux(B)
}

func foo() -> Foo { ... }

switch foo() {
case let Bar(a):
    ...
case let Baz:
    ...
case let Qux(b):
    ...
}
```

私は、今述べたようなエラー型を指定しない `Result` を提供するライブラリResultK[^4]を実装しました。さまざまな型のエラーが混在していても、うまく機能します。

```swift
// [ Swift ]
let a: Result<Int> = ... // ErrorA
let b: Result<Int> = ... // ErrorB
let sum: Result<Int>= a.flatMap { a in b.map { b in a + b } } // ErrorAまたはErrorB 
```

エラー型を指定しない場合、（ `Int|FormatError` のような）シンタックスシュガーはどうすれば良いでしょうか。 `Result<Int>` の代わりに `Int|` とするのが良いかもしれません。

```swift
// [ Swift ]
let a: Int| = ...
let b: Int| = ...
let sum: Int| = a.flatMap { a in b.map { b in a + b } }
```

そのような、エラー型を指定しない `Result` は（型安全性を失ってしまったという意味で）ダークサイドかもしれません。しかし、私が考えた限りでは、今のところこれが最良の方法でした。

## 3. Try

:::message
ジョブズのスピーチ

> Death

僕の発表

> Try

1単語でそろえました。しかも、これは記念すべき最初のtry! Swiftでの発表で、最後の話のタイトルとしてTryはこれ以上なく適したもののように思いました。
:::

私の三つ目の話は `try` についてです。

### エラーの*Automatic Propagation*

Swift 2.0では、Javaの `try/catch` と似たような構文が導入されました。

```swift
// [ Swift ]
func toInt(string: String) throws -> Int {
    ...
}

do {
    let number = try toInt(string)
    ...
} catch let error {
    // ここでエラーハンドリング
    ...
}
```

私の第一印象は良くありませんでした。Java時代に逆戻りしたくはありませんでした。しかし、それを学ぶにつれ、かなり良いものだと思うようになりました。

SwiftのCore Teamは、"Error Handling Rationale and Proposal"[^5]という文書で、なぜ `try/catch` 構文を採用したのかを説明しています。

その文書の中で、Core Teamはエラーの***Manual Propagation***と***Automatic Propagation***を定義しています。*Manual Propagation*では、エラーは制御構文を用いてで手動でハンドリングされますが、*Automatic Propagation*では、エラーが発生すると自動的にハンドラにジャンプします。

```swift
// [ Swift ]
// Manual Propagation
switch(toInt(string)) {
case let .Success(number):
    ...
case let .Failure(error): // エラーを手動でハンドリング
    ...
}

// Automatic Propagation
do {
    let number = try toInt(string) // 自動的に`catch`にジャンプ
    ...
} catch let error {
    ...
}
```

*Automatic Propagation*は、特に複数のエラーをまとめてハンドリングしたい場合に便利です。*Manual Propagation*でも、 `map` や `flatMap` を使って関数型言語的にエラーハンドリングすることはできます。しかし、構文的に複雑で理論的に難しいです。

```swift
// [ Swift ]
// Manual Propagation
let a: Result<Int> = toInt(aString)
let b: Result<Int> = toInt(bString)
switch a.flatMap { a in b.map { b in a + b } } {
case let .Success(sum):
    ...
case let .Failure(error):
    ...
}

// Automatic Propagation
do {
    let a: Int = try toInt(aString)
    let b: Int = try toInt(bString)
    let sum: Int = a + b
    ...
} catch let error {
    ...
}
```

そのドキュメントの中で、Core TeamはHaskellの `do` 記法に関する興味深いトピックに言及しています。それ（ `do` 記法）は `flatMap` のチェーンとネストした `flatMap` を簡略化するための記法です。

```swift
// [ Swift ]
let sum = toInt(aString).flatMap { a in
    toInt(bString).flatMap { b in
        .Some(a + b)
    }
}
```

```haskell
-- [ Haskell ]
sum = do
    a <- toInt aString
    b <- toInt bString
    Just (a + b)
```

Core Teamは、これは一種の*Automatic Propagation*だと述べています。

:::message
実際、これは `Result` でのエラーハンドリングを `try/catch` で書き直したものと等価と考えられます。
:::

つまり、（結局Haskellのような関数型言語であったとしても、 `map` や `flatMap` だけで記述するのはわずらわしいので）関数型でも関数型でなくてもエラーハンドリングを簡潔な表記で書くためには、*Automatic Propagation*が必要になるということです。そのため、（当初Javaの `try/catch` のような構文をSwiftに導入するのは不安に思いましたが、最終的に）私は*Automatic Propagation*をSwiftに導入するのは良いことだと理解しました。

### Marked Propagation

私は、Untyped Throws（型付けされていない `throws` ）についても心配していました。

（Swiftでは）これまでのところ、 `throws` にエラーの型を指定できません。

```swift
// [ Swift ]
func toInt(string: String) throws FormatError -> Int { // コンパイルエラー
    ...
}
```

:::message
長い間、Typed Throws（型付けされた `throws` ）をSwiftに導入すべきか議論されてきましたが、最近ようやくProposalがAcceptされました。2016年当時から議論されていて、今が2024年であることを考えると、本当に長い間議論された末に導入されたことがわかります。

- [SE-0413: Typed throws](https://github.com/apple/swift-evolution/blob/main/proposals/0413-typed-throws.md)
:::

型安全ではないように見えますが、 `Result` の場合と同じ理由でそれ（エラーの型を型付けしないこと）は妥当だと思います。私が心配していたのは別のことでした。

Javaには*非検査例外*があります。それらをハンドリングしなくても、コンパイラは何も報告しません。C#やさまざまな動的型付け言語にも同様のメカニズムがあります。

```java
// [ Java ]
class FormatException extends RuntimeException { // 非検査例外
    ...
}
```

```java
// [ Java ]
static int toInt(String string) throws FormatException {
    ...
}
```

```java
// [ Java ]
String string = ...;
int number = toInt(string); // 非検査例外なのでコンパイルエラーにならない
```

それらの言語では、コードのすべての行で予期しないエラーがスローされる可能性があります。

```java
// [ Java ]
void foo() { // `foo`は何をthrowする可能性がある?
    a(); // 非検査例外をthrowするかもしれない
    b(); // 非検査例外をthrowするかもしれない 
    c(); // 非検査例外をthrowするかもしれない
    d(); // 非検査例外をthrowするかもしれない
    e(); // 非検査例外をthrowするかもしれない
    f(); // 非検査例外をthrowするかもしれない
    g(); // 非検査例外をthrowするかもしれない
}
```

そうなると、自分自身で実装した関数でさえ、どのような種類のエラーが実際に `throw` され得るのかわからなくなります。これは非常にまずいことです。エラーハンドリングを完璧にすることは不可能で、ケアレスミス（必要なエラーハンドリングを忘れる事態）を引き起こしかねません。

私は、それがSwiftでも起こり得ると思いました。Swiftには*非検査例外*はありません。しかし、一度関数に `throws` を追加すると、その関数のどの行がエラーをスローする可能性があるのかを知るのは難しくなります。そして、エラーの型を指定する必要がないため、その関数が `throws` するエラーの種類について不注意になります。

```swift
// [ Swift ]
func foo() throws { // `foo`は何をスローする可能性がある?
    a() // エラーをthrowする可能性がある?
    b() // エラーをthrowする可能性がある?
    c() // エラーをthrowする可能性がある?
    d() // エラーをthrowする可能性がある?
    e() // エラーをthrowする可能性がある?
    f() // エラーをthrowする可能性がある?
    g() // エラーをthrowする可能性がある?
}
```

しかし、Swiftでは `throws` を伴う関数を呼び出すときに、 `try` キーワードを付与することが強制されています。Core Teamはこれを***Marked Propagation***と呼んでいます。

```swift
// [ Swift ]
func foo() throws {
    a()
    try b() // エラーをthrowする可能性がある
    c()
    d()
    try e() // エラーをthrowする可能性がある
    f()
    g()
}
```

`try` があれば、どの行がエラーを `throw` する可能性があるのかが明確になります。そして、関数内でどのような種類のエラーがスローされる可能性があるのかを確認するのがずっと簡単になります（ `try` が付与された関数を確認すれば良いので）。

もし `throws` に型があれば、より安全になるでしょう。しかし、*Marked Propagation*は（前述のように）型付けされていない `throws` の最悪の部分を取り除きます。これは、型安全性とシンプルさのバランスを取った妥当なトレードオフだと思います。

*Marked Propagation*はコードを読むのにも役立ちます。*Automatic Propagation*では、（エラー発生時に）どの行から `catch` 節にジャンプし得るのかを理解するのが難しいです。これは"Error Handling Rationale and Proposal"の中で *Implicit Control Flow Problem（暗黙的な制御フロー問題）* として言及されています。*Marked Propagation*は、その暗黙的な部分をより明確にします。

```java
// [ Java ]
try {
    foo();
    bar();
    baz();
} catch (QuxException e) {
    // どこから来るかわからない
}
```

```swift
// [ Swift ]
do {
    foo()
    try bar()
    baz()
} catch let error {
    // bar()から来たのが明確
}
```

つまり、*Marked Propagation*は

- どのようなエラーが `throw` され得るのかわからなくなる
- *Implicit Control Flow Problem（暗黙的な制御フロー問題）*

という二つの問題の解決策なのです。私は、それは（言語の）進化だと思いました。

### `Optional` のためのMarked Propagation

さて、*Marked Propagation*が良いものなら、なぜそれを `Optional` に使わないのかという疑問が生まれます。

"Error Handling Rationale and Proposal"の中でCore Teamは、 `Optional` は*Simple Domain Error*に用いられるべきで、それには*Manual Propagation*が適していると述べています。 `toInt` はその例として挙げられていました。

```swift
// [ Swift ]
// Simple Domain ErrorのためのOptional
func toInt(string: String) -> Int? {
    ...
}

// Manial Propagation
guard let number = toInt(string) {
    // ここでエラーハンドリング
    ...
}
```

:::message
*Simple Domain Error*について補足します。

`toInt` がエラーとなるのは引数に渡された文字列が整数に変換不要な場合です。このような場合、エラーの発生条件が明確で、エラーが起こった際にその原因を特定するのも容易です。そのようなケースでは、 `nil` を返すことでエラーを表しても十分です。

そのような種類のエラーを、"Error Handling Rationale and Proposal"では*Simple Domain Error*と呼んでいます。
:::

しかし、私は `Optional` にも*Automatic Propagation*が役立つと考えています。エラーとしてだけでなく、単に空の値として `nil` を取得することもあります。私たちのコードは `Optional` だらけです。それらを手動でハンドリングするのはコストがかかります。


私は `Optional` のための*Automatic Propagation*をこのように提案します。

```swift
// [ Swift ]
// Manual Propagation
let a: Int? = toInt(aString)
let b: Int? = toInt(bString)
if let sum = (a.flatMap { a in b.map { b in a + b } }) {
    ...
} else {
    ...
}

// Automatic Propagation
do {
    let a: Int = try toInt(aString)
    let b: Int = try toInt(bString)
    let sum: Int = a + b
    ...
} catch {
    ...
}
```

この構文では、 `try` は一種のアンラップです。それを `catch` するか、戻り値の型を `Optional` にしなければいけません。私は、この構文には一貫性があると思います。

:::message
少しわかりづらいので説明を補足します。

ここで言いたいのは `throws/try` を `Optional` に拡張するとどうなるかという話です。通常の `throws/try` で、 `try` 付きで関数が呼び出された場合、 `do/catch` するか、呼び出し元の関数に `throws` を付与することが求められます。

```swift
// エラーをcatchする場合
func foo() -> Int { // throwsは必要ない
    do {
        try bar()
    } catch { // エラーをcatchする
        ...
    }
    return ...
}
```

```swift
// エラーをcatchしない場合
func foo() throws -> Int { // throwsが必要
    try bar()
    return ...
}
```

これを `Optional` に拡張すると次のようになるということを言っています。

```swift
// nilをcatchする場合
func foo() -> Int { // 戻り値はOptionalでなくて良い
    do {
        try bar()
    } catch { // nilをcatchする
        ...
    }
    return ...
}
```

```swift
// nilをcatchしない場合
func foo() -> Int? { // 戻り値の型をOptionalにする必要あり
    try bar()
    return ...
}
```
:::

### `Result` と `try`

この考え方は `Result` にも拡張できます。

`Result` と`throws`は理論的に交換可能です。もし `throws` が `Result` を返すことのシンタックスシュガーであれば、 `Result` と `throws` の両方の世界をシームレスに接続できるでしょう。

```swift
// [ Swift ]
// throwsの場合
func toInt(string: String) throws -> Int { // このように書けば
    ...
}
```

```swift
// [ Swift ]
// Resultの場合
func toInt(string: String) -> Result<Int> { // こう書いたのと同じ意味とする
    ...
}
```

```swift
// [ Swift ]
// throwsの場合
do {
    let a: Int = try toInt(aString) // tryとcatchでハンドリングしても良いし
    let b: Int = try toInt(bString)
    let sum: Int = a + b
    ...
} catch {
    ...
}
```

```swift
// [ Swift ]
// Resultの場合
let a: Result<Int> = toInt(aString) // Resultとして受け取って後でハンドリングしても良い
let b: Result<Int> = toInt(bString)
switch a.flatMap { a in b.map { b in a + b } } {
case let .Success(sum):
    ...
case let .Failure(error):
    ...
}
```

`Result` は、 `Optional` のように（変数に格納して取り回すことで）後からハンドリングするなど、エラーをより柔軟にハンドリングする方法を提供します。そのため、 `throws` と `Result` を自由に行き来して、そのときに取り扱いたい方法で取り扱えることが重要です。

### `Result` と `throws` を行き来できることが重要な例

例を示しましょう。

私は遅延評価される `List` を提供するライブラリListK[^6]を実装しました。これにより、無限リストを作成することができます。

無限であるにもかかわらず、処理が遅延評価されるため、 `map` することができます。

```swift
// [ Swift ]
let infinite: List<Int> = List { $0 } // [0, 1, 2, 3, 4, ...]
let square: List<Int> = infinite.map { $0 * $0 } // [0, 1, 4, 9, 16, ...]
```

:::message alert
Swiftで無限リストを作るのにこのようなライブラリを利用する必要はありません。 `PartialRangeFrom` を使えば次のように書けます。

```swift
let infinite: PartialRangeFrom<Int> = 0...
```

また、 `lazy` を使って `LazySquence` に変換し、 `map` の処理を遅延させることもできます。

```swift
let square = infinite.lazy.map { $0 & $0 }
```
:::

しかし、`throws`のある関数ではうまく機能しません。

```swift
// [ Swift ]
func toInt(string: String) throws -> Int {
    ...
}

let strings: List<String> = ... // ["0", "1", "2", ...]
do {
    // 決して終わらない
    let numbers: List<Int> = try strings.map(transform: toInt)
} catch let error {
    ...
}
```

:::message
当時のSwiftでは `map` の引数にラベルとして `transform` が付けられていました。そのため、 `array.map(transform: foo)` などと記述する必要がありました。

現在は、 `map` の引数のラベルは `_` に変更されたため、 `array.map(foo)` などと書くことができます。
:::

この `map` の処理は決して終わりません。なぜでしょうか。

`throws` のある `map` は、 `Result` を返す型で書くと次のように書けます。

```swift
// [ Swift ]
// throwsを使って
func map<U>(transform: T throws -> U) throws -> List<U>

// Resultを使って
func map<U>(transform: T -> Result<U>) -> Result<List<U>>
```

`Result` を返すにはその値が `.Success` か `.Failure` かを決定しなければいけません。そのためには、無限に続く要素すべてについて `transform` を評価しなければならず、決して終わりません。

私が `List` に求めているのは、次のようなものです。 `Result` と `List` が入れ替わっています。これなら遅延評価することができます。

```swift
// [ Swift ]
// throwsを使って
func map<U>(transform: T throws -> U) -> List<Result<U>>

// Resultを使って
func map<U>(transform: T -> Result<U>) -> List<Result<U>>
```

:::message alert
今の僕の考えでは、このような `map` はわかりづらいと思います。

`transform` で発生したエラーを `map` がrethrowすることと、要素として `Result` の `.failure` を返すことではまったく意味が異なります。本文にもあるように、前者は `Result<List<Int>>` を返すのと等価で、後者は `List<Result<Int>>` を返すことになります。

（ `Array` 等の） `Sequence` の `map` 等がすでに前者の設計で実装されているところに、 `LazySequence` だけ後者の設計にするのは混乱を招くと思います。それがやりたいのであれば、 `transform` に `throws` を付与するのではなく、単に `map` に `Result` を返すクロージャを渡せば良いでしょう。

実際に、標準ライブラリの `LazySequenceProtocol` が定義する `map` メソッドは、 `throws` が付与された `transform` を受け取りません。

```swift
func map<U>(_ transform: @escaping (Element) -> U) -> LazyMapSequence<Elements, U>
```
:::

これにより、 `throws` のある `transform` で無限リストを `map` することができるようになります。

```swift
// [ Swift ]
func toInt(string: String) throws -> Int {
    ...
}

let a: List<String> = ... // ["0", "1", "2", ...]
let b: List<Result<Int>> = strings.map(transform: toInt)
// [Result(0), Result(1), Result(2), ...]

let c: List<Result<Int>> = b.take(10)
// [Result(0), Result(1), Result(2), ..., Result(9)]
let d: Result<List<Int>> = sequence(c)
// Result([0, 1, 2, ..., 9])

do {
    let e: List<Int> = try d // [0, 1, 2, ..., 9]
    ...
} catch let error {
    // `FormatError`のハンドリング
    ...
}
```

`throws` と `Result` の世界を行き来し、必要に応じて `throws` ではなく `Result` でエラーを扱えることで、無限リストの `map` が可能となりました。

### `throws` を `Result` を返すことのシンタックスシュガーとすることの欠点

しかし、 `throws` を `Result` を返すことのシンタックスシュガーとすることの欠点もあります。

Swift 2.xでは、 `try` キーワードを書き忘れると、その場所でコンパイルエラーが発生します。（これは、*Marked Propagation*として前述したように望ましいことです。）

```swift
// Swift 2.x
let a = toInt(aString) // ここでコンパイルエラー
let b = toInt(bString)
let sum = a + b
```

しかし、 `throws` を `Reuslt` を返すことのシンタックスシュガーにしてしまうと、これを実現することができません（ `try` を忘れてもエラーにはならず、 `Result` を返すという意味になるため）。 `try` を忘れた箇所でエラーになる代わりに、 `Result` の値を使おうとしたところでコンパイルエラーが発生します。

```swift
// 私の提案によるSwift
let a = toInt(aString) // aの型はResult<Int>
let b = toInt(bString) // bの型はResult<Int>
let sum = a + b // ここでコンパイルエラー（Result<Int>同士を足すことはできない）
```

直観的ではなく、わかりづらいです。しかし、これまで見てきたように、全体としては `throws` を `Result` を返すことのシンタックスシュガーにするのは良い案だと思います。

### 非同期処理と `try`

さらに、私は `try` がエラーハンドリング以外の目的にも使えると考えています。一つの例は非同期処理です。

JavaScriptは非同期処理のために `Promise` をネイティブにサポートしています。その `then` メソッドは理論的には `map` と `flatMap` に相当します。私はSwiftで `map` と `flatMap` を使って `Promise` ライブラリ[^7]を実装しました。

```swift
// [ Swift ]
let a: Promise<Int> = asyncGetInt(...)
let b: Promise<Int> = asyncGetInt(...)
let sum: Promise<Int> = a.flatMap { a in b.map { b in a + b } }
```

これは`Result`とまったく同じように見えます。

```swift
// [ Swift ]
let a: Result<Int> = failableGetInt(...)
let b: Result<Int> = failableGetInt(...)
let sum: Result<Int> = a.flatMap { a in b.map { b in a + b } }
```

唯一の違いは、非同期処理を表すか、失敗し得る処理を表すかということです。

将来のJavaScriptは、C#で生まれた `async/await` 構文をサポートする予定です。この構文は `Promise` の上に成り立っており、 `then` のチェーンを簡単に書くことができます。

:::message
`async/await` が導入されたのはES2017なので、2016年当時は将来サポート予定の機能でした（Babelを使うなどして当時から使うことはできましたが）。今となってはあって当たり前のものなので、隔世の感があります。
:::

非同期処理は今日のプログラミングで最もホットなトピックの一つです。そのため、Swiftでも `async/await` 構文について議論する必要があると思います。

```csharp
// [ C# ]
async Task<int> AsyncGetInt(...) {
    ...
}

async void PrintSum() {
    int a = await AsyncGetInt(...);
    int b = await AsyncGetInt(...);
    Console.WriteLine(a + b);
}
```

C#の `async/await` 構文は次のように使われます。C#のこの `Task` クラスは `Promise` に相当します。

もしSwiftにこの構文があったら、次のようになるでしょう。

```swift
// [ Swift ]
func asyncGetInt(...) async -> Promise<Int> {
    ...
}

func printSum() async {
    let a: Int = await asyncGetInt(...)
    let b: Int = await asyncGetInt(...)
    print(a + b)
}
```

私は、Swiftではこの構文を変更して、戻り値を暗黙的に `Promise` にラップすべきだと思います。

```swift
// [ Swift ]
func asyncGetInt(...) async -> Int { // <- ここを変更
   ...
}
```

これで、`async/await`と`throws/try`の共通の関係が見えてきました。

```swift
// [ Swift ]
func asyncGetInt(...) async -> Int { // async
    ...
}
func printSum() async { // async
    let a: Int = await asyncGetInt(...) // await
    let b: Int = await asyncGetInt(...) // await
    print(a + b)
}
```

```swift
// [ Swift ]
func failableGetInt(...) throws -> Int { // throws
    ...
}

func printSum() throws { // throws
    let a: Int = try failableGetInt(...) // try
    let b: Int = try failableGetInt(...) // try
    print(a + b)
}
```

`async/await` 構文は `Promise` を返すことと等価で、 `throws/try` 構文は `Result` を返すことと等価です。これは完全に意味を成します。 `async/await/Promise` と `throws/try/Result` は、非同期処理についてのものか、失敗し得る処理についてのものかという1点だけが異なるだけで、共通の概念を表しています。

:::message
これは***モナド***と呼ばれる概念で説明されます。 `Promise` も `Result` もモナドなので、同じ形で取り扱えるのは自然なことです。
:::

ここで、 `try` を `await` の代わりとして使い、（ `async` や `throws` はやめて）単に `Promise` （や `Result` ）を返すようにすることで、非同期処理と失敗し得る処理を統合することができます。

```swift
// [ Swift ]
func asyncGetInt(...) -> Promise<Int> { // Promise
    ...
}

do {
    let a: Int = try asyncGetInt(...) // try
    let b: Int = try asyncGetInt(...) // try
    print(a + b)
}
```

`async`, `await`, `reasync` のような独立したキーワードがあると、コードを読みやすくなる点は良いです。しかし、新しい機能を追加するたびに新しいキーワードが際限なく必要になります。

:::message
*モナド*は `Result` と `Promise` だけではありません。新たな*モナド*を言語に追加したくなったときに、 `throws/try` や `async/await` に相当するキーワードを追加していくと、キリがありません。ここでは、そのようなスタイルではなく、共通の `try` でマークする形にしてはどうかということを提案しています。

実際に、Haskellではあらゆるモナドを `do` 記法で扱うことができます。
:::

`try` を `await` の代わりにするのが良いと確信しているわけではありません。 `await` の代わりに `try` を使うのが良いか、それとも `await` を導入するのが良いか、どちらが良いのか迷っています。ここでそれについて述べたのは、 `try` の可能性（すべてのモナドを扱うキーワードになれるかもしれないということ）を示したかったからです。

## まとめ

私はいくつかのアイデアを紹介しました。

- `Result<T, E>` の代わりに `Result<T>` にする
- `Optional` の*Automatic Propagation*
- `throws` を `Result` を返すことのシンタックスシュガーにする
- `async/await` と非同期処理のための `try`

それらがすべて良いものであるかはわかりません。ですから、意見を聞かせてください。このカンファレンスを通じて、Swiftのエラーハンドリングについて議論したいと思います。そして、私はswift-evolutionメーリングリストに参加するつもりです。

私は誰もがプログラミングの教育を受けている世界を夢見ています。教育に適したプログラミング言語を自分で設計しようともしました。

ある朝、私はSwiftに出会いました。Swiftは私の目的に適していると思いました。今、私は誰もが、"Hello, world!!"から*モナド*まで、幅広いプログラミングの概念を学べるように、Swiftを題材に、無料で読めるオンラインブックを書く計画を立てています。

:::message
残念ながら、このオンラインブックは何度か途中まで書きましたが、2024年3月現在完成していません。
:::

プログラミング言語を設計した経験から、私は、それは安全性と複雑さとの戦いだと言うことができます。別の言い方をすれば、それはこのように言えます。 **"Stay Typed. Stay Practical."（「型付けを保ちながら、実用的でもあり続けよう。」）** このことは、この発表を通してお話ししてきたように、（言語の）進化を生むと思います。 Stay Typed. Stay Practical. 私は、Swiftの設計者に常にそれを願ってきました。そして今、Swiftがオープンソースになったので、私たちに（も）それを願っています。

**Stay Typed. Stay Practical.**

皆さん、どうもありがとうございました。

:::message
"Stay Typed. Stay Practical."は当然"Stay Hungry. Stay Foolish."のオマージュです。

このまとめの最後の部分は、"Stay Typed. Stay Practical."を三度繰り返すことや、間の文の入れ方までジョブズのスピーチに合わせています。

ジョブズのスピーチ

> Beneath it were the words: “Stay Hungry. Stay Foolish.” It was their farewell message as they signed off. Stay Hungry. Stay Foolish. And I have always wished that for myself. And now, as you graduate to begin anew, I wish that for you.
> Stay Hungry. Stay Foolish.
> Thank you all very much.

僕の発表

> It can be said in other words: "Stay Typed. Stay Practical." I'm sure this will make the evolution as I talked through my presentation. Stay Typed. Stay Practical. And I have always wished that for Swift's designers. And now, as Swift became open source, I wish that for us.
> Stay Typed, Stay Practical.
> Thank you all very much.
:::

[^1]: "SwiftyJSON", [https://github.com/SwiftyJSON/SwiftyJSON](https://github.com/SwiftyJSON/SwiftyJSON)
[^2]: "thoughtbot/Runes", [https://github.com/thoughtbot/Runes](https://github.com/thoughtbot/Runes)
[^3]: "antitypical/Result", [https://github.com/antitypical/Result](https://github.com/antitypical/Result)
[^4]: "ResultK", [https://github.com/koher/ResultK](https://github.com/koher/ResultK)
[^5]: "Error Handling Rationale and Proposal", [https://github.com/apple/swift/blob/master/docs/ErrorHandlingRationale.rst](https://github.com/apple/swift/blob/master/docs/ErrorHandlingRationale.rst)
[^6]: "ListK" [https://github.com/koher/ListK](https://github.com/koher/ListK)
[^7]: "PromiseK" [https://github.com/koher/PromiseK](https://github.com/koher/PromiseK)
[^8]: "‘You’ve got to find what you love,’ Jobs says" [https://news.stanford.edu/2005/06/12/youve-got-find-love-jobs-says/](https://news.stanford.edu/2005/06/12/youve-got-find-love-jobs-says/)