---
title: "Swift Concurrency チートシート"
emoji: "🌟"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["swift"]
published: true
---

Swift 5.5 で Swift に Concurrency （並行処理）関連の言語機能が追加されました。これによって、 Swift で**非同期処理・並行処理のコードをより簡潔かつ安全に**書くことができるようになります。

しかし、 Swift Concurrency は [Structured Concurrency](https://github.com/apple/swift-evolution/blob/main/proposals/0304-structured-concurrency.md) や [Actor](https://github.com/apple/swift-evolution/blob/main/proposals/0306-actors.md) など、 iOS / macOS アプリ開発者にとって馴染みの薄い概念を含みます。具体例を通して効率よく Swift Concurrency を習得できるように、本記事は、 Swift Concurrency 導入以前（ **Before** ）と導入後（ **After** ）で何がどのように変わるのか、具体的なコードの形で紹介します。

なお、 Swift Concurrency 関連の機能は次の三つに大別できるため、本記事の Before & After の例もそのように分類します。

- `async` / `await`
- Structured Concurrency
- Actor

:::message
本記事の説明は前から読んで理解できるように書かれていますが、チートシートとして、やりたいことベースでコードの書き方を調べたい場合は目次をご活用下さい。
:::

# `async` / `await`

`async` / `await` は Swift Concurrency の一部ですが、 `async` / `await` 自体が並行処理を扱うわけではありません。 `async` / `await` は非同期処理に関する機能です。

## 💼 Case 1: 非同期関数の利用（エラーハンドリングがない場合）

非同期処理はエラーハンドリングを必要とするケースが多いですが、まずは理解しやすいようにエラーハンドリングを伴わない例を考えてみます。

### Before

非同期関数はこれまで、実行結果を非同期に受け取るために、主に completion ハンドラーなどのコールバック関数を用いて書かれてきました。

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

戻り値として結果を受け取るため、クロージャ式はどこにも現れません。 `downloadData` はデータをダウンロードする関数なので、その結果を戻り値として返すのは自然な形です。

`throws` が付与された関数を呼び出す場合に `try` を書く必要があるのと同様に、 `async` が付与された関数を呼び出すときには `await` を書くことが必須となります。

:::message
この `downloadData` 関数は普通の関数と似た見た目をしていますが、非同期的に実行されることに注意が必要です。 `downloadData` を呼び出すと処理が suspend （中断）され、非同期的に結果が得られてから続きが resume （再開）されます。

さらに言うと、次のようなケースで `print("A")` と `print("B")` は同じスレッドで実行される保証すらありません。

```swift
print("A")
let data = await downloadData(from: url)
print("B") // "A" と同じスレッドとは限らない
```

`downloadData` の呼び出し後にどのようなスレッドで処理が再開されるかは `downloadData` の実装次第です。

このことは、コールバック関数を用いて書かれた次のコードで、 `print("A")` と `print("B")` が同じスレッドで実行されるとは限らないことと同じです。

```swift
print("A")
downloadData(from: url) { data in
    print("B") // "A" と同じスレッドとは限らない
}
```
:::

## 💼 Case 2: 非同期関数の実装（エラーハンドリングがない場合）

Case 1 では非同期関数を利用するコードを扱いましたが、次は自分で非同期関数を実装する場合を考えます。

### Before

API を叩いて JSON を取得し、それをデコードする関数を考えます。

ユーザーの JSON を取得・デコードして返す非同期関数 `fetchUser` は、 Case 1 の `downloadData` 関数を用いて次のように書けます。ユーザーの ID を受け取り、 completion ハンドラーで `User` を返します。

```swift
func fetchUser(for id: User.ID, completion: @escaping (User) -> Void) {
    let url: URL = .init(string: "https://koherent.org/fake-service/api/user?id=\(id)")!
    
    downloadData(from: url) { data in
        let user = try! JSONDecoder().decode(User.self, from: data)
        completion(user)
    }
}
```

`downloadData` の completion ハンドラーとして渡したクロージャ式の中で JSON を受け取ってデコードし、得られた結果を `fetchUser` が受け取った completion ハンドラーに渡します。

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

completion ハンドラーを使わずに戻り値として結果を返せば良いため、デコードされた `user` を単純に `return` します。

## 💼 Case 3: 非同期関数の利用（エラーハンドリングがある場合）

これまでの例ではエラーハンドリングを無視してきましたが、多くの非同期処理はエラーを引き起こす可能性があります。エラーハンドリングと非同期処理がどのように両立されてきたかを見てみます。

### Before

Case 1 の `downloadData` 関数がエラーを引き起こす場合を考えます。結果として `Data` か `Error` かを受け取る必要があるため、 completion ハンドラーの引数の型が `Result<Data, Error>` となっています。

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
`result` のエラーハンドリングは `switch` 文を使ってパターンマッチングで行うこともできます。筆者は、エラーハンドリング専用の `catch` を使った方が可読性が高いと思うので、このような書き方をしています。
:::

### After

`async` / `await` を使うと `downloadData` 関数は次のように宣言されます。

```swift
func downloadData(from url: URL) async throws -> Data
```

エラーを返す可能性があるため、 `async` に加えて `throws` が付与されています。 `async throws` はこの順である必要があります。

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
`try await` が `async throws` と順番がひっくり返っていることが（ `async` に `await` が、 `throws` に `try` が対応するなら `await try` でないかと）気になるかもしれません。この順番に関しては、すべてが宣言の逆になっていると覚えると覚えやすいです。

たとえば、 `downloadData` の宣言時

```swift
func downloadData(from url: URL) async throws -> Data
```

には `downloadData`, `async`, `throws`, `Data` の順ですが、利用時

```swift
let data = try await downloadData(from: url)
```

には `data`, `try`, `await`, `downloadData` と順番が反転します。

理論的には、 `throws` が `Result` を返すのと同等と考えるのと同じように、 `async` が `Future` を返すのと同等だと考えると、非同期かつエラーになりうる関数の戻り値は `Future<Result<Foo>>` であってほしいと考えられます。もし `Result<Future<Foo>>` だと同期的にエラーであるかが決まってしまうため、非同期的なエラーを扱うことができません。非同期的に得られる結果（ `Future` ）が成功か失敗か（ `Result` ）でなければならないため、 `Future` が外側である必要があります。これに対応させると `async throws` という順になります。

`Future<Result<Foo>>` を剥がして `Foo` を取り出すときには、まず `Future` を剥がしてから `Result` を剥がさなければなりません。そのため、先に `await` するために `try (await foo)` である必要があり、 `try await` の順となります。
:::

## 💼 Case 4: 非同期関数の実装（エラーハンドリングがある場合）

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

`downloadData` が `result` を受け取るため、これを `do` - `try` - `catch` でエラーハンドリングして、エラーを `catch` した場合にはそれを completion ハンドラーに渡します。

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

これは、 `downloadData` や `JSONDecoder.decode` の `try` を、 `fetchUser` の `throws` で受けているためです。わざわざ `catch` して `throw` しなおさなくても、 `fetchUser` には `throws` が付与されているのでそのままエラーを投げることができ、 `catch` して completion ハンドラーにリレーしている Before と比べてコードが簡潔になっています。

Swift には Swift 2.0 の頃から、言語の提供するエラーハンドリング機構として `throws` / `try` がありました。しかし、エラーハンドリングを必要とすることの多い非同期処理では、コールバック関数を挟むため `throws` / `try` を上手く活用することができませんでした。 `async` / `throws` は上記の例のように `trhows` / `try` と相性がよく、 `async` / `await` が導入されたことで `throws` / `try` も活用しやすくなります。

## 💼 Case 5: 非同期関数の連結

次は二つの非同期処理を連結する場合についてです。

ここでは、例としてユーザーのアイコンを取得する関数 `fetchUserIcon` を考えます。ただし、アイコンの URL は ユーザーの JSON の中に書かれており、

1. JSON をダウンロードしてデコードし、アイコンの URL を取得
2. アイコンの URL にアクセスしてダウンロード

という 2 段階の非同期処理を行う必要があるとします。

### Before

この 2 段階の非同期処理をコールバック関数を用いて書くと次のようになります。

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

複雑なコードになっていますが、ポイントはコールバック関数がネストしていることです。

最初に JSON をダウンロードするために `downloadData` 関数を呼び出し、結果をクロージャ式で受けています `(1)` 。そして、その JSON をデコードした後に得られた `iconURL` から、再び `downloadData` 関数でアイコンをダウンロードします `(2)` 。

この二つの非同期処理結果をそれぞれコールバック関数で受け取るため、コールバック関数がネストします。このように、非同期処理を連結するとその度にコールバック関数がネストし、いわゆるコールバック地獄と呼ばれる状態を作り出してしまいます。

:::message
コールバック地獄の解消には `Future` が用いられることもあります。 Combine を導入して `Future` と `flatMap` を用いればネストを解消することができます。

しかし、 `async` / `await` の方が簡潔でパフォーマンスもよく、 Swift Concurrency が使えるなら、あえて `Future` を使うべきケースは限定的でしょう（どうしても `Publisher` として扱いたい場合など）。
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

コールバック関数を用いた場合と異なり、非常に簡潔です。 JSON をダウンロードし、デコードし、得られた URL からアイコンをダウンロードするという 3 段階の処理が、そのまま 3 行のコードに現れています。

今、やりたいことの本質はこの三つのはずです。逆に言えば、 Before の複雑なコードのほとんどは、本質的なオペレーションとは関係のない、非同期処理を扱うためのボイラープレートだと言えます。

Before のコードが複雑になっている原因の一つは、それぞれのコールバック関数で別々にエラーハンドリングが行われていることです。この二つの `do` - `try` - `catch` を一つにまとめることはできません。 `do` - `try` - `catch` がコールバックのクロージャ式を越えることができないからです。一方で、 `async` / `await` を使うと、いくつの非同期処理を連結しても、すべての `try` を `throws` に引き受けさせることができます。これが、 `async` / `await` を利用するコードが著しく簡潔になっている理由です。

さらに、 Before のコードでは何箇所にも分散して `completion` の呼び出しを書かなければなりません。たとえば、もしエラーを `catch` したときにログ出力だけして `completion(.failure(error))` を忘れてしまうと、その非同期関数は永久に結果を返さない関数になってしまいます。 `async` 関数の場合、戻り値か `throws` で結果を返すため、そのようなことは決して起こりません。戻り値を `return` しなければコンパイルエラーになりますし、エラーが発生した場合にはそれを `throw` する以外に脱出する手段が存在しないからです。

つまり `async` / `await` を使うと、コードが簡潔になるだけでなく、安全にもなるということです（さらに言えば、パフォーマンスも向上します）。

## 💼 Case 6: コールバックから `async` への変換

:::message
この Case には Before がありません。
:::

### After

これまでの Case で `async` / `await` が優れているのはわかりましたが、現実問題としてすべてがすぐに `async` / `await` になるわけではありません。たとえば、利用しているサードパーティライブラリの非同期 API がコールバックのままになっているかもしれません。

そのような場合でも、コールバック版をラップして `async` 版を自力実装することができれば、アプリケーションコードは `async` / `await` で記述することができます。そんなときに活躍するのが Continuation です。

次の、コールバック版の `downloadData` 関数を使って、 `async` 版の `downloadData` 関数を実装します。

```swift
// コールバック
func downloadData(from url: URL,
    completion: @escaping (Result<Data, Error>) -> Void)
```

`async` 版の `downloadData` 関数は Continuation を使って次のように実装できます。

```swift
// async
func downloadData(from url: URL) async throws -> Data {
    try await withCheckedThrowingContinuation { continuation in
        downloadData(from: url) { result in
            do {
                let data = try result.get()
                continuation.resume(returning: data)
            } catch {
                continuation.resume(throwing: error)
            }
        }
    }
}
```

`async` 版の `downloadData` 関数の中でコールバック版の `downloadData` 関数を呼び出しても、その結果はコールバック関数の引数としてしか受け取れません。なんとかそれを `async` 版の戻り値として `return` しなければ（もしくは、エラーの場合は `throw` しなければ）なりません。

この変換を行ってくれるのが、新しく標準ライブラリに追加された `withCheckedThrowingContinuation` 関数です。 `withCheckedThrowingContinuation` 関数に渡されたクロージャは、引数として `continuation` を受け取ります。 `continuation` の `resume(returning:)` メソッドに結果を渡せばそれを `withCheckedThrowingContinuation` の戻り値として `return` してくれますし、 `resume(throwing:)` メソッドにエラーを渡せばエラーとして `throw` してくれます。

なお、今回のようにコールバック関数が `Result` を受け取っている場合には、 `conitnuation` の `resume(with:)` メソッドが役立ちます。結果とエラーをそれぞれ `resume(returning:)` と `resume(throwing:)` に渡す代わりに、 `resume(with:)` メソッドは直接 `Result` を受け取ります。

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

また、 `withUnsafeContinuation` ・ `withUnsafeThrowingContinuation` という関数もあります。　`withChecked(Throwing)Continuation` では、複数回 `resume` したり、 `resume` することなく `continuation` が破棄された場合に実行時エラーを引き起こしますが、 `withUnsafe(Throwing)Continuation` ではそのようなケースで未定義動作となります（チェックしない分、わずかにパフォーマンスが向上します）。
:::

# Structured Concurrency

ここからは Structured Concurrency に分類される例を紹介します。

## 💼 Case 7: 非同期処理の開始

Concurrency と言いながら、 Case 7 ではまだ非同期処理を扱います。ここでは、同期関数から非同期関数を呼び出すケースを扱います。

例として、ユーザー情報を表示する View Controller 、 `UserViewController` を考えます。 `viewDidAppear` で Case 4 の `fetchUser` 関数を呼び出し、ユーザー情報を取得、画面上に表示します。

### Before

従来のコールバックによる非同期関数を呼び出す場合、特に難しいところはありません。単に `viewDidAppear` から `fetchUser` を呼び出すだけです。

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

`async` / `await` には、 `async` 関数は `async` 関数の中でしか呼び出せないという制約があります。もし、（ `async` でない）普通の関数の中で `async` 関数を呼び出すとコンパイルエラーになります。これは `throws` が付与された関数は `throws` が付与された関数の中でしか呼び出せないのと同じです。

`throws` の場合は `do` - `try` - `catch` でハンドリングすることで、 `throws` が付与されて**いない**関数の中で呼び出すことができました。しかし `async` には `catch` に相当するものがありません。

では、 `veiwDidAppear` から `fetchUser` を呼び出すにはどうすれば良いでしょうか。 `veiwDidAppear` は `async` メソッドでないため、 `veiwDidAppear` の中で `async` な `fetchUser` 関数を呼び出すことはできません。

そんなときに役立つのが `Task` です。 `Task` を使うと、次のようにして、 `veiwDidAppear` の中から `async` な `fetchUser` 関数を呼び出すことができます。

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

わかりづらいですが、この `Task { }` は `Task` のイニシャライザの呼び出しです。 `{ }` は Trailing Closure なので、 `Task({ })` と書いているのと同じです。つまり、 `Task` のイニシャライザに、引数としてクロージャ式を渡しているということです。

この `Task` のイニシャライザが受け取るクロージャの型が `@escaping () async -> Success` となっており、 `async` が付与されているので `{ }` の中で `async` 関数を呼び出すことができます。そのため、 `fetchUser` も問題なく呼び出せます。

`Task` はイニシャライズされると即座に渡されたクロージャを実行しますが、その完了を待たずにイニシャライザは終了します。そのため、上記の例で `Task { }` がメインスレッドをブロックすることはありません。 `fetchUser` の結果を待たずに `viewDidAppear` は完了します。

このように、 `Task` を使って同期関数から `async` 関数を開始することができます。 `viewDidAppear` の例を取り上げましたが、 `@IBAction` や（ iOS 14 からの） [`UIControl` の `addAction(_:for:)`](https://developer.apple.com/documentation/uikit/uicontrol/3600490-addaction) から `async` 関数を呼びたいときも同様です。

ここで重要なのが、**すべての `async` 関数は必ず `Task` の上で実行される**ということです。 `async` 関数は `async` 関数からしか呼び出せないので、コールスタックを遡ると必ず同期関数から `async` 関数を呼び出す入口が必要となります。そこで `Task` が用いられるため、すべての `async` 関数は `Task` に紐付けられ、 `Task` 上で実行されます。

:::message
`Task` のイニシャライザに渡すクロージャには `@escaping` が付与されていますが、 `nameLabel.text = user.name` では明示的に `self.` を付けることなく `UserViewController` のプロパティにアクセスできています。これは、 [Implicit `self`](https://github.com/apple/swift-evolution/blob/main/proposals/0304-structured-concurrency.md#implicit-self) と呼ばれるもので、 `Task` のイニシャライザなどいくつかの API で採用されています。

[標準ライブラリのコード](https://github.com/apple/swift/blob/main/stdlib/public/Concurrency/Task.swift)上では、現状で

```swift
@_implicitSelfCapture operation: __owned @Sendable @escaping () async -> Success
```

のように `@_implicitSelfCapture` という隠し属性を使って実現されています。

これは、 `Task` のイニシャライザに渡されるクロージャはリファレンスサイクルを作らないからです。このクロージャは実行が完了すれば解法されるため、ここでキャプチャされた `self` がリファレンスサイクルを作り、メモリリークにつながる恐れはありません。そのような理由で Implicit `self` が用いられています。
:::

## 💼 Case 9: 並行処理（固定個数の場合）

ようやく並行処理です。並行処理は、複数の処理を同時に実行するための仕組みです。

ここでは、例として大小二つのアイコンを並行してダウンロードすることを考えます。 SNS などでは用途に応じて解像度の異なる複数のユーザーアイコン画像が用意されている場合があります。 small と large 、二つのアイコンを並行してダウンロードする関数 `fetchUserIcons` を考えてみましょう。

| small アイコン | large アイコン |
|---|---|
| ![](/images/swift-concurrency-cheatsheet/icon.png =60x) | ![](/images/swift-concurrency-cheatsheet/icon.png =180x) |

## Before

Case 4 などで見てきたコールバック版の `downloadData` 関数を使って、 `fetchUserIcons` 関数を実装します。

並行処理自体は特に難しくありません。 `downloadData` 関数を二つ並べれば並行に実行されます。問題は二つの処理が終わるのを待ち合わせて completion ハンドラーを呼び出さなければならない点です。ここでは `DispatchGroup` を使って待ち合わせを実現しています（これは Before のコードで、本記事の主題ではないため、 `DispatchGroup` についての説明は省略します）。

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

## After

同じく、 Case 4 の `async` 版の `downloadData` 関数を使って `fetchUserIcons` 関数を実装します。

次の「よくない例」のようなコードでは並行処理を実現できません。 small と large の二つのアイコンをダウンロードするために `downloadData` 関数を 2 回呼び出していますが、それぞれ `await` しているので small → large の順番にダウンロードすることになってしまいます。

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

`async let` Binding を利用すると、次のように Before と比べてはるかに簡潔に並行処理のコードを書くことができます。

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

`(1)` で `async let smallIcon = ...` のように `async let` で定数宣言を行い、右辺で `async` 関数を呼び出すことを `async let` Binding と呼びます。このとき、 `downloadData(from: smallURL)` に `await` は（ `try` も）必要なく、関数を呼び出すと結果を待たずに即時 `(2)` に進みます。

`(2)` も同様で、 `downloadData(from: largeURL)` の関数呼び出し後、結果を待たずに即時 `(3)` に進みます。

そして、 `async let` で宣言された定数を `(3)` で使おうとしたときに、初めて `await` が（ `try` も）必要になります。このようにして `await` を利用時まで遅らせることで、並行に関数を実行させることができます。

`async let` にはもう一つ特筆すべき特徴があります。それは、 `async let` で宣言された定数は、そのスコープを抜ける前に必ず `await` されなければならないということです。 `async let` のまま `return` するなど、スコープの外に持ち出すことはできません。これによって、 `async let` で開始された非同期処理は、必ずその呼び出し元の `async` 関数よりも先に完了することが保証されています。このように非同期関数のライフタイムがスコープで構造化されていることが、 **Structured Concurrency** という名前の由来です。

なお、 `async let` で宣言された定数をスコープ内で `await` しなかった場合、スコープの抜ける前に自動的に `await` されます。

Case 7 ですべての `async` 関数は Task の上で実行されると述べましたが、一つの Task は同時に一つの処理しか実行できません。 `async let` で呼び出される `async` 関数は並行に実行される必要があるため、同じ Task 上では実行することができません。このとき、元の Task から派生した Child Task が作られます。

構造化によって Child Task は必ず親の Task より先に終わることが保証されています（ `async let` で宣言された定数は必ずスコープを抜ける前に `await` されるため）。 Child Task が親の Task とは独立に生き続けることはありません。この親子関係は木構造で表すことができます。

![](/images/swift-concurrency-cheatsheet/task-tree.png)

Child Task がさらに `async let` を用いるなどした場合、上図のように孫 Task が作られることになります。この場合でも構造化は生きており、必ず Task Tree の末端（葉）から順に終了することが保証されます。

:::message
Structured Programming は構造化プログラミング（ Structured Programming ）から着想を得ています。構造化プログラミング以前は、 GOTO を使うなどして制御フローが複雑に絡み合ったコードを書くことが可能でした。構造化プログラミングによって、コードの構造化によってそのような状況が改善されました。

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

## 💼 Case 10: 並行処理（可変個数の場合）

次は可変個数の並行処理を考えます。 `async let` Binding を使えば固定個数の並行処理は簡単に記述できましたが、可変個数の場合は `async let` Binding が使えません。

ここでは例として、 N 人のユーザーのアイコンをダウンロードする関数を考えます。ユーザー ID の `Array` を渡して、それらのユーザーのアイコンを並行してダウンロードします。

### Before

Before については Case 9 と大差ありません。次のコードでは、 Case 9 同様 `DispatchGroup` を使って、並行に実行された非同期処理の待ち合わせをしています。

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

この例では `ids` が `Array` で渡されます。そのため、これを `async let` に書き下すことができません。

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

`(2)` で `addTask` メソッドを使って、 `group` に Child Task を追加します。 `async let` の場合と同様に、それらの Child Task は並行に実行されます。 `addTask` のクロージャの中の `(3)` で `await` をしていますが、 `addTask` 自体は `await` しておらず、このクロージャは並行して実行されるため、 `ids` のループは個々の `id` に対するダウンロードの結果を待たずにイテレートすることができます。

そのように並行で Child Task を実行した後で、 `group` から Child Task の結果を取り出すときに `await` する必要があります（ `(4)` ）。これは、 `async let` で宣言された定数を利用するときに `await` が必要なのとよく似ています。このような 2 段階の処理を行うことで、並行に処理を実行し、後で待ち合わせて結果を取得することが可能となっています。

`(4)` についてもう一つ注目すべきは、 `for` 文で `group` から結果を取り出すときに `for try await` と書かれていることです。 Swift Concurrency で追加された `AsyncSequence` に対して `for` 文を使うときには、 `for await` または `for try await` で値を受け取る必要があります。 `for await` （ `for try await` ）を使うことで、非同期的に得られる値を待ちながら順番に取り出すことができます。

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

## 💼 Case 11: 非同期処理のキャンセル（非同期 API の利用側）

非同期処理のキャンセルを正しく行うのは大変です。特に、非同期処理が別の非同期処理に依存している場合には、一連の非同期処理をすべて正しくキャンセルするには注意深くコードを書く必要があります。 Structured Concurrency は非同期処理のキャンセルを簡単にしてくれます。

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

上記のように、戻り値で canceller が返され、それを使ってキャンセルを実行します。また、キャンセルされたことをハンドリングできるように、 completion ハンドラーとは別にキャンセルをハンドリングするためのコールバック関数（上記の例では `cancellation` ）を渡す仕様になっていることもあるでしょう。

ダウンロードボタンを押すとダウンロードを開始し、キャンセルボタンを押すとダウンロードをキャンセルする例を考えてみます。次のように `@IBAction` のメソッドの中で `downloadData` を呼び出し、 `canceller` を `ViewController` のプロパティに保存しておきます。

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

`async` 関数の場合、同じ手は使えません。戻り値は結果を返すために使われるからです。また、仮に戻り値で `(Data, DownloadCanceller)` のように結果と canceller をペアで返したとしても、それが得られるのは `await` で suspend してから resume された後です。 canceller を取得できるのは非同期処理が完了した後ということになり、意味がありません。

`async` 関数をキャンセルするには `Task` を利用します。 Case 7 では `Task` のイニシャライザを使って非同期処理を開始するだけで、イニシャライズされた `Task` インスタンスは使いませんでした。その `Task` インスタンスには `cancel` メソッドが用意されており、 `cancel` メソッドを呼ぶことで Task をキャンセルすることができます。

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

キャンセルでもエラーが発生した場合でも、処理を中断することに違いはありません。それらのハンドリングに区別が要らない場合は単に `catch` すればいいですが、ときにはキャンセルとエラーを区別したい場合もあります。

たとえば今回のケースでは、エラーが起こったらアラートを表示して、ユーザーにそれを知らせるのが親切でしょう。しかし、キャンセルボタンが押された場合にはユーザーの意思によるものなので、アラートを表示する必要はありません。このような場合、キャンセルかどうかを区別して分岐する必要があります。このような分岐は `Task.isCancelled` を調べることで実現できます。

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

しかし、キャンセル時に本当に `CancellationError` が `throw` されるかは実装依存であり、上記の方法は確実ではありません。まとめて `catch` してから `Task.isCancelled` で分岐することをおすすめします。
:::

**参考文献**

- [SE-0304: Structured concurrency](https://github.com/apple/swift-evolution/blob/main/proposals/0304-structured-concurrency.md)
- [Explore structured concurrency in Swift (WWDC 2021)](https://developer.apple.com/videos/play/wwdc2021/10134/)

## 💼 Case 12: 非同期処理のキャンセル（非同期 API の実装側①）

:::message
この Case には Before がありません。
:::

### After

Case 11 は `async` 関数を呼び出す側で、どのようにキャンセルを実行するかの例でした。キャンセルが実行された場合に、自作の `async` 関数を正しく中断させるには注意が必要です。

多くの場合、 `async` 関数は他の既存の `async` 関数を組み合わせて実装されることになります。その場合、それらの `async` 関数が正しくキャンセル処理を行っていれば、自作の `async` 関数で何もしなくてもキャンセルが実行されます。

しかし、自分でキャンセル処理を実装しないといけない場合もあります。非同期処理の一部として同期的に重めの処理を行っている場合、その途中で `Task` がキャンセルされていないかをチェックして、キャンセルされていた場合には `CancellationError` を `throw` する必要があります。

ここでは例として、動画像処理で動画の中に何人の歩行者が写っているか数える関数 `countPedestrians` を考えてみます。

`countPedestrians` は動画を受け取り、動画から 1 フレームずつ画像を取り出し、それらを画像処理して歩行者をカウントします。とても重い処理なので非同期関数として実装されるのが適しています。また、 `Task` がキャンセルされた場合には途中で処理を中断したいです。Task がキャンセルされたかは `Task.checkCancellation()` でチェックすることができます。 `checkCancellation` メソッドは Task がキャンセルされていた場合、 `CancellationError` を `throw` します。

これを使うと、 `countPedestrians` 関数は次のように実装できます。

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

`output.copyNextSampleBuffer()` メソッドで動画から 1 フレームずつ画像を取り出し、処理を実行します。これを `while` ループで繰り返しますが、毎フレーム最初に `checkCancellation` を呼び出すことで、 `Task` がキャンセルされると処理を中断します。

しかし、単にキャンセルするだけでなく、クリーンアップ処理などキャンセルに付随した処理が必要となる場合もあります。そのような場合には `Task.isCancelled` で分岐し、明示的に `CancellationError` を `throw` します。

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

## 💼 Case 13: 非同期処理のキャンセル（非同期 API の実装側②）

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

これは、並行処理の安全性のために、 `var` で宣言された `canceller` を `onCancel` でキャプチャすることができないためです。

同様の問題は Proposal のコードにも存在し、これもコンパイルすることができません。無理やりなワークアラウンドで回避することは可能ですが、良い解決法が見つかるまで Proposal に沿った上記のコードのままとしておきたいと思います。
:::

**参考文献**

- [SE-0304: Structured concurrency](https://github.com/apple/swift-evolution/blob/main/proposals/0304-structured-concurrency.md)
- [Explore structured concurrency in Swift (WWDC 2021)](https://developer.apple.com/videos/play/wwdc2021/10134/)

# 3️⃣ `actor`

## 💼 Case 14: 共有された状態の変更

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

しかし、現実には両方 2 を返すこともあります。なぜなら、今 `increment` メソッドは同時に呼び出されるので、両者の `count += 1` が呼ばれた後でそれぞれ `return count` されることがあり得るからです。この場合、 2 回インクリメントされた後の値が `return` されるので、両方とも 2 を返すことになります。これがデータ競合です。

データ競合を防ぐには `increment` メソッドが同時に実行されないようにする必要があります。

### Before

データ競合を防ぐための古典的な方法はロックです。しかし、ロックはスレッドをブロックしてしまうためパフォーマンスがよくないですし、メインスレッドをブロックするような場合には UI がフリーズする原因になります。また、（カウンターのような単純な処理はともかく）複雑な並行処理をロックで制御しようとするとデッドロックの危険が高まります。

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

`Counter` に `queue` を持たせ `increment` メソッドを丸ごと `queue` 上で実行することで、オペレーションが同時に実行されることを防いでいます。

ただし、オペレーションを一度 `queue` に入れてから実行するということは、結果は非同期的に得られることになります。そのため、 completion ハンドラーを用いて結果を返します。

### After

Swift Concurrency で導入された `actor` を用いれば、より簡潔に安全な `Counter` を実装することができます。

`actor` はクラスと同じような参照型の型を宣言します。 `actor` を使うと Before と同様の安全な `Counter` は次のように書けます。

```swift
actor Counter {
    private var count: Int = 0

    func increment() -> Int {
        count += 1
        return count
    }
}
```

このコードは最初の安全**でない** `Counter` クラスのコードとほとんど同じです。異なるのは、 `final class` が `actor` になった点だけです。

`actor` はインスタンスごとに内部にキューを持ち（ Actor 用語では mailbox と呼びます）、 `actor` のメソッドを呼び出すと、そのオペレーション（ Actor 用語では message と呼びます）は自動的にキューに追加され、非同期的に実行されます。そのため、上記の `increment` メソッドは必ず同時に一つしか実行されないことが保証されます。

つまり、 `actor` は Before で書いたような処理を自動的に行なってくれると言えます。 `increment` メソッドは非同期的に実行されるため、外部から呼び出す場合には `async` メソッドに見えます。

```swift
// 外部から見た increment メソッド
func increment() async -> Int
```

そのため、 `increment` の呼び出しには `await` が必要です。

```swift
let counter: Counter = .init()

Task {
    print(await counter.increment()) // 1 or 2
}

Task {
    print(await counter.increment()) // 2 or 1
}
```

**参考文献**

- [SE-0306: Actors](https://github.com/apple/swift-evolution/blob/main/proposals/0306-actors.md)
- [Protect mutable state with Swift actors (WWDC 2021)](https://developer.apple.com/videos/play/wwdc2021/10133/)

## 💼 Case 15: 共有された状態の変更（インスタンス内でのメソッド呼び出し）

Case 14 の `increment` メソッドは外部からしか呼ばれていませんが、同じインスタンス内部から `Counter` のメソッドを呼ぶケースを考えてみます。

ここでは、 2 回カウンターをインクリメントする `incrementTwice` メソッドを考えます。すでに `increment` メソッドがあるので、これを使って `incrementTwice` メソッドを実装します。

### Before

`DispatchQueue` & コールバック関数による `Counter` の場合、単純に考えると次のように `incrementTwice` メソッドを実装してしまいます。しかし、これは競合状態を引き起こします。

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

一つの `Counter` に対して同時に `incrementTwice` メソッドを呼び出した場合、期待する結果は片方が 2 を、もう片方が 4 を返すことです。しかし、 `increment` メソッドのオペレーションは `queue` 上で実行されますが、 2 回のインクリメントはばらばらに `queue` に追加されます。そのため、 2 回インクリメントする途中の状態が外部から観測される可能性があります。前述の例では、 2 と 4 ではなく 3 と 4 が返される場合があります。

これを防ぐには、 2 回のインクリメントを同期的に実行する必要があります。そのために、同期的なインクリメントを行う `_increment` メソッドを実装し、**非**同期的な `increment` メソッドは `queue` 上で `_increment` を呼び出す形にします。また、 `incrementTwice` は `queue` 上で同期的に `_increment` メソッドを 2 回呼び出すようにします。

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

このようにすれば、 3 のような途中の状態が外部から観測されるのを防ぐことができます。 `_increment` メソッドは `private` になっていることに注意して下さい。 `Counter` には外部からは `queue` を通してアクセスしなければならないため、 `queue` を介さない `_increment` メソッドを公開するわけにはいきません。

`increment` メソッドや `incrementTwice` メソッドは `_increment` メソッドを呼び出していますが、一度 `queue` に乗せてしまえば同時に実行されることはないので、安心して `_increment` メソッドを呼び出すことができます。 `queue` は同時に一つのオペレーションしか実行せず、同期的なオペレーションは分断されることがないので、 3 のような途中状態が観測されることはありません。

このように、外部からは非同期に、内部からは一度 `queue` に乗ってしまえば同期的に扱えるようにすることで、データ競合や競合状態を防ぐことができます。しかし極論すれば、すべてのメソッドに同期版と非同期版を用意し、非同期版は `queue` 上で同期版を呼び出すような二重化が必要ということです。

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

`actor` のメソッドは外部からは `async` に見えましたが、インスタンス内部からはただの同期メソッドに見えます。これは、インスタンス内部ではすでにオペレーションがキュー上で実行されており、改めてキューに乗せる必要がないからです。そのため、 `incrementTwice` から `increment` メソッドを呼び出す際に `await` は不要です。

これは、 Before で実現したかったこと（外部からは非同期（ `async` ）に、内部からは同期に）を自動的に実現しているということです。コードの二重化なしにこれを実現できるのが、 `actor` の最も重要な機能です。

**参考文献**

- [SE-0306: Actors](https://github.com/apple/swift-evolution/blob/main/proposals/0306-actors.md)
- [Protect mutable state with Swift actors (WWDC 2021)](https://developer.apple.com/videos/play/wwdc2021/10133/)
