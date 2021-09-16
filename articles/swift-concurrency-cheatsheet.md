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

# `async` / `await`

`async` / `await` は Swift Concurrency の一部ですが、 `async` / `await` 自体が並行処理を扱うわけではありません。 `async` / `await` は非同期処理に関する機能です。

## Case 1: 非同期関数の利用（エラーハンドリングがない場合）

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

## Case 2: 非同期関数の実装（エラーハンドリングがない場合）

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

## Case 3: 非同期関数の利用（エラーハンドリングがある場合）

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

## Case 4: 非同期関数の実装（エラーハンドリングがある場合）

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