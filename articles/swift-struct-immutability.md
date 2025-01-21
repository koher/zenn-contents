---
title: "Swiftのstructのイミュータビリティ"
emoji: "💎"
type: "tech"
topics: ["swift"]
published: false
---

```swift
struct Foo {
    var value: Int
}
```

この `struct` はプロパティ `value` が `var` で宣言されているので、一般的にはミュータブルな `struct` と呼ばれます。しかし、 **`struct` はミュータブルでも本質的にはイミュータブルクラスと同じ** ような性質を持ちます。このことを順を追って説明します。

# `struct` のミュータビリティ

`var` プロパティを持つ `struct` と `let` プロパティを持つ `struct` を比較し、 `struct` のミュータビリティ（インスタンスの状態を変更可能かどうか）について考えます。

## `var` プロパティを持つ `struct`

まず、先の `var` プロパティ `value` を持つ `struct Foo` を考えます。

```swift
struct Foo {
    var value: Int
}
```

次に、この `Foo` に `increment` メソッドを実装します。

```swift
extension Foo {
    mutating func increment() {
        value += 1
    }
}
```

`value += 1` するだけの何の変哲もないメソッドです。 ただし、 **`struct` のプロパティを変更するメソッドを宣言するには `mutating` キーワードが必要なため、 `mutating func` となっていることに注意** してください。

この `increment` メソッドは、次のように使うことができます。

```swift
var foo: Foo = .init(value: 0)
foo.increment()
print(foo.value) // 1
```

↑のコードでは `value` の初期値が `0` なので、 `increment` を呼び出すと `foo.value` は `1` に変化します。

## `let` プロパティを持つ `struct`

次に、 `value` を `var` から `let` に変えてみましょう。

```swift
struct Foo {
    let value: Int
}
```

この `struct` は、プロパティ `value` の値を変更できないため、一般的にはイミュータブルな　`struct` と呼ばれます。

もちろん、 `var value` のときと同じように `increment` を実装しようとするとエラーになります。

```swift
extension Foo {
    mutating func increment() {
        value += 1 // ⛔️ コンパイルエラー
    }
}
```

`value` は `let` で宣言されているので、 `value += 1` で変更することができません。

しかし、実は次のようにすると `increment` メソッドを実装することができます。

```swift
extension Foo {
    mutating func increment() {
        self = Foo(value: value + 1) // ✅ これならOK
    }
}
```

↑のやっていることは、 `value + 1` で新しい `Foo` インスタンスを作り、 `self` 自体をそれで上書きするということです。

どうしてこんなことができるのでしょうか？それを知るためには

- メソッドとは何か
- `mutating func` とは何か

の二つを知る必要があります。

## メソッドとは何か

ここで、次のようなメソッド `bar` を考えてみましょう。

```swift
extension Foo {
    func bar() -> Int {
        value * value
    }
}
```

これも、 `value` を2乗して `return` するだけのメソッドです。

:::message
↑のコードには `return` が書かれていませんが、Swiftでは関数のbodyに式が一つしか書かれていない場合、 `return` キーワードを省略することができます。
:::

このメソッドは次のようにして使うことができます。

```swift
let foo: Foo = .init(value: 3)
print(foo.bar()) // 9
```

`value` を `3` として初期化しているので、　`foo.bar()` の値は `3 * 3` で `9` になります。

さて、この `bar` メソッドは、丁寧に（明示的に `self.` を）書くと次のように書くことができます。

```swift
extension Foo {
    func bar() -> Int {
        self.value * self.value
    }
}
```

↑のように、メソッド内では `self` にアクセスすることができます。この `self` はどこから来たのでしょうか？

実は、 **メソッドとは、暗黙の第1引数 `self` を持つ関数のようなもの** です。

つまり、 `bar` メソッドは次のような `bar` 関数として書くことができます。

```swift
func bar(_ self: Foo) -> Int {
    self.value * self.value
}
```

:::message
ここでは引数名を `self` としていますが、この引数名は `foo` でも何でも構いません。メソッドの場合とコードを比較しやすいように `self` という引数名にしているだけです。

なお、Swiftでは引数名に `self` を使うことができるので、↑のコードはコンパイル可能です。
:::

この場合、第1引数に `Foo` インスタンスを渡して呼び出すことになります。

```swift
let foo: Foo = .init(value: 3)
print(bar(foo))  // 9
```

この **関数呼び出し `bar(foo)` と見比べると、メソッド呼び出し `foo.bar()` とは、 `bar` メソッドの暗黙の第1引数 `self` に `foo` を渡して `bar` メソッドを呼び出すこと** だと理解できます。

## `mutating func` とは何か

`var` プロパティを持つ `struct Foo` について、 `increment` メソッドは次のように書けました。

```swift
extension Foo {
    mutating func increment() {
        self.value += 1
    }
}
```

↑のコードでも、明示的に `self.` を追加してあります。

この `increment` メソッドは、 `value` の値を変更するので `mutating` キーワードがないとコンパイルエラーになります。

```swift
extension Foo {
    func increment() {
        self.value += 1 // ⛔️ コンパイルエラー
    }
}
```

なぜ、わざわざ `mutating` キーワードを付けないといけないのでしょうか？これは、 `increment` メソッドを関数の形で書き直してみるとわかります。

`increment` メソッドを、明示的に第1引数 `self` を持つ関数の形で書き換えると次のようになります。

```swift
func increment(_ self: Foo) {
    self.value += 1 // ⛔️ コンパイルエラー
}
```

`self` は引数として渡されますが、引数は定数（ `let` で宣言されている）扱いなので、その値を変更することはできません。そのため、 `self.value += 1` でエラーとなります。

これをコンパイル可能にするには、次のように `self` に `inout` を付与します。

```swift
func increment(_ self: inout Foo) {
    self.value += 1 // ✅ これならOK
}
```

これを、 `mutating` 付きで実装された `increment` メソッド（↓に再掲）と見比べてみましょう。

```swift
extension Foo {
    mutating func increment() {
        self.value += 1
    }
}
```

つまり **`mutating` とは、メソッドの持つ暗黙の第1引数 `self` に `inout` を付与するもの** だと理解することができます。

## （続） `let` プロパティを持つ `struct`

さて、ここで `let` プロパティ `value` を持つ `struct Foo` に話を戻しましょう。

たとえ `value` が `let` で宣言されていても、次のようにすれば `increment` メソッドを実装できるという話でした。

```swift
extension Foo {
    mutating func increment() {
        self = Foo(value: self.value + 1) // ✅ これならOK
    }
}
```

↑のコードでは `self` 自身を書き換えるような実装をしていますが、これがなぜ問題ないのか、このメソッドを関数の形で書き換えてみるとわかります。

```swift
func increment(_ self: inout Foo) {
    self = Foo(value: self.value + 1) // ✅ inout引数は変更可能
}
```

`inout` が付与された引数には値を代入して呼び出し元に反映させることができます。 `mutating func` では暗黙の第1引数 `self` に `inout` が付与されているので、 `self` 自体を書き換えることも可能ということです。

このように宣言された `increment` メソッドは、たとえ `value` プロパティが `let` で宣言されていたとしても、 `var` で宣言されていた場合とまったく同じように使うことができます。

```swift
var foo: Foo = .init(value: 0)
foo.increment() // ✅ valueがletでもOK
print(foo.value) // 1
```

`struct Foo` は `let` プロパティのみを持つのでイミュータブルだったはずなのに、まるでミュータブルな `struct` と同じように状態を変更できてしまいました。

## structのミュータビリティを決めるもの

`var value` のときと `let value` のときを比べてみると、 `Foo` のミュータビリティはプロパティを `var` で宣言するか `let` で宣言するかで決まらないと考えられます。では、 `struct` のミュータビリティを決めるものは何でしょうか？

**`struct` のミュータビリティを決めるのは、変数宣言時の `var` / `let`** です。

たとえ `Foo` の `value` プロパティが `var` で宣言されていても `let` で宣言されていても、次のコードはコンパイルが通ります。

```swift
var foo: Foo = .init(value: 0)
foo.increment() // ✅　fooがvarで宣言されているのでOK
print(foo.value) // 1
```

しかし、 `foo` が `let` で宣言されていると、どちらの場合でも `increment` メソッドの呼び出しはコンパイルエラーとなります。

```swift
let foo: Foo = .init(value: 0)
foo.increment() // ⛔️ fooがletで宣言されているのでエラー
print(foo.value)
```

このように、 `struct` のミュータビリティは変数宣言時に `var` を使うか `let` を使うかで決まります。これは値型の性質によるものです。

:::message
Swiftでは `let` で宣言された場合は変数ではなく定数と呼ぶことが多いですが、 `let` で宣言されたものも広義の変数として扱われることもあります。本記事では、 `var` / `let` どちらで宣言されていても変数と呼ぶものとします。
:::

クラスなどの参照型では、変数にはインスタンスのアドレスが格納され、インスタンスの実体は別の領域に存在します。しかし、値型では変数のために確保されたメモリ領域に直接インスタンスのデータが書き込まれます。言い換えると、 **値型のインスタンスは変数のために確保されたメモリ領域と一体** であると言えます。

そのため、変数のために確保されたメモリ領域が変更可能かどうか、つまり変数が `var` / `let` のどちらで宣言されているかが、インスタンスを変更可能かどうか（ミュータビリティ）を決めることになります。これは、参照型との大きな違いです。参照型（クラスなど）では、インスタンスと変数は独立に存在するため、プロパティが `var` / `let` のどちらで宣言されるかがインスタンスが変更可能かを決めます。

**値型のミュータビリティを参照型と同じように考えてはいけない** ということに注意してください。インスタンスと変数が一体である値型では、変数自体の可変性がインスタンスのミュータビリティを決めるのです。

# イミュータブルクラスとの比較

初めに **`struct` はミュータブルでも本質的にはイミュータブルクラスと同じ** ような性質を持つと言いましたが、それを説明するために今度はイミュータブルクラスについて考えてみます。

今度は `Foo` をクラスとして宣言します。

```swift
final class Foo {
    let value: Int
}
```

`value` プロパティが `let` で宣言されており、 `let` プロパティのみを持つので、この `Foo` クラスはイミュータブルクラスです。

:::message
プロパティに初期値を与えない場合、イニシャライザを明示的に実装しないとコンパイルエラーとなるので、↑のコードを実際に試す場合にはイニシャライザを追加してください。
:::

:::message
イミュータブルクラスは `final class` として実装するのが一般的です。

もし `final` でなければ、継承してサブクラスで `var` プロパティを追加することができます。そうすると、 `Foo` クラス自体のインスタンスはイミュータブルでも、 `Foo` 型の変数にはミュータブルなサブクラスのインスタンスも代入できてしまいます。これではイミュータブルクラスの型として扱いづらいため、イミュータブルクラスを実装する場合には `final` を付与し、継承を禁止します。
:::

このイミュータブルクラス `Foo` に、先ほどと同じように `mutating` を使って `increment` メソッドを実装できないか考えてみます。

```swift
extension Foo {
    // ⛔️ これはSwiftの構文の問題でできない
    mutating func increment() {
        self = Foo(value: self.value + 1)
    }
}
```

残念ながら、Swiftでは参照型のメソッドに `mutating` を付与することきません。そのため、↑はコンパイルエラーとなります。

しかし、これは本質的な問題ではなく、単にSwiftが構文上禁止しているだけです。 `increment` を関数の形で実装することはできます。

```swift
// ✅ これならOK
func increment(_ self: inout Foo) {
    self = Foo(value: self.value + 1)
}
```

この `increment` 関数は、次のように使うことができます。

```swift
var foo: Foo = .init(value: 0)
increment(&foo) // ✅ これならfooを変更可
print(foo.value) // 1
```

なんと、イミュータブルクラスでも `foo.value` の値を変更することができました。

もちろん、これはイミュータブルクラスのインスタンスの `value` を書き換えたわけではありません。 `increment(&foo)` は `foo = Foo(value: foo.value + 1)` したのと同じであり、新しいインスタンスを作って変数 `foo` に再代入しただけです。しかし、見かけ上は `struct` の場合と同じことが起こっています。

イミュータブルクラスでは、インスタンスの状態を変更することはできません。そのため、 **イミュータブルクラスの型の変数に書かれたアドレスは、その変数が表す状態と一体** であると言えます。

つまり、 **`struct` でもイミュータブルクラスでも、変数が表す状態を変更可能かどうかは、その変数が `var` / `let` のどちらで宣言されているかで決まる** ということになります。このことから、状態の可変性について、 **`struct` とイミュータブルクラスは本質的に同じ** ということができます。

# ミュータブルクラスとの比較

`struct` とイミュータブルクラスが似ていることを、ミュータブルクラスとの比較で見てみましょう。

今度は、次のようなミュータブルクラス `Foo` を考えます。

```swift
final class Foo {
    var value: Int
}
```

`value` プロパティが `var` で宣言されているのでミュータブルクラスです。

この `Foo` クラスの `increment` メソッドは次のように実装できます。

```swift
extension Foo {
    func increment() {
        value += 1
    }
}
```

`value` が `var` で宣言されているので、単純に `value += 1` と書くことができます。

この `increment` メソッドを使うコードは次のようになります。

```swift
let foo: Foo = .init(value: 0)
foo.increment()
print(foo.value) // 1
```

次に、 `foo.increment()` する前に、 `foo` を別の変数 `foo2` に代入すると、↓のような結果となります。

```swift
// Fooがミュータブルクラスの場合
let foo: Foo = .init(value: 0)
let foo2 = foo

foo.increment()

print(foo.value)  // 1
print(foo2.value) // 1
```

（ `Foo` は参照型で、変数 `foo` に格納されているのはインスタンスのアドレスなので） `foo2` に `foo` を代入した結果、 `foo` と `foo2` は同じインスタンスを参照することになります。そのため、 `increment` の結果、 `foo.value` だけでなく `foo2.value` も　`1` となります。

もし `Foo` が `struct` なら次の通りです。

```swift
// Fooがstructの場合
var foo: Foo = .init(value: 0)
var foo2 = foo

foo.increment()

print(foo.value)  // 1
print(foo2.value) // 0
```

（ `struct` は値型で、変数 `foo` に格納されているのはインスタンスそのものなので） `foo2` に `foo` を代入すると `foo` の内容が `foo2` にコピーされ、 `foo2` は `foo` のコピーである新しいインスタンスを表すことになります。そのため、 `increment` の結果は（コピーである） `foo2` には影響を与えません（ `foo2.value` が `0` のままとなります）。

`Foo` がイミュータブルクラスの場合は次の通りです。

```swift
// Fooがイミュータブルクラスの場合
var foo: Foo = .init(value: 0)
var foo2 = foo

increment(&foo)

print(foo.value)  // 1
print(foo2.value) // 0
```

この場合、 `foo2 = foo` の結果、 `foo` と `foo2` は同じインスタンスを参照することになりますが、その後、 `increment` メソッドで `foo` には新しいインスタンス（ `value` が `1` のもの）のアドレスが代入されます。その結果、 `foo2` だけが `value` が `0` の古いインスタンスを参照したままとなり、このような挙動となります。

`struct` の場合とイミュータブルクラスの場合は、背後で起こっていることは異なりますが、見かけ上は同じような挙動をします。言い換えると、 **`struct` の場合とイミュータブルクラスの場合では、その背後の実装詳細は異なるけれども、それらの表しているセマンティクスは同じ** であるとも言えます。

一方で、 **`struct` やイミュータブルクラスをミュータブルクラスと比較すると、ミュータブルクラスはまったく異なる挙動をする** ことがわかります。

# `struct` とイミュータブルクラスはshared mutable stateを作らない

↑の例では、ミュータブルクラスの場合には `foo.value` を変更すると `foo2.value` も変更されました。このように、複数箇所から参照され共有されている状態のことをshared mutable stateと呼びます。

一方で、 `struct` やイミュータブルクラスの場合には `foo.value` を変更しても `foo2.value` は変更されませんでした。 `struct` ではインスタンスが共有されないため（sharedでないため）、イミュータブルクラスではインスタンスを変更できないため（mutableでないため）、それらを用いてshared mutable stateを作ることはできません。

そのような、 `struct` とイミュータブルクラスに通じる共通の性質は、たとえば両者がSwift Concurrencyにおいて `Sendable` に準拠できるなど、その他の共通性も導きます。

```swift
final class User: Sendable { // ⛔️ コンパイルエラー
    var name: String
}

final class User: Sendable { // ✅ OK
    let name: String
}

struct User: Sendable { // ✅ OK
    var name: String
}
```

:::message
`Sendable` に準拠した型のインスタンスは、isolation boundaryを超えることができます。isolation boundaryを超えて複数のisolation domain間でインスタンスが共有されると、インスタンスに並行にアクセスすることが可能となります。そして、shared mutable stateに並行にアクセスされると、ロックなどで適切に保護されない限り、data raceを引き起こす可能性があります。

Swiftコンパイラは、コードがdata raceを引き起こさないことを保証するため、shared mutable stateを作る型が `Sendable` 準拠することを許しません。そのため、ミュータブルクラスを `Sendable` 準拠させようとするとコンパイルエラーとなります。
:::

# `struct` のイミュータビリティ

Swiftは値型中心の言語ですが、多くの言語は参照型中心です。それらの言語では、shared mutable stateのもたらす問題を回避するために、イミュータブルクラスが使われることが多いです。

しかし、イミュータブルクラスを用いると状態を変更するのが大変です。 `foo.value += 1` と `foo = Foo(value: foo.value + 1)` では、前者の方が簡潔でわかりやすいのは言うまでもないでしょう。

`struct` であれば、 `foo.value += 1` のように簡潔に書くことを可能でありながら、shared mutable stateを作りません。つまり、 **`struct` を使えばミュータブルでもイミュータブルクラス同様の利益を享受可能** なのです。

そして、Swiftは値型中心の言語なので、 `struct` をはじめとした値型を便利に使うための機能が充実しています。

:::message
たとえば、 `inout` 引数や `mutating func` 、値型のコレクション（ `Array`, `Set`, `Dictionary` ）、サブタイピングではなくパラメトリックポリモーフィズムで抽象化された標準ライブラリ、opaque typeなどが挙げられます。
:::

## まとめ

- `struct` は本質的にイミュータブルクラスと同じ
- `struct` を使えばミュータブルでもイミュータブルクラスと同じ利益を享受できる