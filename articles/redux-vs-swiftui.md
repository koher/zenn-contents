---
title: "ReduxとSwiftUIの類似性をAIに示してもらった"
emoji: "🤖"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["redux", "swiftui", "react", "swift", "chatgpt"]
published: true
---

**React×Redux**と**SwiftUI**によるアプリケーション開発は、共に**Single Source of Truth**を掲げ、**宣言的な仮想DOM/View**を用いるなど、類似点が多く見られます。両者はの対象はそれぞれWebフロントエンド・iOSアプリと異なりますが、両者の対応をコードで示すことができれば、より抽象的なレベルで概念を理解する助けになりそうです。

そこで、今話題のAIである[ChatGPT](https://openai.com/blog/chatgpt/)に、両者を比較するコードを書いてもらいました。いきなり想定したような形にはならなかったので、対話を重ねて徐々に修正する形で最終形にたどり着きました。**対話によって軌道修正できるのが素晴らしいですね。**

![](/images/redux-vs-swiftui/chatgpt.png)

以下、AIとの会話をそのまま掲載します。🧑が僕の発言で、🤖がAIの発言です。**ここまで意思疎通ができることに驚きました。**

:::message
ただし、この形式で挟まれるメモは、AIとの会話ではなく僕が後から追記したものです。それ以外はコードブロックで囲むなど見た目を整えただけで、会話をそのまま掲載しています。
:::

# AIとの対話

## 🧑

ReduxとSwiftUIは似ていますが、両者の対応をコードの例を挙げて説明して下さい。

---

## 🤖

ReduxとSwiftUIは完全に異なるものです。Reduxは、Webフロントエンド開発でよく使われるJavaScriptのライブラリであり、アプリケーションの状態を管理するためのツールです。一方、SwiftUIは、AppleのiOS、macOS、tvOS、およびwatchOSプラットフォーム向けのUIフレームワークです。両者の間には直接的な対応はありません。

:::message
依頼がざっくりしすぎていて、意図を汲み取ってもらえませんでした。
:::

---

## 🧑

両者はSingle Source of Truthを基本とした状態管理を行う点で似ています。

---

## 🤖

そうですね。Reduxは、アプリケーションの状態を単一の管理するための「単一の真実」と呼ばれる概念を基本としています。SwiftUIでは、ほとんど同じような概念があります。SwiftUIでは、View Hierarchyと呼ばれる仕組みを使用して、UIを定義します。このView Hierarchyは、構成されるすべてのViewが含まれる単一の入力から始まります。そして、そのView Hierarchyの中では、状態を管理するために、`@State`プロパティラッパーを使用します。

:::message
こちらの意図を少し補足すると、どういう観点での類似性かを理解し、軌道修正してくれたのは驚きでした。人間との会話でもこのようなステップを経るかもしれません。
:::

具体的な例としては、以下のようになります。

```js
// Redux
const initialState = {
  count: 0
};

function reducer(state = initialState, action) {
  switch (action.type) {
    case 'increment':
      return { count: state.count + 1 };
    case 'decrement':
      return { count: state.count - 1 };
    default:
      return state;
  }
}
```

```swift
// SwiftUI
struct ContentView: View {
  @State var count = 0

  var body: some View {
    VStack {
      Button("Increment") {
        self.count += 1
      }
      Button("Decrement") {
        self.count -= 1
      }
      Text("Count: \(count)")
    }
  }
}
```

Reduxでは、reducer関数が「単一の真実」となる状態を管理する役割を果たします。一方、SwiftUIでは、`@State`プロパティラッパーがView Hierarchyの中で状態を管理する役割を果たします。

また、Reduxでは、アクションを通じて状態を更新しますが、SwiftUIでは、View Hierarchyが変化することによって状態が更新されます。

ちなみに、`@State`プロパティラッパーを使用して状態を管理することは、SwiftUIでのみ可能です。Reduxのようなライブラリを使用しない限り、`@State`プロパティラッパーを使用することはできません。

---

## 🧑

Reduxの場合の例は、Reactを使ったものにしてもらえますか？

:::message
Reduxの方の例にはUIが含まれていなかったので、SwiftUIとそろえるためにReactでUIを書いてもらいました。
:::

---

## 🤖

もちろんです。Reduxを使用するには、Reactやその他のJavaScriptフレームワークと組み合わせることができます。例えば、以下のようになります。

```js
import React from 'react';
import { createStore } from 'redux';

// Redux
const initialState = {
  count: 0
};

function reducer(state = initialState, action) {
  switch (action.type) {
    case 'increment':
      return { count: state.count + 1 };
    case 'decrement':
      return { count: state.count - 1 };
    default:
      return state;
  }
}

const store = createStore(reducer);

// React
class Counter extends React.Component {
  state = {
    count: store.getState().count
  };

  componentDidMount() {
    store.subscribe(() => {
      this.setState({
        count: store.getState().count
      });
    });
  }

  increment = () => {
    store.dispatch({ type: 'increment' });
  };

  decrement = () => {
    store.dispatch({ type: 'decrement' });
  };

  render() {
    return (
      <div>
        <button onClick={this.increment}>Increment</button>
        <button onClick={this.decrement}>Decrement</button>
        <div>Count: {this.state.count}</div>
      </div>
    );
  }
}
```

上記の例では、Reduxを使用して状態を管理し、Reactを使用してUIを定義しています。Reduxでは、`reducer`関数が「単一の真実」となる状態を管理する役割を果たし、Reactでは、`Counter`コンポーネントがUIを定義する役割を果たしています。

この例では、Reduxの`store`オブジェクトに状態を保存し、`store.getState()`を使用してその状態を取得し、`store.dispatch()`を使用してアクションを定義し、`store.subscribe()`を使用して状態が更新されたときにUIを更新するように設定しています。

ReduxとReactを組み合わせることで、状態をより簡単に管理できるようになります。また、Reduxのような状態管理ライブラリを

:::message
個人的にChatGPTの一番残念なところですが、ときどきこのような形で返答が途切れてしまうことがあります。
:::

---

## 🧑

SwiftUIの例は、`@StateObject`を使う形にして下さい。

:::message
SwiftUIの方の例は、UIのコードと状態管理が一体化してしまっていたので、`ObservableObject`に分離することを期待して修正をお願いしました。
:::

---

## 🤖

もちろんです。SwiftUIでも、`@State`プロパティラッパーを使用して状態を管理することができます。また、`@StateObject`プロパティラッパーを使用することで、アプリケーション内で共有される状態を管理することができます。

例えば、次のようになります。

```swift
struct Counter {
  @State var count: Int = 0
}

struct ContentView: View {
  @StateObject var counter = Counter()

  var body: some View {
    VStack {
      Button("Increment") {
        self.counter.count += 1
      }
      Button("Decrement") {
        self.counter.count -= 1
      }
      Text("Count: \(self.counter.count)")
    }
  }
}
```

上記の例では、`Counter`という名前のデータ型を定義しています。このデータ型は、`@State`プロパティラッパーを使用して状態を管理します。そして、`ContentView`という名前の`View`を定義しています。この`View`は、`@StateObject`プロパティラッパーを使用して`Counter`データ型のインスタンスを保持します。

これにより、`ContentView`が持つ`@StateObject`プロパティである`counter`は、`ContentView`のサブビューからアクセスできるようになります。`counter`プロパティを使用して、`count`プロパティを更新することで、状態を更新することができます。

`@StateObject`プロパティラッパーを使用することで、アプリケーション内で共有される状態を管理することができます。これにより、状態をより簡単に管理できるようになります。

:::message
AIの書いたこのコードは正しくありません。`Counter`は`ObservableObject`に準拠していないですし、クラスでもないので`@StateObject`として扱うことはできません。

この返答については、AIは本当に理解して受け答えしているのではなく、パターンを当てはめてそれっぽい回答を作っているだけではないかと感じました。
:::

---

## 🧑

`@StateObject`は、`ObservableObject`に準拠したクラスに対してしか使えません。`Counter`構造体に対して`@StateObject`を使うとコンパイルエラーになります。

---

## 🤖

その通りです。`@StateObject`プロパティラッパーは、`ObservableObject`に準拠したクラスに対してのみ使用することができます。つまり、次のようなコードでは、コンパイルエラーになります。

```swift
struct Counter {
  @State var count: Int = 0
}

struct ContentView: View {
  @StateObject var counter = Counter() // Compile error: 'Counter' does not conform to protocol 'ObservableObject'
  
  // ...
}
```

この場合は、`Counter`構造体を`ObservableObject`プロトコルに準拠させることで、`@StateObject`プロパティラッパーを使用することができます。例えば、次のようになります。

```swift
class Counter: ObservableObject {
  @Published var count: Int = 0
}

struct ContentView: View {
  @StateObject var counter = Counter()

  // ...
}
```

上記の例では、`Counter`クラスを`ObservableObject`プロトコルに準拠させることで、`@StateObject`プロパティラッパーを使用することができるようになりました。

`@StateObject`プロパティラッパーを使用する場合は、必ず`ObservableObject`に準拠したクラスを使用する必要があることに注意してください。そうしないと、コンパイルエラーになる可能性があります。

:::message
このやりとりはおもしろかったです。僕の指摘に対して、AIは自分の間違いを理解し修正することができました。

色々話してみて感じるのは、ChatGPTは膨大な知識を持っているけれども、それらを自分で検証していなさそうだということです。しかし、こちらから誤りを指摘すると、それを他の知識と照らし合わせて検証し、誤りを訂正できることがあります。

↓は別の会話ですが、それをより明確に感じたやりとりです。一度、指摘に対して反論してきましたが、詳しく説明すると自らの誤りを認めて訂正することができました。錯覚かもしれませんが、知性のようなものを感じてしまいます。

![](/images/redux-vs-swiftui/ai-recovery.png)
:::

---

## 🧑

Reduxの例に対応するように、`Counter`クラスに`increment`メソッドと`decrement`メソッドを追加して下さい。

---

## 🤖

もちろんです。以下のように、`Counter`クラスに`increment`メソッドと`decrement`メソッドを追加することで、Reduxの例に対応することができます。

```swift
class Counter: ObservableObject {
  @Published var count: Int = 0

  func increment() {
    self.count += 1
  }

  func decrement() {
    self.count -= 1
  }
}

struct ContentView: View {
  @StateObject var counter = Counter()

  var body: some View {
    VStack {
      Button("Increment") {
        self.counter.increment()
      }
      Button("Decrement") {
        self.counter.decrement()
      }
      Text("Count: \(self.counter.count)")
    }
  }
}
```

上記の例では、`Counter`クラスに`increment`メソッドと`decrement`メソッドを追加しました。これにより、`ContentView`内で`counter.increment()`や`counter.decrement()`といった形で、`Counter`クラスの状態を更新することができるようになりました。

また、`@Published`プロパティラ

---

## 🧑

Reduxの方の例は、Redux Toolkitを使用してもらえますか？

---

## 🤖

もちろんです。Redux Toolkitを使用することで、Reduxをより簡単に使用することができます。例えば、以下のようになります。

```js
import { createSlice } from '@reduxjs/toolkit';

// Redux Toolkit
const initialState = {
  count: 0
};

const counterSlice = createSlice({
  name: 'counter',
  initialState,
  reducers: {
    increment(state) {
      state.count += 1;
    },
    decrement(state) {
      state.count -= 1;
    }
  }
});

const store = configureStore({
  reducer: counterSlice.reducer
});

// React
class Counter extends React.Component {
  state = {
    count: store.getState().count
  };

  componentDidMount() {
    store.subscribe(() => {
      this.setState({
        count: store.getState().count
      });
    });
  }

  increment = () => {
    store.dispatch(counterSlice.actions.increment());
  };

  decrement = () => {
    store.dispatch(counterSlice.actions.decrement());
  };

  render() {
    return (
      <div>
        <button onClick={this.increment}>Increment</button>
        <button onClick={this.decrement}>Decrement</button>
        <div>Count: {this.state.count}</div>
      </div>
    );
  }
}
```

上記の例では、Redux Toolkitを使用しています。Redux Toolkitを使用すると、`createSlice`関数を使用して、reducerとactionを同時に定義することができます。これにより、Reduxの設定をより簡潔にすることができます。

また、Redux Toolkitを使用することで、`configureStore`関数を使用して`store`を作成することができます。これにより、Reduxの`store`を作成するための設定をより簡単にすることができます。

上記の例では、`counterSlice`という名前の`createSlice`関数で定義されたスライスを使用しています。このスライスには、`increment`と`decrement`という2つのreducerが定

---

## 🧑

Reactでは、Function Componentを使うようにして下さい。

---

## 🤖

もちろんです。Function Componentを使用することで、Reactのコンポーネントをより簡潔に書くことができます。例えば、次のようになります。

```js
import { createSlice } from '@reduxjs/toolkit';
import { useSelector, useDispatch } from 'react-redux';

// Redux Toolkit
const initialState = {
  count: 0
};

const counterSlice = createSlice({
  name: 'counter',
  initialState,
  reducers: {
    increment(state) {
      state.count += 1;
    },
    decrement(state) {
      state.count -= 1;
    }
  }
});

const store = configureStore({
  reducer: counterSlice.reducer
});

// React
function Counter() {
  const count = useSelector((state) => state.count);
  const dispatch = useDispatch();

  function increment() {
    dispatch(counterSlice.actions.increment());
  }

  function decrement() {
    dispatch(counterSlice.actions.decrement());
  }

  return (
    <div>
      <button onClick={increment}>Increment</button>
      <button onClick={decrement}>Decrement</button>
      <div>Count: {count}</div>
    </div>
  );
}
```

上記の例では、`Counter`という名前の関数を定義しています。この関数をFunction Componentと呼びます。Function Componentを使用することで、Reactのコンポーネントをより簡潔に書くことができます。

また、Function Componentを使用する場合は、`useSelector`と`useDispatch`という2つの関数を使用することができます。`useSelector`関数を使用することで、Reduxの`store`から`state`を取得することができます。また、`useDispatch`関数を使用することで、Reduxの`store`に対してactionをdispatchすることができます。

上記の例では、`useSelector`関数を使用して`state`から`count`を取得しています。また、`useDispatch`関数を使用して`increment`と`decrement`という2つのactionをdispatchしています。これにより、Function Component内でReduxの`store`から`state`を取得したり、actionをdispatchすることができるようになります。

Function

:::message
このコードには`configureStore`を`import`できていないというミスがあります。そのようなミスから、やはりChatGPTは何かを理解しているのではなく、それっぽい返答を作り上げているだけなのではないかという疑念が生まれます。

一方で、たとえ人間がコードを書いてもこの程度のミスは十分起こり得るでしょう。ChatGPTがコードを実行したりコンパイルしたりできないことを考えると、十分にミスが少ないとも言えます。

`import`を修正し、`Provider`やrootを補ってみたところ、このコードは正しく動作しました。
:::

# まとめ

このようにして、AIと対話しながら、React×ReduxとSwiftUIの対応を示すコードサンプルを作り上げることができました。完成したコードを見比べると、**actionとreducerが`ObservableObject`メソッドに、Reactの仮想DOMがSwiftUIのView**に対応していることがわかります。

ChatGPTはまだまだ荒削りだとは感じますが、**対話を通して文脈を理解し、こちらの意図を汲んで修正を重ねてくれるのは本当に素晴らしいです**。これからさらに発展していくだろうことを考えると、今後欠かせないツールになるのは間違いないと思います。