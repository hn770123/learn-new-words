# 第12章：「言語機能で雰囲気で使ってる」系

**特徴:** コード書けるけど説明は無理なやつ

---

プログラミング言語の機能は、日々使っているからこそ「なんとなく動く」という状態に陥りやすい。`async/await`を書けば非同期処理ができる。`<T>`と書けばジェネリクスになる。`=>`と書けばアロー関数になる。動く。動くのだから問題ない。

問題が発生するのは、面接官が「では、クロージャとは何かを説明してください」と言った瞬間である。

本章では、毎日のように書いているのに「それ何？」と聞かれると固まる言語機能を解剖する。読み終わる頃には、少なくとも面接で30秒は時間を稼げるようになるだろう。

---

## async/await

### 雰囲気での理解

「非同期処理を同期っぽく書けるやつ」

### もう少し正確な理解

async/awaitは、非同期処理を「見た目は同期的に」書くための構文糖衣である。ここで重要なのは「見た目は」という部分だ。実際には非同期のままである。

従来のコールバック地獄を見てみよう。

```javascript
getUser(userId, function(user) {
  getOrders(user.id, function(orders) {
    getOrderDetails(orders[0].id, function(details) {
      // ここに到達する頃には、インデントが画面の右端に消えている
    });
  });
});
```

これがPromiseで改善され、async/awaitでさらに読みやすくなった。

```javascript
async function fetchOrderDetails(userId) {
  const user = await getUser(userId);
  const orders = await getOrders(user.id);
  const details = await getOrderDetails(orders[0].id);
  return details;
}
```

しかし「同期っぽく書ける」という言葉に騙されてはいけない。`await`を書いた行で処理が完了するまで「待っている」わけではない。正確には、その関数の実行が一時停止し、イベントループに制御が戻り、Promiseが解決されたら続きから再開される。

この説明を聞いて「それって結局何が嬉しいの？」と思った人は、おそらく今まで通りで問題ない。ただし、`await`を`for`ループの中に書いて「なぜかすごく遅い」と悩んだ経験があるなら、今こそ理解すべき時である。

### よくある誤解

**誤解：** `async`関数は自動的に並列実行される

**現実：** `async`関数は呼び出した順に実行される。並列実行したいなら`Promise.all()`を使う必要がある。筆者はこの事実を知るまでに、不必要に遅いAPIを3つほど本番環境にデプロイした。

```javascript
// 遅い（直列実行）
const user = await getUser();
const orders = await getOrders();
const products = await getProducts();

// 速い（並列実行）
const [user, orders, products] = await Promise.all([
  getUser(),
  getOrders(),
  getProducts()
]);
```

---

## ジェネリクス

### 雰囲気での理解

「`<T>`って書くやつ。型を後から決められる」

### もう少し正確な理解

ジェネリクスとは、型をパラメータ化する仕組みである。これにより、同じロジックを異なる型に対して再利用できる。

例えば、配列の最初の要素を返す関数を考えてみよう。

```typescript
function firstNumber(arr: number[]): number {
  return arr[0];
}

function firstString(arr: string[]): string {
  return arr[0];
}
```

ロジックは同じなのに、型が違うだけで関数を2つ書いている。これは効率が悪い。ジェネリクスを使えば、こう書ける。

```typescript
function first<T>(arr: T[]): T {
  return arr[0];
}

const num = first([1, 2, 3]);       // number
const str = first(['a', 'b', 'c']); // string
```

ここで`<T>`は「この関数は何らかの型Tを受け取り、同じ型Tを返す」という契約を表している。`T`は慣習的に「Type」の頭文字だが、実際には何でも良い。`<Banana>`でも動く。ただし、コードレビューで「なぜバナナなのか」という質問に答える覚悟が必要である。

### 制約付きジェネリクス

「任意の型を受け取る」だけでなく、「特定の条件を満たす型だけを受け取る」こともできる。

```typescript
function getLength<T extends { length: number }>(item: T): number {
  return item.length;
}

getLength("hello");     // OK: stringにはlengthがある
getLength([1, 2, 3]);   // OK: 配列にもlengthがある
getLength(123);         // エラー: numberにlengthはない
```

この`extends`は継承ではなく「〜の条件を満たす」という意味である。英語の意味と一致しないため混乱するが、言語設計者たちもおそらく後悔しているだろう。

---

## デコレータ/アノテーション

### 雰囲気での理解

「@マークのおまじない。付けると何かが起こる」

### もう少し正確な理解

デコレータとアノテーションは、どちらも「@」記号で始まるが、役割が異なる。

**アノテーション（Java、Kotlinなど）:**
メタデータを付与する仕組み。それ自体は何も「しない」。コンパイラやフレームワークがそのメタデータを読み取って、何かをする。

```java
@Override
public String toString() {
  return "これはオーバーライドです";
}
```

`@Override`は「このメソッドは親クラスのメソッドをオーバーライドしている」というメタデータである。コンパイラはこれを見て、本当にオーバーライドしているかチェックする。もしスペルミスで`toStrng()`と書いていたら、エラーになる。

**デコレータ（Python、TypeScriptなど）:**
関数やクラスを「装飾」する、つまり振る舞いを変更する仕組み。実際にコードを実行する。

```python
def log_calls(func):
    def wrapper(*args, **kwargs):
        print(f"Calling {func.__name__}")
        return func(*args, **kwargs)
    return wrapper

@log_calls
def greet(name):
    return f"Hello, {name}"

greet("World")  # "Calling greet" と表示されてから "Hello, World" を返す
```

`@log_calls`は`greet`関数を受け取り、ログを出力してから元の関数を呼び出す新しい関数で「包んで」返している。

### 本質的な違い

- **アノテーション:** 「このコードについてのメモ書き」。誰かが読んで判断する。
- **デコレータ:** 「このコードを加工して」という指示。実際に加工される。

この違いを理解していないと、Springの`@Transactional`がなぜ時々動かないのかを理解できない。ヒント：同じクラス内から呼び出すと、プロキシを経由しないため動かない。筆者はこのバグを3回修正した。すべて自分が書いたコードだった。

---

## 型推論

### 雰囲気での理解

「型を書かなくてもコンパイラが勝手に判断してくれるやつ」

### もう少し正確な理解

型推論とは、プログラマが明示的に型を書かなくても、コンパイラが文脈から型を推測する機能である。

```typescript
let x = 42;        // xはnumber型と推論される
let y = "hello";   // yはstring型と推論される
let z = [1, 2, 3]; // zはnumber[]型と推論される
```

これにより、冗長な型注釈を省略でき、コードが簡潔になる。しかし、「書かなくていい」と「書くべきではない」は違う。

### 書くべき場所、省略すべき場所

**省略して良い場所：**
- ローカル変数で、右辺から型が明らかな場合
- ラムダ式の引数で、文脈から型が確定する場合

```typescript
// 右辺から明らか
const users = fetchUsers();  // User[]

// 文脈から確定
users.filter(user => user.isActive);  // userはUser型
```

**明示すべき場所：**
- 関数の引数と戻り値
- パブリックなAPIの境界
- 複雑な型や`any`に推論されそうな場合

```typescript
// 関数シグネチャは明示すべき
function processUser(user: User): ProcessedUser {
  // ...
}
```

「全部推論に任せればいいじゃん」という意見もあるが、6ヶ月後の自分は今の自分ではない。型注釈は未来の自分への手紙である。

---

## Null安全

### 雰囲気での理解

「Optionalとか?とか付けるやつ。nullチェックしろってこと？」

### もう少し正確な理解

Null安全とは、「nullかもしれない値」と「絶対にnullではない値」を型システムで区別する仕組みである。

Tony Hoare（クイックソートの発明者）は、nullの発明を「10億ドルの過ち」と呼んだ。この過ちを型システムで防ごうというのがNull安全の思想である。

```kotlin
var name: String = "Alice"    // nullを代入できない
var nickname: String? = null  // nullを代入できる

println(name.length)          // OK
println(nickname.length)      // コンパイルエラー：nullかもしれない
println(nickname?.length)     // OK：nullなら何もしない
println(nickname!!.length)    // OK：nullでないことをプログラマが保証する（危険）
```

### Optional、Maybe、nullable型の違い

これらは「nullかもしれない」を表現する異なる方法である。

- **nullable型（Kotlin、TypeScript）:** 型に`?`を付ける。`String?`は「StringまたはNull」
- **Optional（Java）:** `Optional<String>`というラッパークラス。値があるかないかを明示
- **Maybe（Haskell）:** `Maybe String`という代数的データ型。`Just "value"`か`Nothing`

本質的には同じ問題を解決しているが、アプローチが異なる。JavaのOptionalは「値をラップする」アプローチで、Kotlinの`?`は「型システムに組み込む」アプローチである。

`!!`（Kotlinのnon-null assertion）を多用しているコードを見たら、それは「nullかもしれないけど、きっと大丈夫」という祈りの表明である。コードレビューで指摘すべきポイントだ。

---

## パターンマッチング

### 雰囲気での理解

「switch文の進化系。なんかもっと色々できるらしい」

### もう少し正確な理解

パターンマッチングとは、値の「構造」に対してマッチングを行い、同時に値を分解・抽出する機能である。switch文とは似て非なるものだ。

switch文は「値が何に等しいか」を調べる。

```java
switch (status) {
  case "active":
    // ...
    break;
  case "inactive":
    // ...
    break;
}
```

パターンマッチングは「値がどのような構造を持っているか」を調べ、その部品を取り出す。

```rust
match user {
    User { name, age: 18..=65, active: true } => {
        println!("{} is an active adult user", name);
    }
    User { name, active: false, .. } => {
        println!("{} is inactive", name);
    }
    _ => {
        println!("Other case");
    }
}
```

ここでは、`user`がどのような構造を持っているかでマッチングし、同時に`name`や`age`を変数として取り出している。

### 網羅性チェック

パターンマッチングの真価は、網羅性チェックにある。すべてのケースを処理しているかをコンパイラが検証してくれる。

```rust
enum Shape {
    Circle(f64),
    Rectangle(f64, f64),
}

fn area(shape: Shape) -> f64 {
    match shape {
        Shape::Circle(radius) => 3.14159 * radius * radius,
        // Shape::Rectangleを忘れている！
        // → コンパイルエラー：non-exhaustive patterns
    }
}
```

将来`Triangle`を追加したとき、すべてのmatch式でコンパイルエラーが発生する。これは機能であり、バグではない。修正すべき場所をコンパイラが教えてくれているのだ。

---

## ラムダ式/アロー関数

### 雰囲気での理解

「`=>`とか`->`で書く短い関数。無名関数ともいう」

### もう少し正確な理解

ラムダ式（アロー関数）は、名前を持たない関数を簡潔に記述する構文である。

```javascript
// 従来の関数
function add(a, b) {
  return a + b;
}

// アロー関数
const add = (a, b) => a + b;
```

### `=>`と`->`の違い

言語によって矢印の向きが異なる。

- **JavaScript/TypeScript:** `=>` を使う
- **Java:** `->` を使う
- **Kotlin:** `->` を使う
- **Scala:** `=>` を使う
- **Haskell:** `->` を使う

なぜ統一されていないのか。それは言語設計者がそれぞれ独自の美的感覚を持っているからである。プログラミング言語の世界に標準化委員会は存在するが、矢印の向きを議論する委員会は存在しない。

### thisの束縛

JavaScriptにおいて、アロー関数と従来の関数の最大の違いは`this`の扱いである。

```javascript
const obj = {
  name: "Object",
  traditionalMethod: function() {
    setTimeout(function() {
      console.log(this.name);  // undefined（thisがwindowを指す）
    }, 100);
  },
  arrowMethod: function() {
    setTimeout(() => {
      console.log(this.name);  // "Object"（外側のthisを保持）
    }, 100);
  }
};
```

アロー関数は「レキシカルにthisを束縛する」。つまり、定義された場所のthisを覚えている。これを知らないと、Reactで`this.setState is not a function`というエラーと3時間戦うことになる。

---

## クロージャ

### 雰囲気での理解

「関数の中の関数。外側の変数を使えるやつ」

### もう少し正確な理解

クロージャとは、関数とその関数が定義された環境（レキシカルスコープ）を一緒にパッケージしたものである。

「何が閉じている（close）のか」という疑問への答えは、「外側のスコープの変数への参照を閉じ込めている」である。

```javascript
function createCounter() {
  let count = 0;

  return function() {
    count++;
    return count;
  };
}

const counter = createCounter();
console.log(counter()); // 1
console.log(counter()); // 2
console.log(counter()); // 3
```

`createCounter`は実行が終わっているのに、返された関数は`count`にアクセスできている。これがクロージャである。`count`変数は関数と一緒に「閉じ込められて」いる。

### 実用的な例

クロージャは以下の場面で活躍する。

**プライベート変数の実現:**
```javascript
function createBankAccount(initial) {
  let balance = initial;

  return {
    deposit: (amount) => balance += amount,
    withdraw: (amount) => balance -= amount,
    getBalance: () => balance
  };
}

const account = createBankAccount(1000);
account.deposit(500);
console.log(account.getBalance()); // 1500
// account.balance にはアクセスできない
```

**ループとクロージャの罠:**
```javascript
// 期待通りに動かない例
for (var i = 0; i < 3; i++) {
  setTimeout(function() {
    console.log(i);
  }, 100);
}
// 出力: 3, 3, 3（すべて3）

// 正しく動く例
for (let i = 0; i < 3; i++) {
  setTimeout(function() {
    console.log(i);
  }, 100);
}
// 出力: 0, 1, 2
```

`var`はブロックスコープを持たないため、ループ終了後の`i`（=3）が参照される。`let`はブロックスコープを持つため、各イテレーションで新しい`i`が作られる。この違いは、筆者が深夜2時に学んだ。

---

## イテレータ/ジェネレータ

### 雰囲気での理解

「forで回せるやつ。配列っぽいやつ」

### もう少し正確な理解

**イテレータ**は、コレクションの要素を順番にアクセスするためのオブジェクトである。「次の要素を取得する」という操作を抽象化したもの。

```javascript
const arr = [1, 2, 3];
const iterator = arr[Symbol.iterator]();

console.log(iterator.next()); // { value: 1, done: false }
console.log(iterator.next()); // { value: 2, done: false }
console.log(iterator.next()); // { value: 3, done: false }
console.log(iterator.next()); // { value: undefined, done: true }
```

**ジェネレータ**は、イテレータを簡単に作成するための関数である。`function*`と`yield`を使う。

```javascript
function* countUp(max) {
  for (let i = 1; i <= max; i++) {
    yield i;
  }
}

for (const num of countUp(3)) {
  console.log(num); // 1, 2, 3
}
```

### 遅延評価

ジェネレータの真価は「遅延評価」にある。すべての値を最初に計算するのではなく、必要になったときに次の値を計算する。

```javascript
function* infiniteNumbers() {
  let n = 1;
  while (true) {
    yield n++;
  }
}

const numbers = infiniteNumbers();
console.log(numbers.next().value); // 1
console.log(numbers.next().value); // 2
// 無限に続く（が、呼び出すまで計算されない）
```

無限のシーケンスを表現できるのは、遅延評価のおかげである。すべての値をメモリに持つ必要がない。

---

## スプレッド構文

### 雰囲気での理解

「`...`で展開するやつ。便利」

### もう少し正確な理解

スプレッド構文（`...`）は、イテラブルなオブジェクトを「展開」する。配列やオブジェクトを「ばらす」イメージだ。

```javascript
// 配列の展開
const arr1 = [1, 2, 3];
const arr2 = [4, 5, 6];
const combined = [...arr1, ...arr2]; // [1, 2, 3, 4, 5, 6]

// オブジェクトの展開
const obj1 = { a: 1, b: 2 };
const obj2 = { c: 3, d: 4 };
const merged = { ...obj1, ...obj2 }; // { a: 1, b: 2, c: 3, d: 4 }
```

### 浅いコピーに注意

スプレッド構文は「浅いコピー」を行う。ネストしたオブジェクトは参照がコピーされるだけである。

```javascript
const original = {
  name: "Alice",
  address: { city: "Tokyo" }
};

const copy = { ...original };
copy.name = "Bob";           // 独立している
copy.address.city = "Osaka"; // 元のオブジェクトも変わる！

console.log(original.name);         // "Alice"
console.log(original.address.city); // "Osaka" ← 変わっている
```

「Immutableに書いているつもりなのにバグる」原因の8割はこれである。

### レスト構文との関係

同じ`...`でも、「集める」方向に使うとレスト構文になる。

```javascript
// スプレッド（展開）
const arr = [1, 2, 3];
console.log(...arr); // 1 2 3

// レスト（収集）
function sum(...numbers) {
  return numbers.reduce((a, b) => a + b, 0);
}
console.log(sum(1, 2, 3, 4, 5)); // 15
```

見た目は同じだが、役割が逆である。これを混同すると、コードを読むときに混乱する。

---

## 分割代入（Destructuring）

### 雰囲気での理解

「`const { a, b } = obj`みたいなやつ。オブジェクトから取り出す」

### もう少し正確な理解

分割代入は、配列やオブジェクトから値を取り出し、個別の変数に代入する構文である。

```javascript
// オブジェクトの分割代入
const user = { name: "Alice", age: 30, city: "Tokyo" };
const { name, age } = user;
console.log(name); // "Alice"
console.log(age);  // 30

// 配列の分割代入
const coords = [10, 20, 30];
const [x, y] = coords;
console.log(x); // 10
console.log(y); // 20
```

### デフォルト値とリネーム

```javascript
// デフォルト値
const { name, country = "Japan" } = { name: "Alice" };
console.log(country); // "Japan"

// リネーム
const { name: userName } = { name: "Alice" };
console.log(userName); // "Alice"

// 両方組み合わせ
const { name: userName = "Anonymous" } = {};
console.log(userName); // "Anonymous"
```

### ネストした分割代入

深いネストも分割できる。ただし、可読性との戦いになる。

```javascript
const response = {
  data: {
    user: {
      profile: {
        name: "Alice",
        avatar: "alice.png"
      }
    }
  }
};

const { data: { user: { profile: { name } } } } = response;
console.log(name); // "Alice"
```

これが便利なのか、それとも読みにくいだけなのかは、チームの議論が必要である。個人的には、2段階までは許容、3段階以上は「一旦変数に入れよう」と言いたくなる。

---

## 高階関数

### 雰囲気での理解

「map/filter/reduceのこと。使えるけど説明は難しい」

### もう少し正確な理解

高階関数とは、「関数を引数として受け取る」または「関数を戻り値として返す」関数である。

```javascript
// 関数を引数として受け取る
const numbers = [1, 2, 3, 4, 5];

const doubled = numbers.map(n => n * 2);         // [2, 4, 6, 8, 10]
const evens = numbers.filter(n => n % 2 === 0);  // [2, 4]
const sum = numbers.reduce((a, b) => a + b, 0);  // 15

// 関数を戻り値として返す
function multiply(factor) {
  return function(number) {
    return number * factor;
  };
}

const double = multiply(2);
const triple = multiply(3);
console.log(double(5));  // 10
console.log(triple(5));  // 15
```

### map, filter, reduce の違い

この3つは高階関数の代表格だが、役割が異なる。

- **map:** 各要素を変換する（要素数は変わらない）
- **filter:** 条件に合う要素を抽出する（要素数が減ることがある）
- **reduce:** 全要素を1つの値にまとめる

```javascript
const products = [
  { name: "Apple", price: 100, inStock: true },
  { name: "Banana", price: 80, inStock: false },
  { name: "Cherry", price: 150, inStock: true },
];

// 在庫ありの商品だけを抽出し、価格を取り出し、合計する
const totalInStock = products
  .filter(p => p.inStock)     // 在庫ありだけ抽出
  .map(p => p.price)          // 価格だけ取り出す
  .reduce((a, b) => a + b, 0); // 合計する

console.log(totalInStock); // 250
```

### メソッドチェーンの罠

上記のチェーンは読みやすいが、毎回新しい配列を作成している。巨大なデータセットでは、パフォーマンスの問題になることがある。

```javascript
// 3回の配列生成
const result = hugeArray
  .filter(...)  // 1回目
  .map(...)     // 2回目
  .filter(...); // 3回目

// 1回のループで済ませる（最適化が必要な場合）
const result = hugeArray.reduce((acc, item) => {
  if (/* filter条件1 */) {
    const mapped = /* map処理 */;
    if (/* filter条件2 */) {
      acc.push(mapped);
    }
  }
  return acc;
}, []);
```

ただし、可読性とパフォーマンスのトレードオフである。「遅い」と判明するまでは、読みやすいコードを書くべきだ。

---

## 面接で聞かれて詰んだ言語機能ランキング

筆者の経験と、同僚へのアンケートから集計した非公式ランキングである。

### 第1位：クロージャ

「クロージャとは何か説明してください」は、面接官の定番質問である。正確に答えられる人は少ない。「関数と、その関数が参照する外側のスコープの変数を一緒にパッケージしたもの」と答えられれば合格。「関数の中の関数」だけでは不十分。

### 第2位：イベントループ

JavaScriptの場合、「なぜsetTimeoutの0msは即座に実行されないのか」という質問が来る。コールスタック、タスクキュー、マイクロタスクキューを説明できないと詰む。

### 第3位：ジェネリクスの共変・反変

「なぜ`List<Dog>`を`List<Animal>`に代入できないのか」。これを正確に説明できる人は、言語設計者か、過去に痛い目にあった人だけである。

### 第4位：async/awaitの内部動作

「awaitした瞬間、何が起きているか説明してください」。Promiseの解決、イベントループへの制御の返却、コルーチンの再開まで説明できれば完璧。

### 第5位：「参照渡し」と「値渡し」

「JavaScriptは参照渡しですか、値渡しですか」。正解は「値渡しだが、オブジェクトの場合は参照が値としてコピーされる」。これを「参照の値渡し」と呼ぶ人もいる。

---

## コラム1：なぜ全言語がasync/awaitを採用したのか

C#が2012年に導入したasync/awaitは、瞬く間に他の言語に広まった。Python、JavaScript、Rust、Kotlin、Swift、Dart...主要な言語のほぼすべてが採用している。

なぜこれほど普及したのか。

**1. コールバック地獄の解消**

非同期処理はコールバックで書くのが伝統だった。しかし、コールバックがネストすると、コードはピラミッド状に右へ右へとずれていく。これを「コールバック地獄」または「破滅のピラミッド」と呼ぶ。

```javascript
getUser(function(user) {
  getOrders(user, function(orders) {
    getDetails(orders, function(details) {
      // インデントが画面の右端に消えている
    });
  });
});
```

async/awaitはこれを平坦なコードに変換する。人間の脳は上から下へ読むのが得意なので、理にかなっている。

**2. エラーハンドリングの統一**

コールバックベースでは、エラー処理が煩雑になる。各コールバックでエラーをチェックする必要があった。async/awaitなら`try/catch`で統一的に扱える。

**3. 既存の構文との親和性**

`for`ループ、`if`文、`try/catch`といった既存の制御構文がそのまま使える。新しい概念を学ぶコストが低い。

**4. デバッグのしやすさ**

スタックトレースが追いやすくなる。コールバックチェーンでは、エラーがどこから来たのか追跡が困難だった。

---

## コラム2：ジェネリクスの `<T>` は何の略？

`<T>`の`T`は「Type」の頭文字である。ただし、これは慣習であり、言語仕様で定められているわけではない。

### よく使われる型パラメータ名

- `T` - Type（一般的な型）
- `E` - Element（コレクションの要素）
- `K` - Key（マップのキー）
- `V` - Value（マップの値）
- `N` - Number（数値型）
- `R` - Return type / Result（戻り値の型）

```java
// Javaの例
public interface Map<K, V> {
    V get(K key);
    void put(K key, V value);
}
```

### なぜ一文字なのか

歴史的経緯である。Javaのジェネリクス設計時、「型パラメータは一文字の大文字」という慣習が確立された。これはC++のテンプレートから引き継がれたものだ。

しかし、現代では意味のある名前を使うことも増えている。

```typescript
// 意味のある名前を使う例
function fetchData<ResponseType>(url: string): Promise<ResponseType> {
  // ...
}
```

一文字か意味のある名前か、チームのスタイルガイドに従うのが無難である。

---

## コラム3：「関数型っぽく書いて」と言われたときの対処法

コードレビューで「もっと関数型っぽく書けない？」と言われたことはないだろうか。この曖昧な指示にどう対応すべきか。

### 「関数型っぽい」の具体的な意味

多くの場合、以下のいずれかを指している。

**1. for/whileループをmap/filter/reduceに置き換える**

```javascript
// 命令型
const results = [];
for (const item of items) {
  if (item.active) {
    results.push(item.name);
  }
}

// 関数型っぽい
const results = items
  .filter(item => item.active)
  .map(item => item.name);
```

**2. 変数の再代入を避ける**

```javascript
// 再代入あり
let total = 0;
for (const n of numbers) {
  total += n;
}

// 再代入なし
const total = numbers.reduce((a, b) => a + b, 0);
```

**3. 副作用を分離する**

純粋関数（同じ入力には同じ出力を返し、外部に影響を与えない関数）を中心に書き、副作用（API呼び出し、DOM操作など）は端に追いやる。

### 反論すべきとき

「関数型っぽく」が常に正しいわけではない。以下のケースでは、命令型の方が適切なこともある。

- パフォーマンスが重要な場合（ループの方が速いことがある）
- 途中で処理を中断したい場合（`break`に相当する機能がない）
- コードが複雑になりすぎる場合

「関数型で書いた結果、チームの誰も読めないコードになりました」は本末転倒である。

---

## 本章のまとめ

言語機能は「使える」と「説明できる」の間に深い溝がある。その溝を埋めることで、以下の効果が得られる。

1. **デバッグが速くなる** - なぜ動くのかを理解していれば、なぜ動かないのかも分かる
2. **適切な場面で使える** - 道具の特性を理解すれば、適切な場面で使える
3. **面接で困らない** - 少なくとも30秒は時間を稼げる
4. **コードレビューで指摘できる** - 「これ、クロージャでメモリリークしますよ」と言える

本章で扱った12の機能すべてを完璧に説明できる必要はない。しかし、「雰囲気で使っている」から「だいたい分かって使っている」への移行は、エンジニアとしての成長に不可欠である。

次に`async/await`や`<T>`を書くとき、その背後で何が起きているのか、少しだけ意識してみてほしい。コードは、理解の深さに応じて、その姿を変えて見えてくるものである。
