---
title: なぜXはフックではないのか?
date: '2019-01-26'
spoiler: できるというのは、すべきという意味ではない。
---

最初のアルファ・バージョンの[React Hooks](https://reactjs.org/hooks)がリリースされて以来、議論の場に何度もあがる疑問がある: 「なぜ*\<あるAPI\>*はフックではないのか？」

念のため、現在フック *である* 少数のものは以下の通りです:

* [`useState()`](https://reactjs.org/docs/hooks-reference.html#usestate) はstate 変数の宣言に用います。
* [`useEffect()`](https://reactjs.org/docs/hooks-reference.html#useeffect) は副作用を宣言するのに用います。
* [`useContext()`](https://reactjs.org/docs/hooks-reference.html#usecontext) はコンテキストの読み取りに用います。

ただ、 `React.memo()` や `<Context.Provider>` のように、フック*ではない* APIも存在します。一般的に提案されるそれらのフック・バージョンは*ノンコンポジショナル*であるか*アンチモジュラー*になりがちです。 この記事はその理由を説明するためのものです。

**注: このポストはAPIの議論に興味がある人のための深く掘り下げた内容です。Reactを用いて生産的であろうとするためだけなら、これらの内容を考える必要はありません！**

---

私たちがReact APIに維持させたい2つの重要な特性があります。

1. **コンポジション:** [カスタム・フック](https://reactjs.org/docs/hooks-custom.html) は私たちがフックAPIにexciteした理由の大きな部分を占めています。私たちは人々が頻繁に自分自身のフックをつくることを期待しており、別の人によって書かれたフックが互いに[コンフリクトしない](/why-do-hooks-rely-on-call-order/#flaw-4-the-diamond-problem) ようにする必要があります。（私たちはコンポーネントが綺麗に構成されて、互いを壊しあってないということを当たり前だと思ってしまっていませんか？）

2. **デバッギング:** 
私たちはアプリケーションが大きくなってもバグを [見つけやすい](/the-bug-o-notation/) 状態に保ちたいと思っています。Reactのもっとも優れた機能の一つは、何かおかしいものが描画された時に、間違いを起こしたプロップやステイトを見つけるまでツリーを上に登っていけるところです。

これら2つの制約を合わせることで、何がフックになり得て何が*なり得ない*のかの判断が可能です。いくつかの例を見てみましょう。

---

##  A Real Hook: `useState()`
##  実際のフック: `useState()`

### コンポジション

それぞれ`useState()`を呼んでいる複数のカスタムフックはコンフリトしません:

```jsx
function useMyCustomHook1() {
  const [value, setValue] = useState(0);
  // ここで起こることは、ここにとどまります。
}

function useMyCustomHook2() {
  const [value, setValue] = useState(0);
  // ここで起こることは、ここにとどまります。
}

function MyComponent() {
  useMyCustomHook1();
  useMyCustomHook2();
  // ...
}
```

`useState()`の条件無しの呼び出しを新たに追加することは常に安全です。
そのコンポーネントが新たなState変数を宣言するために使用しているその他のフックについては何も知る必要がありません。また、それらのうちの一つによって更新されている別のState変数を壊すこともできません。

**評決:** ✅ `useState()` は カスタム・フックを脆くしません。

### デバッギング

フックは、フック*間*で値を渡すことができるという点で便利です。


```jsx{4,12,14}
function useWindowWidth() {
  const [width, setWidth] = useState(window.innerWidth);
  // ...
  return width;
}

function useTheme(isMobile) {
  // ...
}

function Comment() {
  const width = useWindowWidth();
  const isMobile = width < MOBILE_VIEWPORT;
  const theme = useTheme(isMobile);
  return (
    <section className={theme.comment}>
      {/* ... */}
    </section>
  );
}
```

しかしもしミスをしたらどうなるのでしょう？どのようにデバッグすることになるのでしょう？

例えば `theme.comment`から取得するCSSクラスが誤っていたとします。
どのようにデバッグすることになるでしょうか？コンポーネント内にブレイクポイントやログをセットすることができるでしょう。

`theme`が誤っていて、`width`と`isMobile`は正しいことがわかるかもしれません。であればそれによって問題が`useTheme()`内にあるだろうことがわかります。あるいは`width`自体が誤っていると気づくかもしれません。であれば`seWindowWidth()`を見るべきだとわかります。

**中間の値を一目見ることでトップ・レベルのどのフックにバグがあるのかわかります。** それらの全ての実装を見てみる必要はありません。

その後、そのバグを持ったものに「ズーム・イン」することが可能です、それを繰り返せばよいのです。

これはカスタムフックのネストが深くなるほど重要になります。
３レベルの深さにネストしたカスタム・フックがあって、それぞれのレベルが３つの異なったカスタム・フックを内部で使用していると想像してください。**3箇所**のバグを探すのと、**3 + 3×3 + 3×3×3 = 39 箇所**をチェックするかもしれないこと、はとても大きな[違い](/the-bug-o-notation/) です。
幸運なことに, `useState()`は別のフックやコンポーネントに「影響を与える」ことができません。あるフックに返却されたバグのありそうな値は、その他の変数同様、その跡を残します。🐛

**評決:** ✅ `useState()`はコード内の因果関係を隠しません。私たちはバグへのパンくずを直接たどることができます。

---

## フックではないもの: `useBailout()`

最適化として、フックをつかってるコンポーネントは再描画を回避することが可能です。

その方法の一つは[`React.memo()`](https://reactjs.org/blog/2018/10/23/react-v-16-6.html#reactmemo)ラッパーをそのコンポーネント全体を囲むようにぷっとすることです。

One way to do it is to put a [`React.memo()`](https://reactjs.org/blog/2018/10/23/react-v-16-6.html#reactmemo) wrapper around the whole component. It bails out of re-rendering if props are shallowly equal to what we had during the last render. This makes it similar to `PureComponent` in classes.

`React.memo()` takes a component and returns a component:

```jsx{4}
function Button(props) {
  // ...
}
export default React.memo(Button);
```

**But why isn’t it just a Hook?**

Whether you call it `useShouldComponentUpdate()`, `usePure()`, `useSkipRender()`, or `useBailout()`, the proposal tends to look something like this:

```jsx
function Button({ color }) {
  // ⚠️ Not a real API
  useBailout(prevColor => prevColor !== color, color);

  return (
    <button className={'button-' + color}>  
      OK
    </button>
  )
}
```

There are a few more variations (e.g. a simple `usePure()` marker) but in broad strokes they have the same flaws.

### Composition

Let’s say we try to put `useBailout()` in two custom Hooks:

```jsx{4,5,19,20}
function useFriendStatus(friendID) {
  const [isOnline, setIsOnline] = useState(null);

  // ⚠️ Not a real API
  useBailout(prevIsOnline => prevIsOnline !== isOnline, isOnline);

  useEffect(() => {
    const handleStatusChange = status => setIsOnline(status.isOnline);
    ChatAPI.subscribe(friendID, handleStatusChange);
    return () => ChatAPI.unsubscribe(friendID, handleStatusChange);
  });

  return isOnline;
}

function useWindowWidth() {
  const [width, setWidth] = useState(window.innerWidth);
  
  // ⚠️ Not a real API
  useBailout(prevWidth => prevWidth !== width, width);

  useEffect(() => {
    const handleResize = () => setWidth(window.innerWidth);
    window.addEventListener('resize', handleResize);
    return () => window.removeEventListener('resize', handleResize);
  });

  return width;
}
```

Now what happens if you use them both in the same component?


```jsx{2,3}
function ChatThread({ friendID, isTyping }) {
  const width = useWindowWidth();
  const isOnline = useFriendStatus(friendID);
  return (
    <ChatLayout width={width}>
      <FriendStatus isOnline={isOnline} />
      {isTyping && 'Typing...'}
    </ChatLayout>
  );
}
```

When does it re-render?

If every `useBailout()` call has the power to skip an update, then updates from `useWindowWidth()` would be blocked by `useFriendStatus()`, and vice versa. **These Hooks would break each other.**

However, if `useBailout()` was only respected when *all* calls to it inside a single component “agree” to block an update, our `ChatThread` would fail to update on changes to the `isTyping` prop.

Even worse, with these semantics **any newly added Hooks to `ChatThread` would break if they don’t *also* call `useBailout()`**. Otherwise, they can’t “vote against” the bailout inside `useWindowWidth()` and `useFriendStatus()`.

**Verdict:** 🔴 `useBailout()` breaks composition. Adding it to a Hook breaks state updates in other Hooks. We want the APIs to be [antifragile](/optimized-for-change/), and this behavior is pretty much the opposite.

### Debugging

How does a Hook like `useBailout()` affect debugging?

We’ll use the same example:

```jsx
function ChatThread({ friendID, isTyping }) {
  const width = useWindowWidth();
  const isOnline = useFriendStatus(friendID);
  return (
    <ChatLayout width={width}>
      <FriendStatus isOnline={isOnline} />
      {isTyping && 'Typing...'}
    </ChatLayout>
  );
}
```

Let’s say the `Typing...` label doesn’t appear when we expect, even though somewhere many layers above the prop is changing. How do we debug it?

**Normally, in React you can confidently answer this question by looking *up*.** If `ChatThread` doesn’t get a new `isTyping` value, we can open the component that renders `<ChatThread isTyping={myVar} />` and check `myVar`, and so on. At one of these levels, we’ll either find a buggy `shouldComponentUpdate()` bailout, or an incorrect `isTyping` value being passed down. One look at each component in the chain is usually enough to locate the source of the problem.

However, if this `useBailout()` Hook was real, you would never know the reason an update was skipped until you checked *every single custom Hook* (deeply) used by our `ChatThread` and components in its owner chain. Since every parent component can *also* use custom Hooks, this [scales](/the-bug-o-notation/) terribly.

It’s like if you were looking for a screwdriver in a chest of drawers, and each drawer contained a bunch of smaller chests of drawers, and you don’t know how deep the rabbit hole goes.

**Verdict:** 🔴 Not only `useBailout()` Hook breaks composition, but it also vastly increases the number of debugging steps and cognitive load for finding a buggy bailout — in some cases, exponentially.

---

We just looked at one real Hook, `useState()`, and a common suggestion that is intentionally *not* a Hook — `useBailout()`. We compared them through the prism of Composition and Debugging, and discussed why one of them works and the other one doesn’t.

While there is no “Hook version” of `memo()` or `shouldComponentUpdate()`, React *does* provide a Hook called [`useMemo()`](https://reactjs.org/docs/hooks-reference.html#usememo). It serves a similar purpose, but its semantics are different enough to not run into the pitfalls described above.

`useBailout()` is just one example of something that doesn’t work well as a Hook. But there are a few others — for example, `useProvider()`, `useCatch()`, or `useSuspense()`.

Can you see why?

*(Whispers: Composition... Debugging...)*
