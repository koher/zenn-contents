---
title: "SVVSとは何か"
emoji: "⚡️"
type: "tech"
topics: ["swift", "swiftui", "ios", "svvs"]
published: false
---

***SVVS***は**Store**・**View**・**ViewState**の略で、それら三つを中心にアプリケーションを構築するアーキテクチャです。

*SVVS*は非常にシンプルなアーキテクチャで、特に*SwiftUI*を使ってアプリ開発をしていれば、自然と*SVVS*と似た形になることも多いでしょう。*SVVS*は特別難しいことをしませんが、多くのケースでは十分実用的です。

先日、[iOSDC Japan 2023のトーク](https://fortee.jp/iosdc-japan-2023/proposal/edcec751-4f7f-46aa-9c17-92249a1a1771)で*SVVS*について発表がありました。本記事では、*SVVS*とは何か、その中身により踏み込んで解説します。

:::message
*SVVS*は、僕（@koher）が元々*SwiftUI*でやっていたことをベースに、ChatworkのiOSチームと話し合って作り上げたものです。*SVVS*という命名もChatworkのエンジニアの方によるものです。
:::

# なぜ*ViewState*や*Store*が必要なのか

すべてを*View*に記述する形から始めて、なぜ*ViewState*や*Store*が必要になるのか、順を追って説明します。

## 1️⃣ すべてを*View*に記述する場合

*SVVS*における*View*は、*SwiftUI*では `View` で実現されますが、すべての `View` が*SVVS*の*View*であるわけではありません。たとえば、再利用可能なコンポーネントなどは多くの場合＊SVVS*の*View*にあたりません。＊SVVS*の*View*は多くの場合、一つの画面を表す `View` で、自身の状態を持ちます。

例として、SNSのようなアプリでユーザーの情報を表示するページ `UserView` について考えます。 `UserView` は自分自身だけではなく、任意のユーザーの情報を閲覧できるページです。たとえば、Zennでは[僕（@koher）というユーザーの情報を閲覧するページ](https://zenn.dev/koher)（↓）があります。 `UserView` はこのようなページを表すものとします。

![](/images/what-is-svvs/zenn-koher.png)

`UserView` はIDを受け取り、APIを叩いてユーザー情報を取得・表示します。*SwiftUI*で書くと次のようになります（コードを簡潔にするために、ユーザーの名前だけを表示することとします）。

```swift
import SwiftUI

struct UserView: View {
    let id: User.ID

    @State private var user: User?
    @State private var isReloadButtonDisabled: Bool = false

    var body: some View {
        VStack {
            // ユーザー情報を表示
            Text(user?.name ?? "User Name")
                .font(.title)
                .redacted(reason: user == nil ? .placeholder : [])
            // Reloadボタンを押すとユーザー情報を再取得
            Button("Reload") {
                // ユーザー情報の取得中はボタンを無効化
                isReloadButtonDisabled = true
                Task {
                    defer { isReloadButtonDisabled = false }
                    do {
                        self.user = try await UserRepository.fetchValue(for: id)
                    } catch {
                        // エラーハンドリング
                    }
                }
            }
        }
        // ユーザー情報を取得
        .task {
            // ユーザー情報の取得中はボタンを無効化
            isReloadButtonDisabled = true
            defer { isReloadButtonDisabled = false }
            do {
                self.user = try await UserRepository.fetchValue(for: id)
            } catch {
                // エラーハンドリング
            }
        }
    }
}
```

:::message
APIを叩く部分については `UserRepository` にカプセル化され、隠蔽されているものとします。
:::

:::message
Reloadボタンが押されたときの処理について、 `Task { }` の外に `isReloadButtonDisabled = true` が書かれているのは、ボタンが押されたときに同期的にボタンを無効化するためです。これによって、ボタンが押されてから無効化されるまでの間に再度ボタンが押されてしまうことを防げます。
:::

このように、*View*にすべてを書こうとすると、*View*のロジック（初回ロードやReloadボタンを押したときのリロード、リロード中にReloadボタンを無効化する処理）とレイアウトのコードが混ざってしまい、コードの見通しが悪くなってしまいます。

## 2️⃣ *ViewState*に*View*の状態とロジックを分離する

これを解決するために、*ViewState*に*View*の状態とロジックを分離します。

```swift
import Combine

@MainActor
final class UserViewState: ObservableObject {
    let id: User.ID

    @Published private(set) var user: User?
    @Published private(set) var isReloadButtonDisabled: Bool = false

    init(id: User.ID) {
        self.id = id
    }

    // ユーザー情報を取得
    func load() async {
        // ユーザー情報の取得中はボタンを無効化
        isReloadButtonDisabled = true
        defer { isReloadButtonDisabled = false }
        do {
            self.user = try await UserRepository.fetchValue(for: id)
        } catch {
            // エラーハンドリング
        }
    }

    // ユーザー情報を再取得
    func reload() {
        // ユーザー情報の取得中はボタンを無効化
        isReloadButtonDisabled = true
        Task {
            defer { isReloadButtonDisabled = false }
            do {
                self.user = try await UserRepository.fetchValue(for: id)
            } catch {
                // エラーハンドリング
            }
        }
    }
}
```

さらに `load` と `reload` の重複部分を取り除くと次のようになります。

```swift
func reload() {
    // ユーザー情報の取得中はボタンを無効化
    isReloadButtonDisabled = true
    Task {
        await load()
    }
}
```

これを使えば、 `UserView` は次のように書けます。

```swift
import SwiftUI

struct UserView: View {
    @StateObject private var state: UserViewState

    init(id: User.ID) {
        self._state = .init(wrappedValue: .init(id: id))
    }

    var body: some View {
        VStack {
            // ユーザー情報を表示
            Text(state.user?.name ?? "User Name")
                .font(.title)
                .redacted(reason: state.user == nil ? .placeholder : [])
            // Reloadボタンを押すとユーザー情報を再取得
            Button("Reload") {
                state.reload()
            }
        }
        // ユーザー情報を取得
        .task {
            await state.load()
        }
    }
}
```

このように、***ViewState*を導入することで、*View*のロジックとレイアウトのコードを分離することができます。**

*ViewState*を実装する際には、次の点に注意してください。

- *ViewState*の `@Published` プロパティを `private(set)` にする
- *ViewState*のメソッドに `throws` を付与しない

### *ViewState*の `@Published` プロパティを `private(set)` にする

*ViewState*の `@Published` プロパティを `private(set)` とすることで、*View*は直接それらのプロパティを変更することができなくなります。*View*が状態を変更するには*ViewState*のメソッドを呼び出すしかなく、それによって*View*にロジックが記述されることを防止します。

ただし、値をセットする以上のロジックが存在しない場合は例外です。そのような場合は無駄に*setter*のメソッドを提供するのではなく、プロパティの `set` を公開した方が良いでしょう。特に、 `Binding` を使って他のコンポーネント（ `Toggle` や `TextField` など）に状態を渡したい場合は `set` を公開したいことも多いでしょう。そのような場合でも、 `set` に渡す値を計算するロジックを*View*に書いてしまわないように注意しましょう。

### *ViewState*のメソッドに `throws` を付与しない

*ViewState*のメソッドに `throws` を付与すると、それを呼び出す*View*側にエラーハンドリングのロジックが記述されることになります。それを防止するために、エラーハンドリングは*ViewState*内部で完結させ、その結果を*View*に反映する場合（エラーメッセージやアラートを表示するなど）は `@Published` プロパティの変更を通して行います。

たとえば、*ViewState*にエラーメッセージを表す `@Published` プロパティを追加、その値が `nil` でない場合に*View*がエラーメッセージを表示するなどが考えられます。

## 3️⃣ *View*のライフサイクルを超えて扱いたい状態を*Store*で管理する

*ViewState*を導入することでコードはクリーンになりましたが、今のままでは `UserView` を開くたびにユーザー情報を読み込むことになります。一度読み込んだ情報はキャッシュし、再表示時には即座にキャッシュの値が表示されるのが望ましいです（そして、キャッシュを表示している間に再取得を行います）。

*ViewState*のライフサイクルは*View*と一致しているため、画面間で共有されるような状態を扱えません。そこで、*View*のライフサイクルを超えて状態を管理する*Store*を導入します。

```swift
import Combine

@MainActor
final class UserStore {
    @Published private(set) var values: [User.ID: User] = [:]

    static let shared: UserStore = .init()
    private init() {}

    // ユーザー情報を取得
    func loadValue(for id: User.ID) async throws {
        if let value = try await UserRepository.fetchValue(for: id) {
            values[value.id] = value
        } else {
            values.removeValue(forKey: id)
        }
    }
}
```

そして、*ViewState*は自身の状態を*Store*に同期させる形にします。

```swift
import Combine

@MainActor
final class UserViewState: ObservableObject {
    let id: User.ID

    @Published private(set) var user: User?
    @Published private(set) var isReloadButtonDisabled: Bool = false

    init(id: User.ID) {
        self.id = id

        // 状態をStoreと同期する
        UserStore.shared.$values
            .map { $0[id] }
            .removeDuplicates()
            .assign(to: &$user)
    }

    func load() async {
        isReloadButtonDisabled = true
        defer { isReloadButtonDisabled = false }
        do {
            // Storeのメソッドを呼び出してStoreの状態を更新する
            try await UserStore.shared.loadValue(for: id)
        } catch {
            // エラーハンドリング
        }
    }

    func reload() {
        isReloadButtonDisabled = true
        Task {
            await load()
        }
    }
}
```

:::message
*Combine*の代わりにiOS 17からの[*Observation*](https://developer.apple.com/documentation/observation)を利用する場合は、*Store*の状態を*ViewState*に反映するコードは不要となります。 `user` は*Computed Property*として実装しておけば、自動的に*Store*の状態変更が*View*に反映されるでしょう。
:::

このように、***Store*を導入することで、*View*のライフサイクルを超えて状態を保持し続けることができます。**

*View*間で*Store*を通して状態を共有することで、Single Source of Truthが実現され、*View*間の状態のずれ（たとえば、一覧ページから詳細ページを開き、状態を更新してから一覧に戻ると更新内容が反映されていないなど）を防ぐことができます。

:::message
*Store*を `@MainActor` として実装するのは、*Store*は*View*に同期したい状態を保持するものだからです。

*Store*の状態は*Store*のメソッドを介して更新されますが、その実際の処理は別の型（上記の例では `UserRepository` ）に切り出されることが多く、*Store*がするのはその結果を自身の状態に反映するだけです。そのため、メインスレッドで重い処理が実行されることはなく、 `@MainActor` を付与しても問題とはなりません。
:::

*Store*を実装する際には、次の点に注意してください。

- *Store*のライフサイクルを適切に管理する
- *ViewState*側で値を反映するときは `removeDuplicates` する

### *Store*のライフサイクルを適切に管理する

上記の例では `UserStore` をシングルトンとして実装していますが、これは必須の要件ではありません。

この例では `UserStore` がアプリのライフサイクルで（アプリのプロセスが完了するまで）保持したい状態を扱うため、シングルトンとして実装しています。しかし、たとえばユーザーのログインからログアウトまでのライフサイクルで保持したい状態などもあります。そのような場合はシングルトンとせず、そのライフサイクルに応じて*Store*のインスタンスを生成・破棄する必要があります。

### *ViewState*側で値を反映するときは `removeDuplicates` する

`UserStore` はそれまでにロードしたすべての `User` を保持します。 `UserView` が今表示したい `User` に限定されません。

もし裏側で何らかの非同期処理（たとえば、定期的に `User` の最新の状態を受け取っているなど）が走っていると、そのとき `UserView` が表示しているのとは関係のない `User` が更新されることもあり得ます。

`UserViewState` は `UserStore.shared.$values` を通して変更を監視しているため、関係のない `User` が更新された場合もその情報を受け取ってしまいます。そのような場合に、*View*の `body` が無駄に実行されないように、 `removeDuplicates` を挟むことでその*View*に関係のない更新を無視します。

```swift
UserStore.shared.$values
    .map { $0[id] }
    .removeDuplicates() // 関係のないデータの更新を無視
    .assign(to: &$user)
```

# まとめ

これまで見てきたように、*SVVS*では*View*の状態とロジックを*ViewState*に分離し、*View*のライフサイクルを超えて扱いたい状態を*Store*に分離して管理します。これは、*SwiftUI*を使っていれば自然と実現されることであり、呼び方は異なっても、*ViewState*や*Store*に相当するものを実装しているケースは多いでしょう。*SVVS*は特別なことはしていませんが、*View*と*ViewState*・*Store*の責務を意識してコードを書くことで、クリーンで見通しの良い実装を実現することができます。

*SVVS*は難しいことをしませんが、多くのケースでは十分に機能します。逆に、*SVVS*程度の責務の分割もできていないようであれば、*SVVS*を意識するだけでよりクリーンなコードを実現することができるでしょう。