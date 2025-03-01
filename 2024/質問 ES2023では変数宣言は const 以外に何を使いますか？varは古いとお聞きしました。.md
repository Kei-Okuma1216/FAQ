その通りです。最新のJavaScript（ES2023）では、変数宣言に以下の2つの方法が推奨されています：

1. **`let`**:
   - ブロックスコープの変数を宣言します。
   - 再代入が可能です。

   ```javascript
   let count = 0;
   count = 5; // 再代入可能
   ```

2. **`const`**:
   - ブロックスコープの定数を宣言します。
   - 再代入不可（ただし、オブジェクトや配列の内容は変更可能）。

   ```javascript
   const PI = 3.14159;
   // PI = 3.14; // エラー: 再代入は不可

   const array = [1, 2, 3];
   array.push(4); // 配列の内容は変更可能
   ```

一方で、`var`は従来から使われてきた変数宣言の方法ですが、スコープの扱いが異なり、いくつかの問題があるため、近年では`let`や`const`の使用が推奨されています。具体的には、`var`は関数スコープを持ち、意図しない挙動を引き起こすことがあります。

例えば、`var`のスコープの問題：
```javascript
if (true) {
    var message = "Hello, world!";
}
console.log(message); // "Hello, world!" が表示される（関数スコープ）
```

同じコードを`let`で書くと：
```javascript
if (true) {
    let message = "Hello, world!";
}
console.log(message); // エラー: messageは定義されていない（ブロックスコープ）
```

したがって、最新のコードでは`let`と`const`を使用することをお勧めします。これにより、コードの可読性と予測可能性が向上します。

他にもJavaScriptに関する質問があれば、どうぞ教えてください！😊