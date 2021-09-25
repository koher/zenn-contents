---
title: "Swift Concurrency チートシート"
emoji: "🌟"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["swift"]
published: true
---

Swift 5.5 で Swift に Concurrency （並行処理）関連の言語機能が追加されました。これによって、 Swift で**非同期処理・並行処理のコードをより簡潔かつ安全に**書くことができるようになります。

しかし、 Swift Concurrency は [Structured Concurrency](https://github.com/apple/swift-evolution/blob/main/proposals/0304-structured-concurrency.md) や [Actor](https://github.com/apple/swift-evolution/blob/main/proposals/0306-actors.md) など、多くの人にとって馴染みが薄いだろうと思われる概念を含みます。具体例を通して効率よく Swift Concurrency を習得できるように、本記事では iOS アプリを題材に、 Swift Concurrency 導入以前（ **Before** ）と導入後（ **After** ）のコードを比較することで、何がどのように変わるのかを紹介します。

なお、 Swift Concurrency 関連の機能は次の三つに大別できるため、本記事の Before & After の例もそのように分類します。

- `async` / `await`
- Structured Concurrency
- Actor

:::message
本記事は iOSDC Japan 2021 のセッション ["async/awaitやactorでiOSアプリ開発がどう変わるか Before&Afterの具体例で学ぶ"](https://fortee.jp/iosdc-japan-2021/proposal/19c076c3-18cb-4d04-9e9d-59cc02220538) をベースにテキスト化したものです。

本記事の説明は前から読んで理解できるように書かれていますが、チートシートとしてやりたいことベースでコードの書き方を調べることもできます。そのような場合には目次をご活用下さい。
:::

# `async` / `await`

`async` / `await` は Swift Concurrency の一部ですが、 `async` / `await` 自体が並行処理を扱うわけではありません。 `async` / `await` は非同期処理に関する機能です。

## 💼 Case 1 (`async` / `await`): 非同期関数の利用（エラーハンドリングがない場合）

非同期処理はエラーハンドリングを必要とする場合が多いですが、まずは理解しやすいようにエラーハンドリングを伴わない例を考えてみます。

### Before

非同期関数が結果を非同期的に受け取るために、これまでは主にコールバック関数（ completion ハンドラーなど）が用いられてきました。

コールバック関数を用いると、たとえば URL を指定してデータをダウンロードし、結果としてバイト列を受け取る関数は、次のような形で宣言されます。

```swift
func downloadData(from url: URL, completion: @escaping (Data) -> Void)
```

これを使うコードは次のようになります。

```swift
downloadData(from: url) { data in
    // data を使う処理
}
```

クロージャ式の引数として、非同期的に結果の `data` を受け取ります。

### After

`async` / `await` を使うと、 `download` は次のように宣言されます。

```swift
func downloadData(from url: URL) async -> Data
```

completion ハンドラーがなくなり、戻り値として `Data` が返されます。また、 `async` というキーワードが、普段 `throws` を書く位置に記述されています。

これを使うコードは次のようになります。

```swift
let data = await downloadData(from: url)
// data を使う処理
```

戻り値として結果を受け取るため、クロージャ式はどこにも現れません。 `downloadData` はデータをダウンロードする関数なので、その結果を戻り値として返すのは自然です。

`async` 関数を呼び出すときには `await` を付けることが必須です。 `await` を書かないとコンパイルエラーになります。これは、 `throws` が付与された関数を呼び出す場合に `try` が必須なのと同様です。

:::message
この `downloadData` 関数は普通の関数と似た見た目をしていますが、非同期的に実行されることに注意が必要です。 `downloadData` を呼び出すと処理が suspend （中断）され、非同期的に結果が得られてから続きが resume （再開）されます。

さらに言うと、次のようなケースで `print("A")` と `print("B")` は同じスレッドで実行される保証すらありません。

```swift
print("A")
let data = await downloadData(from: url)
print("B") // "A" と同じスレッドとは限らない
```

`downloadData` の呼び出し後にどのようなスレッドで処理が再開されるかは `downloadData` の実装次第です。

このことは、コールバック関数を用いて書かれた次のコードで、 `print("A")` と `print("B")` が同じスレッドで実行されるとは限らないのと同じことです。

```swift
print("A")
downloadData(from: url) { data in
    print("B") // "A" と同じスレッドとは限らない
}
```
:::

**参考文献**

- [SE-0296: Async/await](https://github.com/apple/swift-evolution/blob/main/proposals/0296-async-await.md)
- [Meet async/await in Swift (WWDC 2021)](https://developer.apple.com/videos/play/wwdc2021/10132/)

## 💼 Case 2 (`async` / `await`): 非同期関数の実装（エラーハンドリングがない場合）

Case 1 では非同期関数を利用するコードを扱いましたが、次は自分で非同期関数を実装する場合を考えます。

### Before

API を叩いて JSON を取得し、それをデコードする関数を考えます。

ここでは例として、ユーザーの JSON を取得・デコードして返す非同期関数 `fetchUser` を考えてみます。 Case 1 の `downloadData` 関数を用いると、 `fetchUser` 関数は次のように書けます。ユーザーの ID を受け取り、 completion ハンドラーで `User` を返します。

```swift
func fetchUser(for id: User.ID, completion: @escaping (User) -> Void) {
    let url: URL = .init(string: "https://koherent.org/fake-service/api/user?id=\(id)")!
    
    downloadData(from: url) { data in
        let user = try! JSONDecoder().decode(User.self, from: data)
        completion(user)
    }
}
```

`downloadData` に渡したクロージャ式の中で JSON を受け取ってデコードし、得られた結果を `fetchUser` が受け取った completion ハンドラーに渡します。

### After

`async` / `await` を利用すると、 `fetchUser` 関数は次のように書けます。

```swift
func fetchUser(for id: User.ID) async -> User {
    let url: URL = .init(string: "https://koherent.org/fake-service/api/user?id=\(id)")!
    
    let data = await downloadData(from: url)
    let user = try! JSONDecoder().decode(User.self, from: data)
    return user
}
```

ポイントは、 `fetchUser` 関数の宣言に `async` を付与することです。

completion ハンドラーを使わずに戻り値として結果を返せば良いため、デコードされた `user` を単純に `return` します。

**参考文献**

- [SE-0296: Async/await](https://github.com/apple/swift-evolution/blob/main/proposals/0296-async-await.md)
- [Meet async/await in Swift (WWDC 2021)](https://developer.apple.com/videos/play/wwdc2021/10132/)

## 💼 Case 3 (`async throws` / `try await`): 非同期関数の利用（エラーハンドリングがある場合）

これまでの例ではエラーハンドリングを無視してきましたが、多くの非同期処理はエラーを引き起こす可能性があります。エラーハンドリングと非同期処理がどのように両立されてきたかを見てみます。

### Before

Case 1 の `downloadData` 関数がエラーを引き起こす場合を考えます。処理の結果として `Data` または `Error` を受け取る必要があるため、 completion ハンドラーの引数の型が `Result<Data, Error>` となっています。

```swift
func downloadData(from url: URL,
    completion:@escaping (Result<Data, Error>) -> Void)
```

これを使う場合、次のようにしてエラーハンドリングします。

```swift
downloadData(from: url) { result in
    do {
        let data = try result.get()
        // data を使う処理
    } catch {
        // エラーハンドリング
    }
}
```

:::message
`result` のエラーハンドリングは `switch` 文を使ってパターンマッチングで行うこともできます。筆者は、エラーハンドリング専用の `catch` を使った方が可読性が高いと思うので、このような書き方をしています。複数のエラーをまとめてハンドリングしたい場合にも便利です。
:::

### After

`async` / `await` を使うと `downloadData` 関数は次のように宣言されます。

```swift
func downloadData(from url: URL) async throws -> Data
```

エラーを返す可能性があるため、 `async` に加えて `throws` が付与されています。 `async throws` はこの順に記述する必要があります。

これを利用するコードは次のようになります。

```swift
do {
    let data = try await downloadData(from: url)
    // data を使う処理
} catch {
    // エラーハンドリング
}
```

通常の関数と同じように、 `catch` を使ってエラーハンドリングすることができます。非同期処理の前後を `do` - `catch` で囲むことができるのが、おもしろいところです。

`async throws` が付与されている場合、関数を呼び出すときには `try await` を書く必要があります。 `try await` はこの順である必要があります。

:::message
`try await` と `async throws` で順番が逆になっていることが（ `async` に `await` が、 `throws` に `try` が対応するなら `await try` でないかと）気になるかもしれません。この順番については、利用と宣言で順番が逆になると覚えると覚えやすいです。

たとえば、 `downloadData` の宣言時

```swift
func downloadData(from url: URL) async throws -> Data
```

には `downloadData`, `async`, `throws`, `Data` の順ですが、利用時

```swift
let data = try await downloadData(from: url)
```

には `data`, `try`, `await`, `downloadData` と順番が逆転します。

理論的には次のように説明できます。 `throws` は `Result` を返すのと同等です。同じように、 `async` は `Future` を返すのと同等と考えることができます（この `Future` は Combine のものと違い、純粋に非同期処理だけを扱い、エラーを扱わないものとします）。このとき、非同期かつエラーになりうる関数の戻り値は `Future<Result<Foo>>` にようになってほしいはずです。もし `Result<Future<Foo>>` だとエラーであるかが同期的に決まってしまうため、非同期的に得られるエラーを扱うことができません。非同期的に得られる結果（ `Future` ）が、成功か失敗か（ `Result` ）であるため、 `Future` が外側である必要があります。これに対応させると `async throws` という順になります。

`Future<Result<Foo>>` を剥がして `Foo` を取り出すときには、まず `Future` を剥がしてから `Result` を剥がさなければなりません。そのため、先に `await` するために `try (await foo)` である必要があり、 `try await` の順となります。
:::

**参考文献**

- [SE-0296: Async/await](https://github.com/apple/swift-evolution/blob/main/proposals/0296-async-await.md)
- [Meet async/await in Swift (WWDC 2021)](https://developer.apple.com/videos/play/wwdc2021/10132/)

## 💼 Case 4 (`async throws` / `try await`): 非同期関数の実装（エラーハンドリングがある場合）

### Before

Case 2 の `fetchUser` がエラーを返す可能性がある場合を考えます。

```swift
func fetchUser(for id: User.ID,
        completion: @escaping (Result<User, Error>) -> Void) {
    let url: URL = .init(string: "https://koherent.org/fake-service/api/user?id=\(id)")!
    
    downloadData(from: url) { result in
        do {
            let data = try result.get()
            let user = try JSONDecoder().decode(User.self, from: data)
            completion(.success(user))
        } catch {
            completion(.failure(error))
        }
    }
}
```

`downloadData` が `result` を受け取るため、これを `do` - `try` - `catch` でエラーハンドリングします。エラーを `catch` した場合にはそれを completion ハンドラーに渡します。

### After

`async` / `await` を使うと `fetchUser` 関数は次のように書けます。

```swift
func fetchUser(for id: User.ID) async throws -> User {
    let url: URL = .init(string: "https://koherent.org/fake-service/api/user?id=\(id)")!
    
    let data = try await downloadData(from: url)
    let user = try JSONDecoder().decode(User.self, from: data)
    return user
}
```

Before と比べてずいぶんとすっきりしています。見比べると、 After には `do` - `try` - `catch` が存在しないことがわかります。

これは、 `downloadData` や `JSONDecoder.decode` の `try` を、 `fetchUser` の `throws` で受けているためです。わざわざ `catch` して `throw` しなおさなくても、 `fetchUser` には `throws` が付与されているのでそのままエラーを投げることができます。そのため、エラーを `catch` して completion ハンドラーにリレーしている Before と比べてコードが簡潔になっています。

Swift には Swift 2.0 の頃から、言語の提供するエラーハンドリング機構として `throws` / `try` がありました。しかし、エラーが発生し得ることの多い非同期処理では、コールバック関数を挟むため `throws` / `try` を上手く活用することができませんでした（関数に `throws` を付与してハンドリングさせることができず、コールバック関数に `Result` を渡すことでエラーを扱っていました）。 `async` / `await` は上記の例のように `trhows` / `try` と相性がよく、 `async` / `await` が導入されたことで `throws` / `try` もより活用しやすくなります。

**参考文献**

- [SE-0296: Async/await](https://github.com/apple/swift-evolution/blob/main/proposals/0296-async-await.md)
- [Meet async/await in Swift (WWDC 2021)](https://developer.apple.com/videos/play/wwdc2021/10132/)

## 💼 Case 5 (`async throws` / `try await`): 非同期関数の連結

次は二つの非同期処理を連結する場合についてです。

ここでは例として、ユーザーのアイコンを取得する関数 `fetchUserIcon` を考えます。ただし、アイコンの URL は ユーザーの JSON の中に書かれており、

1. JSON をダウンロードしてデコードし、アイコンの URL を取得
2. アイコンの URL にアクセスしてダウンロード

という 2 段階の非同期処理を行う必要があるとします。

### Before

この 2 段階の非同期処理をコールバック関数を用いて書くと次のようになります。複雑なコードになっていますが、ポイントはコールバック関数がネストしていることです。

```swift
func fetchUserIcon(for id: User.ID,
        completion: @escaping (Result<Data, Error>) -> Void) {
    let url: URL = .init(string: "https://koherent.org/fake-service/api/user?id=\(id)")!

    downloadData(from: url) { data in // (1)
        do {
            let data = try data.get()
            let user = try JSONDecoder().decode(User.self, from: data)
            downloadData(from: user.iconURL) { icon in // (2)
                do {
                    let icon = try icon.get()
                    completion(.success(icon))
                } catch {
                    completion(.failure(error))
                }
            }
        } catch {
            completion(.failure(error))
        }
    }
}
```

最初に JSON をダウンロードするために `downloadData` 関数を呼び出し、結果をクロージャ式で受けています `(1)` 。そして、その JSON をデコードした後に得られた `iconURL` から、再び `downloadData` 関数でアイコンをダウンロードします `(2)` 。

この二つの非同期処理の結果をそれぞれコールバック関数で受け取るため、コールバック関数がネストします。このように、非同期処理を連結するとその度にコールバック関数がネストし、いわゆるコールバック地獄と呼ばれる状態を作り出してしまいます。

:::message
コールバック地獄の解消には `Future` が用いられることもあります。 Combine を導入して `Future` と `flatMap` を用いればネストを解消することができます。

しかし、 `async` / `await` の方が簡潔でパフォーマンスもよく、 `async` / `await` が使えるなら、あえて `Future` を使うべきケースは限定的でしょう（どうしても `Publisher` として扱いたい場合など）。
:::

### After

`async` / `await` を使うと `fetchUserIcon` 関数は次のように書けます。

```swift
func fetchUserIcon(for id: User.ID) async throws -> Data {
    let url: URL = .init(string: "https://koherent.org/fake-service/api/user?id=\(id)")!

    let data = try await downloadData(from: url)
    let user = try JSONDecoder().decode(User.self, from: data)
    let icon = try await downloadData(from: user.iconURL)
    return icon
}
```

コールバック関数を用いた場合と異なり、非常に簡潔です。 JSON をダウンロードし、デコードし、得られた URL からアイコンをダウンロードするという 3 段階の処理が、そのまま 3 行のコードとして現れています。

今、やりたいことの本質はこの三つのはずです。逆に言えば、 Before の複雑なコードのほとんどは、本質的なオペレーションとは関係のない、非同期処理を扱うためのボイラープレートだと言えます。

Before のコードが複雑になっている原因の一つは、それぞれのコールバック関数で別々にエラーハンドリングが行われていることです。この二つの `do` - `try` - `catch` を一つにまとめることはできません。 `do` - `try` - `catch` がコールバックのクロージャ式を越えることができないからです。 `async` / `await` を使うと、いくつの非同期処理を連結しても、すべての `try` を `throws` に引き受けさせることができます。これが、 `async` / `await` を利用するコードが著しく簡潔になっている理由です。

さらに、 Before のコードでは何箇所にも分散して `completion` の呼び出しを書かなければなりません。たとえば、もしエラーを `catch` したときにログ出力だけして `completion(.failure(error))` を忘れてしまうと、その非同期関数は永久に結果を返さない関数になってしまいます。このようなバグは発見するのが困難です。 `async` 関数の場合、必ず戻り値か `throws` で結果を返す必要があるため、そのようなことは起こりません。戻り値を `return` しなければコンパイルエラーになりますし、エラーが発生した場合にはそれを `throw` する以外に脱出する手段が存在しないからです。

つまり `async` / `await` を使うと、コードが簡潔になるだけでなく、安全にもなるということです（さらに言えば、パフォーマンスも向上します）。

**参考文献**

- [SE-0296: Async/await](https://github.com/apple/swift-evolution/blob/main/proposals/0296-async-await.md)
- [Meet async/await in Swift (WWDC 2021)](https://developer.apple.com/videos/play/wwdc2021/10132/)

## 💼 Case 6 (`CheckedContinuation`, `withChecked(Throwing)Continuation`): コールバックから `async` への変換

:::message
この Case には Before がありません。
:::

### After

これまでの Case で `async` / `await` が優れているのはわかりました。しかし現実問題として、すべての API がすぐに `async` / `await` になるわけではありません。たとえば、利用しているサードパーティライブラリの非同期 API がコールバックのままになっているかもしれません。

そのような場合でも、コールバック版をラップして `async` 版を自力実装することができれば、アプリケーションコードは `async` / `await` で記述することができます。そんなときに活躍するのが Continuation です。

コールバック版の `downloadData` 関数を使って、 `async` 版の `downloadData` 関数を実装することを考えてみます。

```swift
// コールバック
func downloadData(from url: URL,
    completion: @escaping (Result<Data, Error>) -> Void)
```

`async` 版の `downloadData` 関数は Continuation を使って次のように実装できます。

```swift
// async
func downloadData(from url: URL) async throws -> Data {
    try await withCheckedThrowingContinuation { continuation in // (2)
        downloadData(from: url) { result in // (1)
            do {
                let data = try result.get()
                continuation.resume(returning: data) // (3)
            } catch {
                continuation.resume(throwing: error) // (4)
            }
        }
    }
}
```

コールバック版の `downloadData` 関数を呼び出しても、その結果はコールバック関数の引数としてしか受け取れません `(1)` 。なんとかそれを `async` 版の戻り値として `return` しなければ（もしくは、エラーの場合は `throw` しなければ）なりません。

この変換を行ってくれるのが、新しく標準ライブラリに追加された `withCheckedThrowingContinuation` 関数です。 `withCheckedThrowingContinuation` 関数に渡されたクロージャは、引数として `continuation` を受け取ります `(2)`。 `continuation` の `resume(returning:)` メソッドに結果を渡せばそれを `withCheckedThrowingContinuation` の戻り値として `return` してくれますし `(3)`、 `resume(throwing:)` メソッドにエラーを渡せばエラーとして `throw` してくれます `(4)`。

なお、今回のようにコールバック関数が `Result` を受け取っている場合には、 `conitnuation` の `resume(with:)` メソッドが便利です。結果とエラーをそれぞれ `resume(returning:)` と `resume(throwing:)` に渡す代わりに、 `resume(with:)` メソッドに直接 `Result` を渡すことができます。

```swift
// async （ resume(with:) を使う場合）
func downloadData(from url: URL) async throws -> Data {
    try await withCheckedThrowingContinuation { continuation in
        downloadData(from: url) { result in
            continuation.resume(with: result)
        }
    }
}
```

:::message
もし `throws` が必要ない場合には、代わりに `withCheckedContinuation` 関数を利用します。

また、 `withUnsafeContinuation` ・ `withUnsafeThrowingContinuation` という関数もあります。これらの関数を利用した場合、 `continuation` がチェックを行わない分だけわずかにパフォーマンスが向上します。その代わり、 `withChecked(Throwing)Continuation` が行っていたチェック（複数回 `resume` したり、 `resume` することなく `continuation` が破棄された場合に実行時エラーで知らせる）が行われません。個人的には、一般的なアプリケーションでは `withChecked(Throwing)Continuation` が適切なケースが多いのではないかと思います。
:::

**参考文献**

- [SE-0300: Continuations for interfacing async tasks with synchronous code](https://github.com/apple/swift-evolution/blob/main/proposals/0300-continuation.md)
- [SE-0296: Async/await](https://github.com/apple/swift-evolution/blob/main/proposals/0296-async-await.md)
- [Meet async/await in Swift (WWDC 2021)](https://developer.apple.com/videos/play/wwdc2021/10132/)

# Structured Concurrency

ここからは Structured Concurrency に分類される例を紹介します。

## 💼 Case 7 (`Task.init`): 非同期処理の開始

同期関数から非同期関数を呼び出すケースを考えます。

例として、ユーザー情報を表示する View Controller 、 `UserViewController` を考えてみます。 `UserViewController` は `viewDidAppear` で（ Case 4 の） `fetchUser` 関数を呼び出し、ユーザー情報を取得、画面上に表示します。 `viewDidAppear` は同期的なメソッドなので、同期→非同期の呼び出しが必要になります。

### Before

コールバックによる従来型の非同期関数を呼び出す場合、特に難しいところはありません。単に `viewDidAppear` から `fetchUser` を呼び出すだけです。

```swift
extension UserViewController {
    override func viewDidAppear(_ animated: Bool) {
        super.viewDidAppear(animated)

        fetchUser(for: userID) { [self] user in
            do {
                let user = try user.get()
                nameLabel.text = user.name
            } catch {
                // エラーハンドリング
            }
        }
    }
}
```

### After

`fetchUser` が `async` 関数の場合、次のような問題があります。

`async` / `await` には、 `async` 関数は `async` 関数の中でしか呼び出せないという制約があります。もし、（ `async` でない）普通の関数の中で `async` 関数を呼び出すとコンパイルエラーになります。これは、 `throws` が付与された関数は、 `throws` が付与された関数の中でしか呼び出せないのと同じことです。

`throws` の場合は `do` - `try` - `catch` でハンドリングすることで、 `throws` が付与されて**いない**関数の中でも呼び出すことができました。しかし `async` には `catch` に相当するものがありません。

では、 `veiwDidAppear` から `fetchUser` を呼び出すにはどうすれば良いでしょうか。 `veiwDidAppear` は `async` メソッドでないため、 `veiwDidAppear` の中で `async` な `fetchUser` 関数を呼び出すことはできません。

そんなときに役立つのが `Task` です。 `Task` を使うと次のようにして、（同期的な） `veiwDidAppear` の中から、 `async` な `fetchUser` 関数を呼び出すことができます。

```swift
extension UserViewController {
    override func viewDidAppear(_ animated: Bool) {
        super.viewDidAppear(animated)

        Task {
            do {
                let user = try await fetchUser(for: userID)
                nameLabel.text = user.name
            } catch {
                // エラーハンドリング
            }
        }
    }
}
```

わかりづらいですが、この `Task { }` は `Task` のイニシャライザの呼び出しです。 `{ }` は Trailing Closure なので、 `Task(operation: { })` と書いているのと同じです。つまり、 `Task` のイニシャライザに、引数としてクロージャ式を渡している形です。

この `Task` のイニシャライザに渡すクロージャの型は `@escaping () async -> Success` です。これに `async` が付与されているので、 `{ }` の中で `async` 関数を呼び出すことができます。そのため、 `fetchUser` も問題なく呼び出せます。

`Task` はイニシャライズされると即座に渡されたクロージャを実行しますが、その完了を待たずにイニシャライザは終了します。そのため、上記の例で `Task { }` がメインスレッドをブロックすることはありません。 `fetchUser` の結果を待たずに `viewDidAppear` は完了します。

このように、 `Task` を使えば同期関数から `async` 関数を開始することができます。 `viewDidAppear` の例を取り上げましたが、 `@IBAction` や（ iOS 14 からの） [`UIControl` の `addAction(_:for:)`](https://developer.apple.com/documentation/uikit/uicontrol/3600490-addaction) などで `async` 関数を呼びたいときも同様です。

ここで重要なのが、**すべての `async` 関数は必ず `Task` の上で実行される**ということです。 `async` 関数は `async` 関数からしか呼び出せないので、コールスタックを遡ると必ず `async` 関数を呼び出す入口が必要となります。そこで `Task` が用いられるため、すべての `async` 関数は `Task` に紐付けられ、 `Task` 上で実行されます。

:::message
`Task` のイニシャライザに渡すクロージャには `@escaping` が付与されていますが、 `nameLabel.text = user.name` では明示的に `self.` を付けることなく `UserViewController` のプロパティにアクセスできています。これは、 [Implicit `self`](https://github.com/apple/swift-evolution/blob/main/proposals/0304-structured-concurrency.md#implicit-self) と呼ばれるもので、 `Task` のイニシャライザなどいくつかの API で採用されています。

[標準ライブラリのコード](https://github.com/apple/swift/blob/main/stdlib/public/Concurrency/Task.swift)上では、現状で

```swift
@_implicitSelfCapture operation: __owned @Sendable @escaping () async -> Success
```

のように `@_implicitSelfCapture` という隠し属性を使って実現されているようです。

なぜ `self` を暗黙的に strong キャプチャしても問題ないのでしょうか。その理由は、 `Task` のイニシャライザに渡されるクロージャはリファレンスサイクルを作らないからです。このクロージャは実行が完了すれば解放されるため、ここでキャプチャされた `self` がリファレンスサイクルを作り、メモリリークにつながる恐れはありません。そのような理由で Implicit `self` が採用されています。
:::

**参考文献**

- [SE-0304: Structured concurrency](https://github.com/apple/swift-evolution/blob/main/proposals/0304-structured-concurrency.md)
- [SE-0296: Async/await](https://github.com/apple/swift-evolution/blob/main/proposals/0296-async-await.md)
- [Explore structured concurrency in Swift (WWDC 2021)](https://developer.apple.com/videos/play/wwdc2021/10134/)
- [Meet async/await in Swift (WWDC 2021)](https://developer.apple.com/videos/play/wwdc2021/10132/)

## 💼 Case 8: （欠番）

:::message
iOSDC Japan 2021 の発表とそろえるため、 Case 8 は欠番とします。
:::

## 💼 Case 9 (`async let`): 並行処理（固定個数の場合）

ようやく並行処理です。並行処理は、複数の処理を同時に実行するための仕組みです。

ここでは、例として大小二つのアイコンを並行してダウンロードすることを考えます。 SNS などでは用途に応じて解像度の異なる複数のユーザーアイコン画像が用意されている場合があります。 small と large 、二つのアイコンを並行してダウンロードする関数 `fetchUserIcons` を考えてみましょう。

| small アイコン | large アイコン |
|---|---|
| ![](/images/swift-concurrency-cheatsheet/icon.png =60x) | ![](/images/swift-concurrency-cheatsheet/icon.png =180x) |

### Before

（ Case 4 などで見てきた）コールバック版の `downloadData` 関数を使って、 `fetchUserIcons` 関数を実装します。

コールバック版の場合、並行処理自体は特に難しくありません。 `downloadData` 関数を二つ並べれば並行に実行されます。

問題は二つの処理が終わるのを待ち合わせて completion ハンドラーを呼び出さなければならない点です。ここでは `DispatchGroup` を使って待ち合わせを実現しています（ `DispatchGroup` の使い方については本題ではないので説明を省略します）。

```swift
func fetchUserIcons(for id: User.ID, completion:
        @escaping (Result<(small: Data, large: Data), Error>) -> Void) {
    let smallURL: URL = .init(string: "https://koherent.org/fake-service/data/user-icons/small/\(id).png")!
    let largeURL: URL = .init(string: "https://koherent.org/fake-service/data/user-icons/large/\(id).png")!
    
    let group: DispatchGroup = .init()

    var smallIcon: Result<Data, Error>!
    group.enter()
    downloadData(from: smallURL) { icon in
        smallIcon = icon
        group.leave()
    }
    
    var largeIcon: Result<Data, Error>!
    group.enter()
    downloadData(from: largeURL) { icon in
        largeIcon = icon
        group.leave()
    }
    
    group.notify(queue: .global()) {
        do {
            let icons = try (small: smallIcon.get(), large: largeIcon.get())
            completion(.success(icons))
        } catch {
            completion(.failure(error))
        }
    }
}
```

### After

同じく、（ Case 4 の） `async` 版の `downloadData` 関数を使って `fetchUserIcons` 関数を実装します。

次の「よくない例」のようなコードでは並行処理を実現できません。 small と large の二つのアイコンをダウンロードするために `downloadData` 関数を 2 回呼び出していますが、それぞれ `await` しているので、 small をダウンロードしてから large をダウンロードと、順番にダウンロードすることになってしまいます。

:::details よくない例
```swift
func fetchUserIcons(for id: User.ID) async throws -> (small: Data, large: Data) {
    let smallURL: URL = .init(string: "https://koherent.org/fake-service/data/user-icons/small/\(id).png")!
    let largeURL: URL = .init(string: "https://koherent.org/fake-service/data/user-icons/large/\(id).png")!

    let smallIcon = try await downloadData(from: smallURL)
    let largeIcon = try await downloadData(from: largeURL)
    let icons = (small: smallIcon, large: largeIcon)
    return icons
}
```
:::

並行処理を実現するためには **`async let` Binding** を利用します。 `async` 関数を呼び出すときは `await` する必要がありますが、 `async let` Binding を利用すると `await` なしに `async` 関数を呼び出すことができます。

`async let` Binding を利用すると、次のように Before と比べてはるかに簡潔に並行処理のコードを記述することができます。

```swift
func fetchUserIcons(for id: User.ID) async throws -> (small: Data, large: Data) {
    let smallURL: URL = .init(string: "https://koherent.org/fake-service/data/user-icons/small/\(id).png")!
    let largeURL: URL = .init(string: "https://koherent.org/fake-service/data/user-icons/large/\(id).png")!

    async let smallIcon = downloadData(from: smallURL) // (1)
    async let largeIcon = downloadData(from: largeURL) // (2)
    let icons = try await (small: smallIcon, large: largeIcon) // (3)
    return icons
}
```

`async let smallIcon = ...` のように `async let` で定数宣言を行い、右辺で `async` 関数を呼び出すことを `async let` Binding と呼びます `(1)`。このとき、 `downloadData` の呼び出しに `await` を付ける必要はなく（ `try` も必要ありません）、関数を呼び出すと結果を待たずに即時 `(2)` に進みます。

`(2)` も同様で、 `downloadData(from: largeURL)` の後、結果を待たずに即時 `(3)` に進みます。

そして、 `async let` で宣言された定数を `(3)` で使おうとしたときに、初めて `await` （と `try` ）が必要になります。このようにして、利用時まで `await` を遅らせることで、並行に関数を実行することができます。

`async let` にはもう一つ特筆すべき特徴があります。それは、 `async let` で宣言された定数は、そのスコープを抜ける前に必ず `await` されなければならないということです。 `async let` のまま `return` するなど、スコープの外に持ち出すことはできません。これによって、 `async let` で開始された非同期処理は、必ずその呼び出し元の `async` 関数よりも先に完了することが保証されています。このように非同期関数のライフタイムがスコープで構造化されていることが、 **Structured Concurrency** という名前の意味するところです。

:::message
`async let` で宣言された定数をスコープ内で `await` しなかった場合、スコープの抜ける前に自動的に `await` されます。
:::

Case 7 ですべての `async` 関数は Task の上で実行されると述べましたが、一つの Task は同時に一つの処理しか実行できません。 `async let` で呼び出される `async` 関数は並行に実行される必要があるため、同じ Task 上では実行することができません。このとき、元の Task から派生した Child Task が作られます。

`async let` で宣言された定数はそのスコープで必ず `await` されるという構造化によって、 Child Task は親の Task より先に終了することが保証されています。 Child Task が親の Task と独立に生き続けることはありません。この親子関係は木構造で表すことができます。

![](/images/swift-concurrency-cheatsheet/task-tree.png)

Child Task の中でさらに `async let` が用いられるなどした場合、上図のように孫 Task が作られることになります。この場合でも構造化は生きており、必ず Task Tree の末端（葉）から順に終了することが保証されます。

:::message
Structured Programming は構造化プログラミング（ Structured Programming ）から着想を得ています。構造化プログラミング以前は、 goto を使うなどして制御フローが複雑に絡み合ったコードを書くことが可能でした。構造化プログラミングによってコードが構造化され、そのような状況が改善されました。

たとえば、次のコードでは `for` 文の中に `if` 文がありますが、この `if` 文の分岐が `for` 文のスコープを越えて生き残ることはできません。

```swift
for value in values {
    if value.isValid {
        ...
    } else {
        ...
    }
}
```

これと同じように、 Structured Concurrency は並行処理のコードに、スコープによる構造化をもたらします。

なお、 Structured Programming は Swift の発明ではありません。比較的新しい概念ですが、 Kotlin などに先行して取り入れられています。
:::

**参考文献**

- [SE-0317: async let bindings](https://github.com/apple/swift-evolution/blob/main/proposals/0317-async-let.md)
- [SE-0304: Structured concurrency](https://github.com/apple/swift-evolution/blob/main/proposals/0304-structured-concurrency.md)
- [Explore structured concurrency in Swift (WWDC 2021)](https://developer.apple.com/videos/play/wwdc2021/10134/)

## 💼 Case 10 (`(Throwing)TaskGroup`, `with(Throwing)TaskGroup`): 並行処理（可変個数の場合）

次は可変個数の並行処理を考えます。 `async let` Binding を使えば固定個数の並行処理は簡単に記述できました。しかし、可変個数の場合は `async let` Binding が使えません。

ここでは例として、 N 人のユーザーのアイコンをダウンロードする関数を考えます。ユーザー ID の `Array` を渡して、それらのユーザーのアイコンを並行してダウンロードします。

### Before

Before については Case 9 と大差ありません。 Case 9 同様に `DispatchGroup` を使って、並行に実行された非同期処理の待ち合わせをしています。

```swift
func fetchUserIcons(for ids: [User.ID], completion: @escaping (Result<[User.ID: Data], Error>) -> Void) {
    var results: [User.ID: Result<Data, Error>] = [:]
    let group: DispatchGroup = .init()
    for id in ids {
        let url: URL = .init(string: "https://koherent.org/fake-service/data/user-icons/small/\(id).png")!
        
        group.enter()
        downloadData(from: url) { icon in
            results[id] = icon
            group.leave()
        }
    }
    
    group.notify(queue: .global()) {
        do {
            let icons = try results.mapValues { try $0.get() }
            completion(.success(icons))
        } catch {
            completion(.failure(error))
        }
    }
}
```

### After

この例では `ids` が `Array` で渡されます。 Case 9 のように個数が二つと決定されているような場合は `async let` に書き下すことができますが、 `ids` の要素数は実行時に決定されるため、一つずつ `async let` に書き下すことができません。

`ids` を `for` 文で一つずつ取り出して `async let` を書くわけにもいきません。なぜなら、 `async let` は宣言されたスコープ内で `await` する必要があるため、すべての ID について並行に処理を実行して、後でまとめて `await` することができないからです。

これを実現するには `TaskGroup` （や `ThrowingTaskGroup` ）を使います。次のコードでは、 `withThrowingTaskGroup` 関数に渡したクロージャの引数として `group` （ `ThrowingTaskGroup` インスタンス）を受け取っています。

```swift
func fetchUserIcons(for ids: [User.ID]) async throws -> [User.ID: Data] {
    try await withThrowingTaskGroup(of: (User.ID, Data).self) { group in // (1)
        for id in ids {
            group.addTask { // (2)
                let url: URL = .init(string: "https://koherent.org/fake-service/data/user-icons/small/\(id).png")!
                return try await (id, downloadData(from: url)) // (3)
            }
        }
        
        var icons: [User.ID: Data] = [:]
        for try await (id, icon) in group { // (4)
            icons[id] = icon
        }
        return icons // (5)
    }
}
```

まず、 `(1)` でクロージャ式の引数として `group` を受け取ります。

`(2)` で `addTask` メソッドを使って、 `group` に Child Task を追加します。 `async let` の場合と同様に、それらの Child Task は並行に実行されます。 `addTask` のクロージャの中で `await` していますが `(3)` 、 `addTask` 自体は `await` しておらず、このクロージャは並行して実行されます。そのため、 `ids` のループは個々の `id` に対するダウンロードの結果を待たずにイテレートすることができます。

このように並行して Child Task を実行した後で、 `group` から Child Task の結果を取り出すときに `await` して待ち合わせます `(4)` 。これは、 `async let` で宣言された定数を利用するときに `await` が必要なのとよく似ています。このような 2 段階に分けて処理を行うことで、並行な実行と待ち合わせを実現しています。

`(4)` についてもう一つ注目すべきは、 `for` 文で `group` から結果を取り出すときに `for try await` と書かれていることです。これまで `for` 文でイテレートするには `Sequence` プロコトルに適合することが必要でしたが、 Swift Concurrency で新しく `AsyncSequence` が追加されました。 `AsyncSequence` に対して `for` 文を使うときには、 `for await` または `for try await` で値を受け取る必要があります。 `for await` （ `for try await` ）を使うことで、非同期的に得られる値を待ちながら順番に取り出すことができます。

最後に、 `(5)` で `return` した結果が `(1)` の `withThrowingTaskGroup` の結果として返されます。

この一連の処理を振り返ると、 `addTask` で作られる Child Task のライフタイムが `group` に縛られており（ `withThrowingTaskGroup` のスコープを越えて生存できない）、構造化されていることがわかります。

:::message
Case 9 のように固定個数の処理を並行で実行したい場合にも、 `async let` Binding ではなく `TaskGroup` を使うことができます。 `TaskGroup` が使えればどのようなケースにも対応できますが、コードがやや複雑になります。 `async let` Binding は固定個数の場合に、簡潔に並行処理を記述するためのものと考えると良いでしょう。
:::

**参考文献**

- [SE-0304: Structured concurrency](https://github.com/apple/swift-evolution/blob/main/proposals/0304-structured-concurrency.md)
- [SE-0298: Async/Await: Sequences](https://github.com/apple/swift-evolution/blob/main/proposals/0298-asyncsequence.md)
- [Explore structured concurrency in Swift (WWDC 2021)](https://developer.apple.com/videos/play/wwdc2021/10134/)
- [Meet AsyncSequence (WWDC 2021)](https://developer.apple.com/videos/play/wwdc2021/10058/)

## 💼 Case 11 (`task.cancel`): 非同期処理のキャンセル（非同期 API の利用側）

非同期処理のキャンセルを正しく行うのは大変です。特に、非同期処理が別の非同期処理に依存している場合には、一連の非同期処理をすべて正しくキャンセルする必要があります。 Structured Concurrency は非同期処理のキャンセルを簡単にしてくれます。

ここでは、 Case 4 などで見てきた `downloadData` 関数をキャンセルする例を考えてみます。

### Before

コールバック関数を用いた非同期関数の場合、戻り値を使ってキャンセルを実現されることが多いです。

```swift
func downloadData(
    from url: URL,
    cancellation: @escaping () -> Void,
    completion: @escaping (Result<Data, Error>) -> Void
) -> DownloadCanceller
```

上記のように、戻り値で canceller が返され、それを使ってキャンセルを実行します。また、キャンセルされたことをハンドリングできるように、（ completion ハンドラーとは別に）キャンセルをハンドリングするためのコールバック関数（上記の例では `cancellation` ）が渡せることもあるでしょう。

ダウンロードボタンを押すとダウンロードを開始し、キャンセルボタンを押すとダウンロードをキャンセルする例を考えてみます。

次のように `@IBAction` のメソッドの中で `downloadData` を呼び出し、 `canceller` を `ViewController` のプロパティに保存します。

```swift
extension ViewController {
    @IBAction func downloadButtonPressed(_ sender: UIButton) {
        canceller = downloadData(from: url, cancellation: {
            // キャンセル時の処理
        }, completion: { data in
            do {
                let data = try data.get()
                // data を使う処理
            } catch {
                //  エラーハンドリング
            }
        })
    }
}
```

キャンセルボタンが押されたときには、この `canceller` を使ってキャンセルを実行します。

```swift
extension ViewController {
    @IBAction func cancelButtonPressed(_ sender: UIButton) {
        canceller?.cancel()
        canceller = nil
    }
}
```

`URLSession` の `dataTask` メソッドなども、これと似たような API 設計になっています（キャンセルのハンドリングは delegate を使いますが）。

### After

`async` 関数の場合、同じ方法は使えません。戻り値は結果を返すために使われるからです。 `(Data, DownloadCanceller)` のようにして、無理やり結果と canceller をペアにして返すことはできますが、その場合 canceller が得られるのは、 `await` で suspend してから resume された後です。つまり、 canceller が得られたときにはすでに非同期処理が完了していることになります。戻り値で canceller を返すことに意味はありません。

`async` 関数をキャンセルするには `Task` を利用します。 Case 7 では `Task` のイニシャライザを使って非同期処理を開始しましたが、イニシャライズされた `Task` インスタンス自体は使いませんでした。 `Task` インスタンスには `cancel` メソッドが用意されており、 `cancel` メソッドを呼ぶことで Task をキャンセルすることができます。

Before と同じように `task` を `ViewController` のプロパティに保存し、キャンセルボタンが押されたら `task` の `cancel` メソッドを使ってキャンセルします。

```swift
extension ViewController {
    @IBAction func downloadButtonPressed(_ sender: UIButton) {
        task = Task {
            do {
                let data = try await downloadData(from: url)
                // data を使う処理
            } catch {
                if Task.isCancelled {
                    // キャンセル時の処理
                } else {
                    //  エラーハンドリング
                }
            }
        }
    }
}
```

```swift
extension ViewController {
    @IBAction func cancelButtonPressed(_ sender: UIButton) {
        task?.cancel()
        task = nil
    }
}
```

`cancel` メソッドが呼ばれた場合、 `async` 関数はエラーを `throw` することで処理を中断します。そのため、キャンセルのハンドリングは `catch` によって行います。

キャンセルでもエラーが発生した場合でも、処理を中断することに違いはありません。キャンセルかエラーかの区別が必要ない場合は単に `catch` すれば良いですが、ときにはキャンセルとエラーを区別したい場合もあります。

たとえば今回のケースでは、ダウンロード中にエラーが起こったらアラートを表示して、ユーザーに知らせるのが親切でしょう。しかし、キャンセルボタンが押された場合は、ユーザーの意思によるものなのでアラートを表示する必要はありません。このような場合、キャンセルかどうかを区別して分岐する必要があります。そのような分岐は、上記のコードのように `Task.isCancelled` を調べることで実現できます。

:::message
慣例上、 Task がキャンセルされた場合には `CancellationError` が `throw` されることになっています。そのため、次のようなコードでもキャンセルをハンドリングできるように思えます。

```swift
do {
    ...
} catch let error as CancellationError {
    // キャンセル時の処理
} catch {
    // エラーハンドリング
}
```

しかし、**キャンセル時に本当に `CancellationError` が `throw` されるかは実装依存**であり、上記の方法は確実ではありません。 `catch` 後に `Task.isCancelled` で分岐することをおすすめします。
:::

Task は、 Case 9 で見たように Task Tree を作ります。

![](/images/swift-concurrency-cheatsheet/task-tree.png)

Task がキャンセルされると Task Tree がトラバースされ、 Tree 全体がキャンセルされます。 Before のようにコールバック関数で非同期処理が実装されている場合、このような Tree 全体のキャンセルは大変です。一つずつ注意深くキャンセルするしかありません。 After では、根っこの Task をキャンセルするだけで、 Tree 全体のキャンセルを簡単に扱うことができます。これは、 Structured Concurrency の構造化のおかげだと言えます。

**参考文献**

- [SE-0304: Structured concurrency](https://github.com/apple/swift-evolution/blob/main/proposals/0304-structured-concurrency.md)
- [Explore structured concurrency in Swift (WWDC 2021)](https://developer.apple.com/videos/play/wwdc2021/10134/)

## 💼 Case 12 (`Task.checkCancellation()`, `Task.isCancelled`): 非同期処理のキャンセル（非同期 API の実装側①）

:::message
この Case には Before がありません。
:::

### After

Case 11 は、 `async` 関数を呼び出す側で、どのようにキャンセルを扱うかの例でした。 `Task` の `cancel` メソッドが呼び出された場合に、自作の `async` 関数を正しく中断させるには注意が必要です。

`Task` の `cancel` メソッドは、それ自体は `isCancelled` のフラグを立てるだけで、キャンセルを実行するわけではありません。処理を中断するのは非同期関数の実行側の責務です。

多くの場合、 `async` 関数は他の既存の `async` 関数を組み合わせて実装されることになります。その場合、それらの（既存の） `async` 関数が正しくキャンセル処理を行っていれば、自作の `async` 関数では特別な処理を書かなくても自動的にキャンセルが実行されます。利用している `async` 関数が `CancellationError` を `throw` することで、自作の `async` 関数も `CancellationError` を `throw` するからです。

しかし、自分でキャンセル処理を実装しないといけない場合もあります。利用している `async` 関数が `CancellationError` を `throw` してくれるのは、その関数を呼び出して `await` しているときだけです。もし、 `await` から `await` までの間に重めの同期処理を行っている場合、その途中で `Task` がキャンセルされていないかをチェックして、キャンセルされていた場合には `CancellationError` を `throw` する必要があります。

ここでは例として、動画像処理を考えます。動画の中に何人の歩行者が写っているか数える関数 `countPedestrians` を考えてみましょう。

`countPedestrians` は動画を受け取ると、動画から 1 フレームずつ画像を取り出し、画像処理して歩行者をカウントします。とても重い処理なので、非同期関数として実装されるのが適しているでしょう。当然、 `Task` がキャンセルされた場合には途中で処理を中断したいです。

つまり、動画像処理の途中で Task がキャンセルされているかを定期的にチェックし、キャンセル済みの場合には `CancellationError` を `throw` したいということになります。 Task がキャンセル済みかは `Task.isCancelled` で調べることもできますが、キャンセルのチェックとエラーの `throw` を行うには `Task.checkCancellation()` が便利です。 `checkCancellation` メソッドは、 Task がキャンセル済みかをチェックし、もしキャンセルされていた場合には `CancellationError` を `throw` してくれます。

`checkCancellation` メソッドを使うと、 `countPedestrians` 関数は次のように実装できます。

```swift
func countPedestrians(in video: AVAsset) async throws -> Int {
    let reader: AVAssetReader = try .init(asset: video)
    // ...
    guard reader.startReading() else {
        throw IOError(...)
    }
    let output = reader.outputs[0]
    
    var count = 0
    while let buffer = output.copyNextSampleBuffer() {
        try Task.checkCancellation() // キャンセルのチェック & 処理の中断
        // 毎フレームの処理
    }
    return count
}
```

`output.copyNextSampleBuffer()` メソッドで動画から 1 フレームずつ画像を取り出し、処理を実行します。これを `while` ループで繰り返しますが、毎フレーム最初に `checkCancellation` を呼び出すことで、 `Task` がキャンセル済みの場合は処理を中断します。

キャンセル時に単に処理を中断すれば良い場合は、このように `checkCancellation` メソッドが便利です。しかし、処理を中断するだけでなく、クリーンアップ処理などキャンセルに付随した処理が必要となる場合もあります。そのようなケースでは `Task.isCancelled` で分岐し、クリーンアップ処理を行ってから明示的に `CancellationError` を `throw` します。

```swift
// キャンセルに付随した処理が必要な場合
func countPedestrians(in video: AVAsset) async throws -> Int {
    let reader: AVAssetReader = try .init(asset: video)
    // ...
    guard reader.startReading() else {
        throw IOError(...)
    }
    let output = reader.outputs[0]
    
    var count = 0
    while let buffer = output.copyNextSampleBuffer() {
        if Task.isCancelled { // キャンセルのチェック
            reader.cancelReading() // クリーンアップ
            throw CancellationError() // 処理の中断
        }
        // 毎フレームの処理
    }
    return count
}
```

**参考文献**

- [SE-0304: Structured concurrency](https://github.com/apple/swift-evolution/blob/main/proposals/0304-structured-concurrency.md)
- [Explore structured concurrency in Swift (WWDC 2021)](https://developer.apple.com/videos/play/wwdc2021/10134/)

## 💼 Case 13 (`withTaskCancellationHandler`): 非同期処理のキャンセル（非同期 API の実装側②）

:::message
この Case には Before がありません。
:::

### After

`downloadData` 関数を `URLSession` の `dataTask` メソッドを使って実装する場合、次のようになります。この例では Case 12 のような方法でキャンセルを実装するのは困難です。実際の処理は `dataTask` メソッドの中に隠蔽されており、 `Task.checkCancellation()` を呼び出すために介入できる箇所がありません。

:::details キャンセルできない例
```swift
func downloadData(from url: URL) async throws -> Data {
    try await withCheckedThrowingContinuation { continuation in
        let task = URLSession.shared.dataTask(with: url) { data, response, error in
            if let error = error {
                continuation.resume(throwing: error)
                return
            }
            
            if let response = response as? HTTPURLResponse {
                guard response.statusCode == 200 else {
                    continuation.resume(throwing: ServerError(statusCode: response.statusCode))
                    return
                }
            }
            
            continuation.resume(returning: data!)
        }
        task.resume()
    }
}
```
:::

このような場合には `withTaskCancellationHandler` 関数が役立ちます。

```swift
func downloadData(from url: URL) async throws -> Data {
    var canceller: URLSessionDataTask?
    return try await withTaskCancellationHandler {
        try await withCheckedThrowingContinuation { continuation in
            let task = URLSession.shared.dataTask(with: url) { data, response, error in
                if let error = error {
                    continuation.resume(throwing: error)
                    return
                }
                
                if let response = response as? HTTPURLResponse {
                    guard response.statusCode == 200 else {
                        continuation.resume(throwing: ServerError(statusCode: response.statusCode))
                        return
                    }
                }
                
                continuation.resume(returning: data!)
            }
            canceller = task
            task.resume()
        }
    } onCancel: {
        canceller?.cancel()
    }
}
```

Task がキャンセルされると、 `withTaskCancellationHandler` の `onCancel` に渡したクロージャが実行されます。これによってキャンセル時に介入し、 `task` のキャンセルにつなぐことができます。

:::message alert
上記のコードは [Proposal のコード](https://github.com/apple/swift-evolution/blob/main/proposals/0304-structured-concurrency.md#cancellation-handlers) を参考に書かれていますが、残念ながら次のようなコンパイルエラーになります。

```
⛔ Reference to captured var 'urlSessionTask' in concurrently-executing code
```

これは、並行処理の安全性のために、 `var` で宣言された `canceller` を `onCancel` のクロージャでキャプチャすることができないためです。

同様の問題は Proposal のコードにも存在し、 Proposal のコードもコンパイルすることができません。無理やりなワークアラウンドで回避することは可能ですが、良い解決法が見つかるまでは、 Proposal に沿った上記のコードのままとしておきたいと思います。
:::

**参考文献**

- [SE-0304: Structured concurrency](https://github.com/apple/swift-evolution/blob/main/proposals/0304-structured-concurrency.md)
- [Explore structured concurrency in Swift (WWDC 2021)](https://developer.apple.com/videos/play/wwdc2021/10134/)

# Actor

## 💼 Case 14 (`actor`): 共有された状態の変更

並行処理において、共有された状態を同時に読み書きするとデータ競合が発生します。

データ競合の例として、単純なカウンターを考えてみます。

```swift
final class Counter {
    private var count: Int = 0

    func increment() -> Int {
        count += 1
        return count
    }
}
```

内部に `count` を持ち、 `increment` メソッドが呼ばれると `count` を 1 増やして返すだけのシンプルなカウンターです。状態を共有するために `Counter` はクラスとして宣言されています（値型のインスタンスは共有されずにコピーされてしまうため）。

このとき、共有されたカウンターに対して同時に 2 回 `increment` メソッドを呼び出すと何が起こるでしょうか。

```swift
let counter: Counter = .init()
    
DispatchQueue.global().async {
    print(counter.increment()) // ?
}
    
DispatchQueue.global().async {
    print(counter.increment()) // ?
}
```

期待すべき挙動は、片方が 1 を、もう片方が 2 を返すことです。どちらがわずかに早く実行されるかは運次第ですが、いずれにせよ片方が 1 でもう片方が 2 であることが期待されます。

しかし、現実には両方 2 を返すこともあります。なぜなら、今 `increment` メソッドは同時に呼び出されるため、まず両者の `count += 1` が実行され、その後でそれぞれの `return count` が実行されることがあり得るからです。この場合、 2 回インクリメントされた後の値が `return` されるので、両方とも 2 を返すことになります。これがデータ競合です。

データ競合を防ぐには `increment` メソッドが同時に実行されないようにする必要があります。

### Before

データ競合を防ぐための古典的な手法はロックです。しかし、ロックはスレッドをブロックしてしまうためパフォーマンスがよくありません。メインスレッドをブロックするような場合には UI がフリーズする原因にもなります。また、（カウンターのような単純な処理はともかく）複雑な並行処理をロックで制御しようとするとデッドロックの危険が高まります。

そのため、 iOS アプリ開発では多くの場合 `DispatchQueue` のシリアルキューが用いられてきました。これを使って `Counter` を実装すると次のようになります。

```swift
final class Counter {
    private let queue: DispatchQueue = .init(label: ...)
    private var count: Int = 0

    func increment(completion: @escaping (Int) -> Void) {
        queue.async { [self] in
            count += 1
            completion(count)
        }
    }
}
```

シリアルキューは、キューに入れられたオペレーションを一つずつ取り出し、順に実行します。 `Counter` に `queue` を持たせ、 `increment` メソッドのオペレーションを丸ごと `queue` に入れることで、オペレーションが同時に実行されることを防いでいます。

ただし、オペレーションを `queue` に入れてから実行するということは、結果は非同期的に得られることになります。そのため、 completion ハンドラーを用いて結果を返します。

### After

Swift Concurrency で導入された `actor` を用いれば、より簡潔に安全な `Counter` を実装することができます。

`actor` はクラスと同じような参照型の型を宣言します。 `actor` を使うと、データ競合を起こさない安全な `Counter` は次のように書けます。

```swift
actor Counter {
    private var count: Int = 0

    func increment() -> Int {
        count += 1
        return count
    }
}
```

このコードは最初に挙げた安全**でない** `Counter` クラスのコードとほとんど同じです。異なるのは、 `final class` が `actor` になった点だけです。

`actor` はインスタンスごとに内部にキューを持ちます（ Actor の用語では mailbox と呼びます）。 `actor` のメソッドを呼び出すと、そのオペレーション（ Actor の用語では message と呼びます）は自動的にキューに追加され、一つずつ順番に実行されます。そのため、上記の `increment` メソッドは必ず同時に一つしか実行されないことが保証されます。

つまり、 `actor` は Before で `DispatchQueue` を用いて書いたような処理を、自動的に行なってくれると言えます。 `increment` メソッドは非同期的に実行されるため、外部から呼び出す場合には `async` メソッドに見えるようになっています。

```swift
// 外部から見た increment メソッド
func increment() async -> Int
```

そのため、 `increment` の呼び出しには `await` が必要です。

```swift
let counter: Counter = .init()

Task.detached {
    print(await counter.increment()) // 1 or 2
}

Task.detached {
    print(await counter.increment()) // 2 or 1
}
```

このように `actor` のキューによって、共有された状態をデータ競合から守る仕組みを **actor isolation** と呼びます。

:::message
`actor` が内部に保持しているキューは serial executor と呼ばれます。 serial executor は、概念的には `DispatchQueue` とよく似ていますが、（デフォルトでは） `DispatchQueue` を用いて実装されているわけではありません。より軽量な実装となっており、 `DispatchQueue` と比べてパフォーマンス上も優れています。
:::

**参考文献**

- [SE-0306: Actors](https://github.com/apple/swift-evolution/blob/main/proposals/0306-actors.md)
- [Protect mutable state with Swift actors (WWDC 2021)](https://developer.apple.com/videos/play/wwdc2021/10133/)

## 💼 Case 15 (`actor`): 共有された状態の変更（インスタンス内でのメソッド呼び出し）

Case 14 の `increment` メソッドは外部からしか呼ばれていませんが、同じインスタンスの内部から `Counter` のメソッドを呼ぶケースを考えてみます。

ここでは、（あまり現実的な例ではないですが） 2 回カウンターをインクリメントする `incrementTwice` メソッドを考えます。すでに `increment` メソッドがあるので、これを使って `incrementTwice` メソッドを実装します。

### Before

`DispatchQueue` & コールバック関数による `Counter` の場合、単純に考えると次のように `incrementTwice` メソッドを実装してしまいそうです。しかし、これは競合状態を引き起こします。

:::details 競合状態を引き起こす実装
```swift
final class Counter {
    private let queue: DispatchQueue = .init(label: ...)
    private var count: Int = 0

    func increment(completion: @escaping (Int) -> Void) { ... }

    func incrementTwice(completion: @escaping (Int) -> Void) {
        increment { [self] _ in
            increment { count in
                completion(count)
            }
        }
    }
}
```
:::

`incrementTwice` メソッドを 2 回同時に呼び出した場合、期待される結果は片方が 2 、もう片方が 4 を返すことです。しかし、上記のような実装の場合、そのような結果が得られるとは限りません。

`increment` メソッドのオペレーションは `queue` で守られています。しかし、 2 回呼び出される `increment` メソッドのオペレーションはバラバラに `queue` に追加されます。そのため、 2 回のインクリメントは一度に同期的に実行されるわけではなく、 2 回に分けて実行されます。すると、 2 回のオペレーションの途中の状態が外部から観測されてしまう可能性があります。たとえば前述の例では、 2 と 4 ではなく 3 と 4 が返される場合があります。

これを防ぐには、 2 回のインクリメントを同期的に実行する必要があります。しかし、 `increment` メソッドは非同期なので、同期的に 2 回呼び出すことはできません。そこで、 `increment` メソッドとは別に、同期的なインクリメントを行う `_increment` メソッドを実装します。このとき、**非**同期的な `increment` メソッドは `_increment` の呼び出しを `queue` に入れる形で実装できます。

`_increment` メソッドがあれば、 2 回のインクリメントを同期的に行うことが可能になります。 `incrementTwice` メソッドは、 `queue` に入れたオペレーションの上で、 `_increment` メソッドを 2 回同期的に呼び出します。

```swift
final class Counter {
    private let queue: DispatchQueue = .init(label: ...)
    private var count: Int = 0

    private func _increment() -> Int {
        count += 1
        return count
    }

    func increment(completion: @escaping (Int) -> Void) {
        queue.async { [self] in
            _increment()
            completion(count)
        }
    }

    func incrementTwice(completion: @escaping (Int) -> Void) {
        queue.async { [self] in
            _increment()
            _increment()
            completion(count)
        }
    }
}
```

このようにすれば、 `incrementTwice` による 2 回のインクリメントの途中状態が外部から観測されるのを防ぐことができます。 `queue` は同時に一つのオペレーションしか実行せず、同期的なオペレーションは分断されることはありません。そのため、（ `increment` や `incrementTwice` を用いて）外部から `queue` を介して `count` を観測する限り、途中状態が観測されることはありません。

`_increment` メソッドは `private` になっていることに注意して下さい。データ競合を防ぐためには、 `queue` を介さない `_increment` メソッドを公開するわけにはいきません。

このように、共有された状態をインスタンス外部からは非同期に、内部（一度 `queue` に乗せてしまって）からは同期的に扱えるようにすることで、データ競合や競合状態を防ぐことができます。しかし、適切にデータを保護するには内部用と外部用に、同期版・非同期版のメソッドを二重に提供することが求められます。それらは定型的な処理で、そのような処理を扱うコードはボイラープレートとなります。

### After

`actor` を用いれば難しいことは何もありません。 `incrementTwice` メソッドから単純に `increment` メソッドを 2 回呼び出すだけです。

```swift
actor Counter {
    private var count: Int = 0

    func increment() -> Int {
        count += 1
        return count
    }

    func incrementTwice() -> Int {
        _ = increment()
        _ = increment()
        return count
    }
}
```

`actor` のメソッドは外部からは `async` に見えましたが、インスタンス内部からはただの同期メソッドに見えます。これは、インスタンス内部ではオペレーションはすでにキューに保護された状態で実行されており、改めてキューに乗せる必要がないからです。そのため、 `incrementTwice` から `increment` メソッドを呼び出す際に `await` は不要です。

これは、 Before で実現したかったこと（外部からは非同期（ `async` ）に、内部からは同期に）を自動的に実現しているということです。コードの二重化なしにこれを実現できるのが、 `actor` の最も重要な機能です。

**参考文献**

- [SE-0306: Actors](https://github.com/apple/swift-evolution/blob/main/proposals/0306-actors.md)
- [Protect mutable state with Swift actors (WWDC 2021)](https://developer.apple.com/videos/play/wwdc2021/10133/)

## 💼 Case 16 (`actor`, `get async`): 共有された状態の変更（ getter ）

Case 14, 15 では `count` プロパティを外部から隠蔽していました。 Case 14, 15 の `Counter` では、カウントを観測するにはインクリメントが必須です。しかし、インクリメントすることなくカウントを取得したいケースも考えられます。どうすれば安全に `count` を公開できるかを考えてみます。

### Before

`queue` を介さずに `count` の値を返してしまうとデータ競合を引き起こす可能性があります。 `queue` を介して値を返すにはコールバック関数が必要なので、 `count` をプロパティではなくメソッドに変更します。

```swift
final class Counter {
    private let queue: DispatchQueue = .init(label: UUID().uuidString)
    
    private var _count: Int = 0
    func count(_ handler: @escaping (Int) -> Void) {
        queue.async { [self] in
            handler(_count)
        }
    }
    
    func increment(completion: @escaping (Int) -> Void) {
        queue.async { [self] in
            _count += 1
            completion(_count)
        }
    }
}
```

ここでも、 Case 15 で見られたのと同じように、非同期的な `count` と同期的な `_count` の二重化が見られます。

### After

`actor` を用いる場合、単に `count` の `private` を外せば OK です（次のコードでは、 `set` は公開したくないので `private(set)` にしています）。

```swift
actor Counter {
    private(set) var count: Int = 0
    
    func increment() -> Int {
        count += 1
        return count
    }
}
```

`actor` のメソッドに外部からアクセスする場合は `async` に見えましたが、プロパティである `count` はどのように見えるのでしょうか。

Swift 5.5 で [Effectful Read-only Properties](https://github.com/apple/swift-evolution/blob/main/proposals/0310-effectful-readonly-properties.md) が追加され、プロパティにも `throws` や `async` を付与することが可能となりました。これを用いて、 `count` は外部からアクセスされた場合、 `async` なプロパティのように振る舞います。つまり、外部から `count` にアクセスするには `await` が必要になります。

```swift
print(await counter.count)
```

:::message
Effectful Read-only Properties は名前の通り read-only です。今のところ `set` を `async` にすることはできません。

上記の例では `count` を `private(set)` にしましたが、これがなくても外部から `count` を変更することはできません。 `actor` に外部からアクセスするには `async` である必要がありますが、 `set async` なプロパティは Swift 5.5 では作れないからです。

しかし、将来的に `set async` が実現される可能性はあります。 `Counter` の仕様上 `count` の `set` を許容しないのであれば `private(set)` にしておく方が良いと思います。
:::

**参考文献**

- [SE-0310: Effectful Read-only Properties](https://github.com/apple/swift-evolution/blob/main/proposals/0310-effectful-readonly-properties.md)
- [SE-0306: Actors](https://github.com/apple/swift-evolution/blob/main/proposals/0306-actors.md)
- [Protect mutable state with Swift actors (WWDC 2021)](https://developer.apple.com/videos/play/wwdc2021/10133/)

## 💼 Case 17 (`actor`): 共有された状態の変更（複数インスタンスの連携）

:::message
この Case には Before がありません。
:::

### After

Case 16 で見たように、 `actor` のインスタンス内部からは同じインスタンスのメソッドが同期的に見えます。しかし、これはあくまでも同じインスタンスに限った話です。同じ型の `actor` であっても、インスタンスが異なれば内部に持つキューも異なります。そのため、別のインスタンスのメソッドを呼び出すときには `async` として扱う必要があります。

ここでは、 `Counter` のあるインスタンスから別のインスタンスへ、カウントを転送する例を考えてみます。あまり現実的な例ではありませんが、 `Counter` ではなく口座間送金などの処理であれば似たような状況が起こり得ます。

今、 `Counter` は `increment` メソッドに加えて `decrement` メソッドを持っているとします。このとき、別のカウンターにカウントを 1 だけ転送する `transferCount` メソッドは次のように書けます。

```swift
actor Counter {
    var count: Int = 0

    func increment() -> Int {
        count += 1
        return count
    }

    func decrement() -> Int {
        count -= 1
        return count
    }

    func transferCount(to another: Counter) async {
        _ = self.decrement()
        _ = await another.increment()
    }
}
```

`transferCount` メソッドでは、自分のカウントをデクリメントしてから転送先カウンターのカウントをインクリメントします。こうすることでカウントが 1 転送されたように振る舞わせることができます。

`another.increment()` に `await` が付与されていることに注意して下さい。自分（ `self` ）の `decrement` メソッドは同期的に呼び出すことができますが、転送先（ `another` ）の `increment` メソッドは同期的に呼び出すことができません。同じ型の `actor` でもインスタンスが異なれば異なるキューで守られているため、外部からのアクセスという扱いになり、 `async` なメソッドとして振る舞うからです。当然、呼び出す際には `await` が必要となります。

このため、 `transferCount` メソッド自体も明示的に `async` である必要があります。このように明示的に `async` が付与された場合、同じインスタンス内から `transferCount` メソッドを呼び出す場合にも `async` メソッドとして振る舞います。

**参考文献**

- [SE-0306: Actors](https://github.com/apple/swift-evolution/blob/main/proposals/0306-actors.md)
- [Protect mutable state with Swift actors (WWDC 2021)](https://developer.apple.com/videos/play/wwdc2021/10133/)

## 💼 Case 18 (`actor`, `ObservableObject`): 共有された状態の変更（非同期処理結果の反映）

カウンターよりも現実的な例として、 ViewModel で非同期処理を行い、その非同期処理結果を View に反映する例を考えてみます。このとき、 ViewModel の状態が同時に読み書きされないよう（データ競合を引き起こさないように）に、状態を保護する必要があります。

Case 7 の `UserViewController` を再度、例として取り上げます。 Case 7 では `viewDidAppear` から直接 `fetchUser` を呼び出していました。ロジックを直接 View Controller に記述するのは望ましくないので、 `fetchUser` を呼び出して状態を変更する処理を ViewModel に移動します。そして、 `viewDidAppear` からは単に処理をトリガーするだけにします。

### Before

ViewModel のクラス名を `UserViewState` とすると、 `actor` を使って次のように書けます。

```swift
final class UserViewState: ObservableObject {
    private let queue: DispatchQueue = .init(label: ...)
    ...
    @Published private var _user: User?
    func user(_ handler: @escaping (User?) -> Void) {
        queue.async { [self] in
            handler(_user)
        }
    }
    
    func loadUser() {
        fetchUser(for: userID) { [self] user in
            queue.async {
                do {
                    _user = try user.get()
                } catch {
                    // エラーハンドリング
                }
            }
        }
    }
}
```

`loadUser` メソッドが呼ばれると `fetchUser` 関数を呼び出して非同期的に状態を更新します。また、 Case 16 と同様に `user` には `queue` を介してアクセスする必要があるため、 `user` はコールバック関数で値を返す形になっています。

これを利用する `UserViewController` のコードは次のようになります。

まず、 `viewDidApperar` から `loadUser` メソッドを呼び出して、サーバーからデータを取得させます。

```swift
final class UserViewController: UIViewController {
    private let state: UserViewState
    ...
    override func viewDidAppear(_ animated: Bool) {
        super.viewDidAppear(animated)
        state.loadUser()
    }
}
```

そして、 `state` の変更を `objectWillChange` を購読して監視し、サーバーから取得された `user` の情報を `nameLabel` に反映します。 `user` はメインスレッド**でない**スレッドから返ってくるかもしれないので、 `DispatchQueue.main` を使ってメインスレッド上で View への反映を行っています。

```swift
final class UserViewController: UIViewController {
    private let state: UserViewState
    ...
    override func viewDidLoad() {
        super.viewDidLoad()
        ...
        state
            .objectWillChange
            .sink { [weak self] _ in
                guard let self = self else { return }
                // state を View に反映する処理
                self.state.user { user in
                    DispatchQueue.main.async {
                        self.nameLabel.text = user?.name
                    }
                }
            }
            .store(in: &cancellables)
    }
    ...
}
```

### After

`actor` を使うと `UserViewState` は次のように書けます。 Before と比べるとずいぶんと簡潔です。これは、明示的に `queue` を扱う必要がないのと、 `user` を同期・非同期の二重化する必要がないからです。

```swift
actor UserViewState: ObservableObject {
    let userID: User.ID
    @Published var user: User?
    ...
    func loadUser() async {
        do {
            user = try await fetchUser(for: userID)
        } catch {
            // エラーハンドリング
        }
    }
}
```

利用側のコードには `await` が必要なことに注意して下さい。 Case 7 と同じように `Task` を使います。

```swift
final class UserViewController: UIViewController {
    private let state: UserViewState
    ...
    override func viewDidAppear(_ animated: Bool) {
        super.viewDidAppear(animated)
        Task {
            await state.loadUser()
        }
    }
}
```

また、変更を View に反映するコードは次のようになります。

```swift
final class UserViewController: UIViewController {
    private let state: UserViewState
    ...
    override func viewDidLoad() {
        super.viewDidLoad()
        ...
        state
            .objectWillChange
            .receive(on: DispatchQueue.main)
            .sink { [weak self] _ in
                guard let self = self else { return }
                // state を View に反映する処理
                Task {
                    self.nameLabel.text = await state.user?.name
                }
            }
            .store(in: &cancellables)
    }
    ...
}
```

このようにして `actor` を使って ViewModel をデータ競合から守ることができました。しかし、これに関してはより良い方法を Case 20 で紹介します。

**参考文献**

- [SE-0306: Actors](https://github.com/apple/swift-evolution/blob/main/proposals/0306-actors.md)
- [SE-0304: Structured concurrency](https://github.com/apple/swift-evolution/blob/main/proposals/0304-structured-concurrency.md)
- [Protect mutable state with Swift actors (WWDC 2021)](https://developer.apple.com/videos/play/wwdc2021/10133/)
- [Explore structured concurrency in Swift (WWDC 2021)](https://developer.apple.com/videos/play/wwdc2021/10134/)

## 💼 Case 19 (`Sendable`, `@Sendable`, `@unchecked`): Actor Boundary を越える

:::message
この Case には Before がありません。
:::

### After

Case 18 では前置きなく `actor` から `User` を取り出しましたが、これはそれほど単純な話ではありません。

取り出した `User` が次のような `struct` であれば問題ありません。

```swift
struct User {
    let id: ID
    var name: String
    var age: Int
    ...
}
```

しかし、クラスだとどうでしょうか。

```swift
final class User {
    let id: ID
    var name: String
    var age: Int
    ...
}
```

このとき、 `actor` から取り出した `User` インスタンスは、 `actor` の内外で共有されることになります。すると、 `actor` から取り出した `user` に変更を加えた場合、その変更は `actor` の内部にも影響を及ぼします。この変更は `actor` に守られていないので、データ競合を引き起こす可能性があります。そのため、コンパイラは次のようなコードをコンパイルエラーとします。

```swift
let user = await state.user // ⛔ コンパイルエラー
user?.age += 1 // actor の外から変更
```

しかし、クラスは絶対にダメというわけではありません。もし `User` がイミュータブルクラスであれば、 `actor` の内外で共有されてもデータ競合を引き起こすことはありません。

```swift
final class User {
    let id: ID
    let name: String
    let age: Int
    ...
}
```

```swift
let user = await state.user // ✅ データ競合の原因にならない
user?.age += 1 // ⛔ 変更できない
```

上記は `actor` から値を取得する例でしたが、外部から `actor` に値を渡す場合にも同じことが言えます。

では、コンパイラはどのようにして `actor` 内外のやりとりを許可するか判断するのでしょうか。

そのために導入されたのが [`Sendable`](https://developer.apple.com/documentation/swift/sendable) プロトコルです。 `Sendable` に適合する型は `actor` の内外でやりとりすることができます。 `Int` や `String` などの型はすべて `Sendable` に適合しており、そのため `Counter` から安全に `count` を取り出すことができました。

`actor` から `User` を取り出せるようにするためには、 `User` を `Sendable` に適合させる必要があります。

```swift
struct User: Sendable {
    let id: ID
    var name: String
    var age: Int
    ...
}
```

:::message
`public` でない値型の場合、暗黙的に `Sendable` に適合します。そのため、上記の例では明示的に `Sendable` に適合させる必要はありません。

本記事のコードは記述が冗長にならないように `public` を省略しています。通常、 `User` のような Entity は `public` で宣言される（アプリ本体とは別モジュールで宣言される）ことが多いと思います。そのような場合には上記のように `Sendable` に適合させることが必要です。
:::

しかし、どんな型でも `Sendable` に適合できるわけではありません。たとえば、 `var` プロパティを持つクラスは `Sendable` に適合することができません。

```swift
// ⛔ var プロパティを持つのでコンパイルエラー
final class User: Sendable {
    let id: ID
    var name: String
    var age: Int
    ...
}
```

`let` プロパティしか持たないイミュータブルクラスは `Sendable` に適合できます。

```swift
// ✅ let プロパティしか持たないので OK
final class User: Sendable {
    let id: ID
    let name: String
    let age: Int
    ...
}
```

ただし、 `struct` の場合でもイミュータブルクラスの場合でも、 `Sendable` に適合するにはすべてのプロパティの型が `Sendable` に適合している必要があります。これは、 `Codable` に適合するためにすべてのプロパティが `Codable` でなければならないのと似ています。

たとえば、 `Sendable` でない `NSString` をプロパティに持つと `Sendable` に適合することはできません。

```swift
// ⛔ Sendable でない型のプロパティを持つのでコンパイルエラー
final class User: Sendable {
    let id: ID
    let name: NSString // ⛔ Sendable でない
    let age: Int
    ...
}
```

Computed Property は問題ありません。次の例では `var name: String` を持ちますが、これはイミュータビリティに影響を与えません。

```swift
// ✅ Computed Property を持っていてもイミュータブルなので OK
final class User: Sendable {
    let id: ID
    let firstName: String
    let familyName: String
    let age: Int
    
    var name: String { // ✅ イミュータビリティに影響しない
        "\(firstName) \(familyName)"
    }
    ...
}
```

では、上記の `User` を改良して、次のような型を作るとどうなるでしょうか。毎回 `"\(firstName) \(familyName)"` で `name` を生成する計算コストを軽減するために、初回アクセス時に生成した文字列を `_name` にキャッシュするようにします。

```swift
// ⛔ セマンティクス上はイミュータブルなのにコンパイルエラーになってしまう
final class User: Sendable {
    let id: ID
    let firstName: String
    let familyName: String
    let age: Int
    
    private var _name: String? // ⛔ var プロパティ
    var name: String {
        // 本当は同時にアクセスされても大丈夫なように守る必要あり
        if _name == nil {
            _name = "\(firstName) \(familyName)"
        }
        return _name!
    }
    ...
}
```

このとき、セマンティクス上は `User` のイミュータビリティは崩れていませんが、 `var` プロパティを持つために `User` は `Sendable` に適合できなくなってしまいます。

このように、 `Sendable` として取り扱っても安全とわかっている型を、強制的に `Sendable` に適合させたいときもあります。それには `@unchecked` を使います。

```swift
// ✅ @unchecked を使って無理やり Sendable に適合させる
final class User: @unchecked Sendable {
    let id: ID
    let firstName: String
    let familyName: String
    let age: Int
    
    private var _name: String? // ✅ @unchecked なので OK
    var name: String {
        // 本当は同時にアクセスされても大丈夫なように守る必要あり
        if _name == nil {
            _name = "\(firstName) \(familyName)"
        }
        return _name!
    }
    ...
}
```

この `@unchecked` は、たとえばロックを使ってデータ競合から守られている型を `Sendable` に適合させる場合などにも役立ちます。

その他に `Sendable` に適合しているものとして `actor` が挙げられます。 `actor` はキューに守られているので、 `actor` の内外で別の `actor` のインスタンスをやりとりしてもデータ競合を引き起こしません。そのため、 `actor` は自動的に `Sendable` に適合します。

プロトコルに適合させられないものもあります。たとえば、 `actor` が高階メソッドを実装したいとします。しかし、関数やクロージャはプロトコルに適合させることができません。そのため、 `Sendable` に適合できず、そのままでは高階メソッドの引数として渡すことができません。

```swift
actor Foo {
    func bar(_ f: (String) -> Int) { // ⛔ (String) -> Int は Sendable でない
        // ...
    }
}
```

関数やクロージャが `Sendable` に適合していることを表すためには、 `@Sendable` が利用できます。

```swift
actor Foo {
    func bar(_ f: @Sendable (String) -> Int) { // ✅　@Sendable クロージャ
        // ...
    }
}
```

**参考文献**

- [SE-0302: `Sendable` and `@Sendable` closures](https://github.com/apple/swift-evolution/blob/main/proposals/0302-concurrent-value-and-concurrent-closures.md)
- [SE-0306: Actors](https://github.com/apple/swift-evolution/blob/main/proposals/0306-actors.md)
- [Protect mutable state with Swift actors (WWDC 2021)](https://developer.apple.com/videos/play/wwdc2021/10133/)

## 💼 Case 20 (`MainActor`, `@MainActor`): 共有された状態の変更（メインスレッド上での処理）

View に変更を加える場合、 iOS アプリ開発ではメインスレッド上で処理を実行する必要があります。そのため、 ViewModel の状態変更がすべてメインスレッドで行われると便利です。

Case 18 では ViewModel が個別のキューを保持・利用していました。これをメインキューにできれば、状態変更はすべてメインスレッド上で実行されることになります。

### Before

Case 18 の `UserViewState` の `queue` を、 `DispatchQueue.main` に変更するだけで目的は達成できます。

```swift
final class UserViewState: ObservableObject {
    private let queue: DispatchQueue = .main
    ...
}
```

### After

`actor` の場合はどうすれば良いでしょうか。 `actor` のキューはインスタンスごとに自動的に作られるため、メインキューに差し替えることができません。

そのような場合に利用できるのが [Global Actor](https://github.com/apple/swift-evolution/blob/main/proposals/0316-global-actors.md) です。 Global Actor は名前の通りグローバルに一つのキューを持つ Actor です。複数のクラスやインスタンスを同じキューで守ったり（ isolate したり）、型を丸ごとではなく個別のメソッド単位で isolate することが可能です。

標準ライブラリが提供する、最も代表的な Global Actor が `MainActor` です。 `MainActor` はキューとして `DispatchQueue.main` を利用します。

:::message
正確には、 Case 14 でも述べたように、 Actor が保持するのは `DispatchQueue` ではなく serial executor です。 `MainActor` は `DispatchQueue.main` をラップした custom executor を利用します。
:::

`UserViewState` を `MainActor` で丸ごと isolate するためには、次のように書きます。

```swift
@MainActor
final class UserViewState: ObservableObject {
    ...
}
```

Case 18 では `actor` だったところを `final class` に変更し、 `@MainActor` を付与しました。これによって、 `UserViewState` のすべてのメンバーが `MainActor` によって isolate されます。

さて、変更をメインスレッド上で行いたいクラスの代表格として、 `UIViewController` が思い浮かびます。 `UIViewController` に `@MainActor` が付与されていたら便利そうだと思いませんか？ [`UIViewController` の API リファレンス](https://developer.apple.com/documentation/uikit/uiviewcontroller)を見てみると、なんと `UIViewController` に `@MainActor` を付与する変更が加えられています。

```swift
@MainActor class UIViewController : UIResponder
```

これによって、（ `UIViweController` を継承した） `UserViewController` と `UserViewState` は同じ Actor の内部ということになります。 Case 15 で見たように、同一 Actor 内であればメソッドを同期的に呼び出すことができました。そのため、 `loadUser` メソッドの呼び出しは `await` 不要ということになります。

```swift
final class UserViewController: UIViewController {
    private let state: UserViewState
    ...
    override func viewDidAppear(_ animated: Bool) {
        super.viewDidAppear(animated)
        state.loadUser()
    }
}
```

`await` が不要になったため、 `Task` のイニシャライザに渡す必要もなくなり、コードがすっきりしました。

`state` への変更を View に反映する箇所も同様です。

```swift
final class UserViewController: UIViewController {
    private let state: UserViewState
    ...
    override func viewDidLoad() {
        super.viewDidLoad()
        ...
        state
            .objectWillChange
            .receive(on: DispatchQueue.main)
            .sink { [weak self] _ in
                guard let self = self else { return }
                // state を View に反映する処理
                self.nameLabel.text = state.user?.name
            }
            .store(in: &cancellables)
    }
    ...
}
```

UIKit ではなく SwiftUI を使う場合も同様です。 `@StateObject` はメインスレッドで更新されなければならないため、 `@MainActor` を付与する必要があります。上記の `UserViewState` クラスはそのような実装になっているので、そのまま `@StateObject` として利用可能です。

```swift
struct UserView: View {
    @StateObject private var state: UserViewState
    ...
    var body: some View {
        VStack {
            if let user = state.user {
                Text(user.name)
            }
            ...
        }
        .onAppear { state.loadUser() }
    }
}
```

**参考文献**

- [SE-0316: Global actors](https://github.com/apple/swift-evolution/blob/main/proposals/0316-global-actors.md)
- [SE-0306: Actors](https://github.com/apple/swift-evolution/blob/main/proposals/0306-actors.md)
- [Protect mutable state with Swift actors (WWDC 2021)](https://developer.apple.com/videos/play/wwdc2021/10133/)
