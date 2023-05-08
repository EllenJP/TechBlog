# タイトル
Unityで使われるコルーチンの戻り値の型とyieldステートメント、StartCoroutine()メソッドについて調べてみた。

<br>

# はじめに
今回はコルーチンについて戻り値の型とyieldステートメント、StartCoroutine()メソッドについて深堀していきます。

コルーチンは非同期処理を実装できる方法の1つで、**処理の途中で一時中断して、再度中断した位置から処理を再開できる機能を持つ関数**のことです。

Unityではじめて非同期処理に触れる方は、まずコルーチンから始める方が多いと思います。

実は、C#にはコルーチンと呼ばれる概念はなくて、Unityが独自にC#の機能を用いて実装した非同期処理をコルーチンと呼んでいます。そのため他の非同期処理の実装に比べて簡単に実装できます。

コルーチンの定義の仕方は下記のようにIEnumerator型を戻り値として、yieldステートメントをメソッド内に記述します。

```cs
// 2秒待機するコルーチン
IEnumerator MyCoroutine()
{
    yield return new WaitForSeconds(2f);
}
```

コルーチンを呼び出す時は、StartCoroutine()メソッドの引数にコルーチンを渡します。

```cs
void Start()
{
    StartCoroutine(MyCoroutine());
}
```

慣れないうちは、コルーチンの戻り値の型をIEnumerable型と間違えたり、yieldステートメントの記述を忘れてしまったりすることが多いです。

なので、はじめは「コルーチンの戻り値の型はIEnumerator型、メソッド内でyieldステートメントを使う」と覚えてしまってもいいかもしれませんが、使われる理由や背景が分かれば理解しやすいと思うので興味がある方は是非最後まで読んでみてください。

<br>

# 本記事で話すこと
下記のような疑問を解決することをゴールにしています。

* なぜコルーチンの戻り値の型はIEnumerator型でなければならないのか？
* コルーチンでreturn文ではなく、yield return文を使う理由は？
* StartCoroutine()メソッドは何をしているのか？

※今回はコルーチンをどう使うのか？については話しませんので、ご注意ください。

<br>

# コルーチンの戻り値の型について
## IEnumerator型とIEnumerable型の違い
まずは、IEnumerator型について話していきます。その際IEnumerable型も一緒に学ぶと理解がしやすいです。

この2つはよく似ており、どちらも反復処理で使われています。反復処理とは、コレクションの要素を順番に処理したり同じ処理を繰り返し実行したりするものです。

IEnumerator型はIEnumeratorインターフェースを実装しているクラスや構造体のインスタンスで、IEnumerable型はIEnumerableインターフェースを実装しているクラスや構造体のインスタンスのことです。

それぞれのインターフェースの役割を簡単に説明すると、
* IEnumeratorインターフェースは**反復処理に必要な機能を提供する**インターフェース
* IEnumerableインターフェースは**反復処理に必要な機能を持つインスタンスを返す**インターフェース

となります。

### IEnumeratorインターフェースについて
まず、IEnumeratorインターフェースで書かれている **"反復処理に必要な機能を提供する"** とは、どういう意味でしょうか？

説明のために、コレクションの要素を順番にアクセスする反復処理のプログラムを用意しました。

**Sample1**
```cs
void Start()
{
    List<int> collectionList = new List<int>{ 1, 2, 3 };
    for (int i = 0; i < collectionList.Count; i++)
    {
        Debug.Log($"data: {collectionList[i]}");
    }
}
```

Sample1を見てみると、反復処理をするためには以下の**3つの機能**が必要になりそうです。

1. 現在のインデックスの要素を取得: collectionList[i]
1. 次の要素があるかの判定: i < collectionList.Count, i++;
1. インデックスの初期化: int i = 0

ここで、IEnumeratorインターフェースの定義を確認すると、

```cs
public interface IEnumerator
{
    // 1. 現在のインデックスの要素を返す。
    object Current { get; }

    // 2. 次の要素があるかの判定
    // あればtrue/なければfalseを返す。
    bool MoveNext();
    
    // 3. インデックスを初期化する。
    void Reset();
}
```
先程の反復処理に必要な3つの機能が定義されています。

また、Sample1ではインデックスを用いて要素にアクセスしていましたが、Sample2のようにIEnumeratorインターフェースのMoveNext()メソッドを使うことで、for文でインデックスを使用せずに要素を取得することもできます。

**Sample2**
```cs
void Start()
{
    List<int> collectionList = new List<int>{ 1, 2, 3 };

    // この部分はのちほど説明します。
    var enumerator = collectionList.GetEnumerator();

    // MoveNext()メソッドは内部のインデックスを1つ進めて次の要素があるか判定する。
    // 次の要素があればtrue、なければfalse。
    while (enumerator.MoveNext())
    {
        // 内部のインデックスが指す要素を取得する。
        Debug.Log($"data: {enumerator.Current}");
    }
}
```

Sample2では、MoveNext()メソッドを呼び出すことで内部インデックスを1つ進めて次の要素があるかを判定します。次の要素があれば現在の内部インデックスが示す値を取得するCurrentプロパティが呼び出されます。

### IEnumerableインターフェースについて

続いて、IEnumerableインターフェースの**反復処理に必要な機能を持つインスタンスを返す**についてですが、パッと見た感じだと何を言っているのか分かりづらいです。

もう少し詳しく説明すると、

オブジェクト指向では、すべての処理をMainクラスに書くのではなく役割ごとにクラスを作成します。そのため、反復処理を行いたい場合は反復処理を専門で行うクラスを用意する必要があります。

そして、反復処理を行いたい場合は、Mainクラス側で反復処理を行うクラスのオブジェクト（インスタンス）を取得して、そのオブジェクト（インスタンス）経由で反復処理を行うクラスのメソッドを呼び出します。

この **"Mainクラス側で反復処理を行うクラスのオブジェクト（インスタンス）を取得して"** の部分がIEnumerableインターフェースの役割となります。

言い換えると、**呼び出し側（Mainクラス）に反復処理を行うクラスのオブジェクト（インスタンス）を渡すこと**がIEnumerableインターフェースの役割となります。

（※本記事ではインスタンスとオブジェクトは同義として取り扱っています。紛らわしくてすみません...）

ここで、IEnumerableインターフェースの中身を確認してみると、
GetEnumerator()メソッドが宣言されており、戻り値としてIEnumerator型を返しています。
```cs
public interface IEnumerable
{
    // IEnumeratorインターフェースを実装したオブジェクトを返す。
    IEnumerator GetEnumerator();
}
```

つまり、IEnumerableインターフェースを実装したクラスや構造体は、GetEnumerator()メソッドで反復処理の機能を持つIEnumerator型オブジェクトを返しています。

IEnumerator、IEnumerableインターフェースの役割が分かったので改めて反復処理の流れを振り返ってみます。

1. IEnumerableインターフェースを実装したオブジェクト（List\<T>）を生成する。
1. 生成したIEnumerableオブジェクトからGetEnumerator()メソッドを呼び出してIEnumeratorオブジェクトを取得する。
1. 取得したIEnumeratorオブジェクトを経由してMoveNext()メソッドを呼び出し反復処理を行う。

また反復処理をするときに、Sample1、2ではインデックスを用意したり、GetEnumerator()メソッドやMoveNext()メソッドを呼び出したりと少し冗長でした。

もっと簡潔に書けるとうれしいです。そこで、foreach文の登場です。

```cs
void Start()
{
    // List<T>クラスはIEnumerableインターフェースを実装している。
    List<int> collectionList = new List<int>{ 1, 2, 3 };
    // 自動的にGetEnumerator()メソッドが呼ばれ、IEnumeratorオブジェクトが生成される。
    foreach (var data in collectionList)
    {
        Debug.Log($"data: {data}");
    }
}
```

上記のように、foreach文にIEnumerableインターフェースを実装したオブジェクト（配列やList\<T>等）を渡すと、自動的にGetEnumerator()メソッドが呼び出されてIEnumeratorオブジェクトが生成されます。

そして内部でIEnumeratorオブジェクトのMoveNext()メソッドを呼び出して、Currentプロパティの値を"data"変数に代入されます。

余談ですが、

「なぜ、IEnumeratorインターフェースとIEnumerableインターフェースを用意するのか？」と疑問に思った方がいるかもしれません。

その理由の1つに機能の分離が挙げられます。
例えば、別々の反復処理の実装を行いたい場合に、2つのインターフェースを1つにまとめてしまうと、IEnumeratorインターフェースの実装部分は異なりますが、呼び出し側に反復処理クラスのオブジェクトを渡すIEnumerableインターフェースの実装部分は2つとも同じ内容になります。

そのため、冗長になるのを防ぐために別々のインターフェースを定義しています。こうして役割を明確にすることで仕様変更に強いプログラムを書くことができます。



一旦ここまでをまとめます。
* 反復処理にはIEnumerator、IEnumerableインターフェースの2つが関係している。
* IEnumeratorインターフェースを実装したオブジェクトは**反復処理に必要な機能を提供する**役割を持つ。
* IEnumerableインターフェースを実装したオブジェクトは、**呼び出し側に反復処理を行うクラスのオブジェクト（インスタンス）を渡す**役割を持つ。

次の章では、List\<T>クラスのような既にIEnumerableインターフェースが実装されているクラスではなく、IEnumerator、IEnumerableインターフェースを独自に実装したクラスの反復処理について見ていきます。

## IEnumerator、IEnumerableインターフェースを独自に実装した反復処理
今回は"敵の情報を反復処理で表示させる"プログラムを例にIEnumerator、IEnumerableインターフェースを実装していきます。

全体のプログラムは以下の4つのクラスで構成されています。
* Enemyクラス: 敵の情報を定義。
* EnemyCollectionクラス: IEnumerableインターフェースを実装。
* EnemyEnumeratorクラス: IEnumeratorインターフェースを実装。
* 呼び出し部分（Mainクラス）: 反復処理を呼び出す。

それでは、各クラスの実装について見ていきます。

まず、Enemyクラスでは敵の情報としてName、Hpが定義されています。

**Sample3.Enemy**
```cs
public class Enemy
{
    public string Name { get; private set; }
    public int Hp { get; private set; }

    public Enemy(string name, int hp)
    {
        Name = name;
        Hp = hp;
    }
}
```

次に、EnemyCollectionクラスでは、IEnumerableインターフェースを実装しているため、GetEnumerator()メソッドで**反復処理のメソッドを持つIEnumeratorオブジェクト**を返します。そのために内部でコレクションを保持しています。

**Sample3.EnemyCollection**
```cs
public class EnemyCollection : IEnumerable
{
    // IEnumeratorオブジェクトを生成するために内部でコレクションを保持。
    private readonly ArrayList enemies;

    public EnemyCollection(ArrayList enemies)
    {
        // コレクションのコピー
        this.enemies = new ArrayList(enemies);
    }

    public IEnumerator GetEnumerator()
    {
        // IEnumeratorインターフェースを実装したクラスのインスタンスを生成して戻り値とする。
        // EnemyEnumerator型をIEnumerator型にアップキャストする。
        return (IEnumerator) new EnemyEnumerator(enemies);
    }
}
```

また、今回は内部コレクション（enemies変数）としてArrayListクラスを使用しています。

ArrayListクラスはobject型の要素を持つコレクションを生成できます。配列やList\<T>クラス等でも可能ですが、今後の説明のために、ここではArrayListクラスを使用しています。

GetEnumerator()メソッドでIEnumerator型にキャストされている理由は、インターフェース側で戻り値の型が決められているからです。

今回のように独自にIEnumeratorインターフェースを実装したクラスのオブジェクトを返すにはIEnumerator型にアップキャストする必要があります。ただし明示的に書かなくても自動的にアップキャストされます。

続いて、EnemyEnumeratorクラスでは反復処理の実装を定義しています。
IEnumeratorインターフェースを実装しているため、3つのメソッド、プロパティを実装する必要があります。

**Sample3.EnemyEnumerator**
```cs
public class EnemyEnumerator : IEnumerator
{
    // 反復処理をしたいコレクションを格納。
    private readonly ArrayList enemies;
    // どの要素か決めるインデックス。
    private int index;

    public EnemyEnumerator(ArrayList enemies)
    {
        this.enemies = new ArrayList(enemies);
        // MoveNext()メソッド呼び出し時に+1されるため。
        index = -1;
    }

    // 次の要素があるかどうか。
    public bool MoveNext()
    {
        index++;
        return index < enemies.Count;
    }

    // 初期化: 再度最初の要素から取得したい場合に用いる。
    public void Reset()
    {
        index = -1;
    }

    // 現在のインデックスの要素を返す。
    // 本来は範囲外の要素にアクセス場合にエラーをキャッチする必要がありますが今回は分かりやすくするために省いています。
    public object Current => enemies[index];
}
```

はじめに、コンストラクタでコレクションのコピーとindexの初期化が行われます。

次の要素があるかの判断はMoveNext()メソッドで行い、次の要素がある場合は、現在のインデックスが指す要素をCurrentプロパティで取得できます。

Currentプロパティについて補足すると、Currentプロパティはobject型を返すことがインターフェースで決められているため、要素の戻り値はobject型でなければなりません。
今回は、ArrayListクラスの要素がobject型のため、アップキャストせずにそのまま返しています。

<br>

最後に、呼び出し部分です。呼び出し部分の流れを説明すると、
1. 反復処理を行いたいコレクションを生成する。
1. EnemyCollectionクラスのインスタンスを生成する。
1. foreach文にEnemyCollectionオブジェクトを渡す。
1. foreach内部でGetEnumerator()メソッドが呼び出されてIEnumeratorオブジェクトを生成する。
1. IEnumeratorオブジェクトのMoveNext()メソッドが呼ばれ、Currentプロパティで取得した要素を"data"変数に代入する。
1. Debug.Log()メソッドで要素を表示する。
1. MoveNext()メソッドから次の要素がなくなるまで繰り返す。

となります。

**Sample3.呼び出し部分（Mainクラス）**
```cs
void Start()
{
    ArrayList enemies = new ArrayList()
    {
        new Enemy("ゴブリン", 5),
        new Enemy("ドラゴン", 200),
    };

    var enemyCollection = new EnemyCollection(enemies);
    // data変数はobject型
    foreach (var data in enemyCollection)
    {
        Enemy enemy = (Enemy)data;
        Debug.Log($"{enemy.Name}: {enemy.Hp}");
    }
}
```

これで独自にIEnumerator、IEnumerableインターフェースを実装した反復処理クラスで反復処理をすることができました。

ここまでで十分かと思われるかもしれませんが、もう一歩踏み込んで**再利用性と型の安全性**を学ぶと、さらに理解が深まると思います。

なので、もう少しお付き合いください。

## 再利用性と型の安全性について
### 簡単な例で説明
さきほどのSample3の例で説明する前に、簡単な例で**再利用性と型の安全性**について話していきます。

まず再利用性についてですが、"引数に渡したint型の値をそのまま返すメソッド"を作る場合を考えてみます。

```cs
void Start()
{
    // 引数にint型を渡す。
    var valueInt = GetValueInt(1);
    Debug.Log(valueInt.GetType());
}

// int型を引数にとり、int型を返すメソッド
private int GetValueInt(int value)
{
    return value;
}
```

この例ではGetValueInt()メソッドはint型を引数に取り、戻り値としてint型を返すことができますが、もしstring型や配列を返したいはどうすればよいでしょうか？

下記のように、それぞれ新しくメソッドを定義することもできますが処理が同じなのに、わざわざ違う型で複数のメソッドを定義するのは冗長です。

```cs
void Start()
{
    var valueInt = GetValueInt(1);
    var valueString = GetValueString("hello");
    var valueArray = GetValueArray(new int[] { 1, 2, 3 });
    Debug.Log(valueInt.GetType());
    Debug.Log(valueString.GetType());
    Debug.Log(valueArray.GetType());
}

private int GetValueInt(int value)
{
    return value;
}

private string GetValueString(string value)
{
    return value;
}

private int[] GetValueArray(int[] value)
{
    return value;
}
```

そこで、C#には**ジェネリック**と呼ばれる**特定の型に依存しない汎用的なコードを書く機能**があります。その機能を利用するために型パラメータ\<T>と呼ばれるものを使用します。

下記は上記の例をジェネリックで書き直したものです。
```cs
void Start()
{
    // int型としてGetValue()を呼び出す。
    var valueInt = GetValue<int>(1);
    // string型としてGetValue()を呼び出す。
    var valueString = GetValue<string>("hello");
    // int型の配列としてGetValue()を呼び出す。
    var valueArray = GetValue<int[]>(new int[] { 1, 2, 3 });
    Debug.Log(valueInt.GetType());
    Debug.Log(valueString.GetType());
    Debug.Log(valueArray.GetType());
}

// メソッド名の右隣に<T>を書くことで、<>内の文字列を型の代わりとして戻り値や引数の型に使用できる。
private T GetValue<T>(T value)
{
    return value;
}

// 出力結果
System.Int32
System.String
System.Int32[]
```

上記のプログラムのGetValue\<T>()メソッドは、特定の型に依存せずに使うことができます。
つまり、int型、string型、int型の配列等に対してGetValue\<T>()メソッド1つを定義するだけで済みます。

呼び出し時に、<>内で指定した型のGetValue\<T>()メソッドが呼ばれます。

型パラメータを定義するためにはメソッド名の右隣に\<T>を書くことで、<>内の文字列を型の代わりとして戻り値や引数の型に使用できます。

ちなみに、<>内の文字は何でもよいので、\<U>みたいにすることもできます。おそらくTはTypeから来ているのかと。

今回は、汎用的に使えるメソッド（ジェネリックメソッド）でしたが、クラスや構造体でも同様に使用できます。

例えば、普段よく使うList\<T>クラスでも\<T>に任意の型を設定して、指定した型の要素を持つコレクションを生成することができます。

続いて、2つ目の**型の安全性**についてですが下記の例を見てください。

こちらはArrayListクラスのコレクションを用意して、for文で各要素をint型にキャストするプログラムです。

```cs
private void Start()
{
    ArrayList arrayList = new ArrayList() { 1, 2, "a" };
    for (int i = 0; i < arrayList.Count; i++)
    {
        // object型(string型)からint型にキャストする際にエラーが発生する。
        int value = (int) arrayList[i];
    }
}
```

しかし、コレクションの要素に文字列が含まれているため、文字列をint型にキャストする際に型キャストエラーが発生します。これは"型の安全性が低い"状態です。

そのため下記のように型キャストする前に型チェックをする方法があります。

```cs
private void Start()
{ 
    ArrayList arrayList = new ArrayList() { 1, 2, "a" };
    for (int i = 0; i < arrayList.Count; i++)
    {
        // Int32型でなければスキップする。
        if (arrayList[i].GetType() != typeof(Int32))
            continue;
        // 要素の1, 2のみキャストされる。
        int value = (int) arrayList[i];
    }
}
```

これで型キャストエラーを防ぐことができますが、毎回if文を書くのはめんどくさいです。書き忘れてしまうかもしれません。

ここで、「そもそもint型を使いたいなら、はじめからint型の要素を使えば解決するのでは？」と思いつきます。

つまり、ArrayListクラスのようなobject型の要素ではなく、int型の要素だけを持つコレクションクラスを作ることができれば解決します。

とはいえ、型に合わせて、その都度コレクションクラスを定義するのはめんどくさいです。どうにか1つのコレクションクラスで任意の型に対応できると嬉しいです。

そこで、1つ目に説明したジェネリックの出番です。ジェネリックを使用すれば指定した型を要素に持つコレクションを作成できます。

その代表的なものが、List\<T>型やDictionary\<TKey, TValue>型です。

（配列も要素の型を決められますが、ややこしくなるためここでは割愛させてください。）

### Sample3のプログラムで説明
ここからは、前回やったSample3（独自の反復処理）の例で**再利用性と型の安全性**を見ていきます。

まず、Sample3において、型キャストエラーが発生する場合を考えてみます。

下記のように、呼び出し部分のコレクション生成部分で、Enemy型以外のオブジェクトを入れると型キャストエラーが発生します。

**Sample3**
```cs
void Start()
{
    ArrayList enemies = new ArrayList()
    {
        new Enemy("ゴブリン", 5),
        new Enemy("ドラゴン", 200),
        "1" // // 文字列の要素を追加
    };

    var enemyCollection = new EnemyCollection(enemies);
    // data変数はobject型
    foreach (var data in enemyCollection)
    {
        // 型キャストエラー発生！
        // string型からEnemy型にキャストすることはできない。
        Enemy enemy = (Enemy)data;
        Debug.Log($"{enemy.Name}: {enemy.Hp}");
    }
}
```

そのため、object型を要素に持つArrayListクラスのコレクションではなく、要素の型が保障されているコレクションを渡してあげれば良さそうです。

下記に修正したプログラムを示します。Sample4ではArrayListクラスからList\<Enemy>クラスのコレクションに変更しています。

**Sample4.呼び出し部分（Mainクラス）**
```cs
// Enemyクラスは前回と同じため省略しています。
public class Sample4 : MonoBehaviour
{
    void Start()
    {
        // 変更: ArrayList -> List<Enemy>
        List<Enemy> enemies = new List<Enemy>()
        {
            new Enemy("ゴブリン", 5),
            new Enemy("ドラゴン", 200),
        };

        var enemyCollection = new EnemyCollection(enemies);
        foreach (var data in enemyCollection)
        {
            Enemy enemy = (Enemy)data;
            Debug.Log($"{enemy.Name}: {enemy.Hp}");
        }
    }
}
```

**Sample4.EnemyCollection**
```cs
public class EnemyCollection : IEnumerable
{
    // 変更: ArrayList -> List<Enemy>
    private readonly List<Enemy> enemies;

    // 変更: ArrayList -> List<Enemy>
    public EnemyCollection(List<Enemy> enemies)
    {
        // 変更: List<Enemy>のコピー
        this.enemies = new List<Enemy>(enemies);
    }

    public IEnumerator GetEnumerator()
    {
        return (IEnumerator) new EnemyEnumerator(enemies);
    }
}
```

**Sample4.EnemyEnumerator**
```cs
public class EnemyEnumerator : IEnumerator
{
    // 変更: ArrayList -> List<Enemy>
    private readonly List<Enemy> enemies;
    private int index;

    // 変更: ArrayList -> List<Enemy>
    public EnemyEnumerator(List<Enemy> enemies)
    {
        // 変更: List<Enemy>のコピー
        this.enemies = new List<Enemy>(enemies);
        index = -1;
    }

    public bool MoveNext()
    {
        index++;
        return index < enemies.Count;
    }

    public void Reset()
    {
        index = -1;
    }

    public object Current => enemies[index];
}
```

これで型キャストエラーを防ぐことができました。ただし、このプログラムは少し使いづらいです。

例えば、EnemyAクラスとEnemyBクラスを定義して、EnemyA、EnemyB型のコレクションで、それぞれ反復処理を行う場合を考えてみてください。

EnemyA型を要素に持つコレクションでEnemyCollectionA、EnemyEnumeratorAクラスを用意して、EnemyB型を要素に持つコレクションでもEnemyCollectionB、EnemyEnumeratorBクラスをそれぞれ用意する必要があります。

この時、同じ反復処理をする場合は内部コレクションの要素の型が違うだけで他の実装は同じです。つまり冗長です。

そのため、冒頭で紹介した**ジェネリック**を使用します。

ジェネリックを使用することで、EnemyA、EnemyB型をそれぞれ要素に持つコレクションを同じEnemyCollection、EnemyEnumeratorクラスで処理することができます。

下記のプログラムは、Sample4をジェネリックで書き直したものです。

**Sample5.EnemyCollection**
```cs
// 変更: 型パラメータ定義
public class EnemyCollection<T> : IEnumerable
{
    // 変更: 型パラメータ使用
    private readonly List<T> enemies;
    
    // 変更: 型パラメータ使用
    public EnemyCollection(List<T> enemies)
    {
        // 変更: 型パラメータ使用
        this.enemies = new List<T>(enemies);
    }

    public IEnumerator GetEnumerator()
    {
        // 変更: 型パラメータを指定して呼び出す
        return (IEnumerator) new EnemyEnumerator<T>(enemies);
    }
}
```

**Sample5.EnemyEnumerator**
```cs 
// 変更: 型パラメータ定義
public class EnemyEnumerator<T> : IEnumerator
{
    // 変更: 型パラメータ使用
    private readonly List<T> enemies;
    private int index;

    // 変更: 型パラメータ使用
    public EnemyEnumerator(List<T> enemies)
    {
        // 変更: 型パラメータ使用
        this.enemies = new List<T>(enemies);
        index = -1;
    }

    public bool MoveNext()
    {
        index++;
        return index < enemies.Count;
    }

    public void Reset()
    {
        index = -1;
    }

    public object Current => enemies[index];
}
```

**Sample5.呼び出し部分**
```cs
void Start()
{
    // EnemyA型を要素とするコレクションの反復処理
    List<EnemyA> enemiesA = new List<EnemyA>()
    {
        new EnemyA("ゴブリン", 5),
        new EnemyA("ドラゴン", 200),
    };
    var enemyCollectionA = new EnemyCollection<EnemyA>(enemiesA);
    foreach (var data in enemyCollectionA)
    {
        EnemyA enemy = (EnemyA)data;
        Debug.Log($"{enemy.Name}: {enemy.Hp}");
    }
    
    // EnemyB型を要素とするコレクションの反復処理
    List<EnemyB> enemiesB = new List<EnemyB>()
    {
        new EnemyB("ピクシー", 5),
        new EnemyB("オーガ", 100),
    };
    var enemyCollectionB = new EnemyCollection<EnemyB>(enemiesB);
    foreach (var data in enemyCollectionB)
    {
        EnemyB enemy = (EnemyB)data;
        Debug.Log($"{enemy.Name}: {enemy.Hp}");
    }
}
```

**Sample5.EnemyA、EnemyB**
```cs
// EnemyA、EnemyBクラスはEnemyクラスと同じ内容です。
public class EnemyA
{
    public string Name { get; private set; }
    public int Hp { get; private set; }

    public EnemyA(string name, int hp)
    {
        Name = name;
        Hp = hp;
    }
}

public class EnemyB
{
    public string Name { get; private set; }
    public int Hp { get; private set; }

    public EnemyB(string name, int hp)
    {
        Name = name;
        Hp = hp;
    }
}
```

これで、要素の型が異なるコレクションでも同じ反復処理クラス（EnemyCollection、EnemyEnumerator）で処理ができます。

少し複雑になってきたので一旦ここまでをまとめます。
* コレクションの要素の型を決めることで、指定した型における型キャストエラーを防ぐことができる。
* コレクションの要素の型は異なるが同じ反復処理を行いたい場合は、ジェネリックを使用することで、同じ反復処理クラスで対応することができる。

## ジェネリックコレクションに対する反復処理

ここまでで型の安全性と再利用性を向上させることができましたが、

Microsoftさんのドキュメントを見ると、これまで使っていた[IEnumerable](https://learn.microsoft.com/ja-jp/dotnet/api/system.collections.ienumerable?view=net-7.0)、[IEnumerator](https://learn.microsoft.com/ja-jp/dotnet/api/system.collections.ienumerator?view=net-7.0)インターフェースは、どうやら非ジェネリックコレクションに対する反復処理に使用するそうです。

非ジェネリックコレクションとジェネリックコレクションの違いは、要素の型を指定しているかどうかです。

例えば、Sample3で使用していたArrayListクラスのコレクションは要素の型を指定せずに要素を追加していたので非ジェネリックコレクションになります。

一方、Sample4、5で使用していたList\<T>クラスのコレクションは生成時に要素の型を指定していたので、ジェネリックコレクションになります。

ジェネリックコレクションに対する反復処理を行う場合は、[IEnumerator\<T>](https://learn.microsoft.com/ja-jp/dotnet/api/system.collections.generic.ienumerator-1?view=net-7.0)、[IEnumerable\<T>](https://learn.microsoft.com/ja-jp/dotnet/api/system.collections.generic.ienumerable-1?view=net-7.0)インターフェースを使用します。

つまり、Sample5ではジェネリックコレクションを使用しているのでIEnumerator\<T>、IEnumerable\<T>インターフェースに変更することが必要です。

とはいえ、「今のままでも安全で汎用的に使えているから、IEnumerator\<T>、IEnumerable\<T>インターフェースに変更する必要はないのでは？」と疑問に思うかもしれません。

実際にSample5ではジェネリックコレクションに対する反復処理でIEnumerator、IEnumerableインターフェースを使用していますが、エラーなく実行ができます。

では、なぜIEnumerator\<T>、IEnumerable\<T>インターフェースを使用するのか、というと型の安全性をより高めてパフォーマンスを向上させることができるからです。

具体的には、値型（構造体）をobject型にアップキャスト（ボックス化）した変数をobject型から値型にダウンキャスト（アンボックス化）する場合にパフォーマンスの差が出ます。

言葉だと分かりづらいのでSample5を元に、

* IEnumerator、IEnumerableインターフェースを実装したEnemyCollection\<T>、EnemyEnumerator\<T>クラス
* IEnumerator\<T>、IEnumerable\<T>インターフェースを実装したEnemyCollection\<T>、EnemyEnumerator\<T>クラス

の2つの場合で簡単な計測をして、それぞれのパフォーマンスを見てみます。

下記のプログラムは1,000,000個の要素を格納したコレクションをforeach文内でobject型からEnemy型にキャストする際の処理時間を計測するプログラムです。

後ほど、IEnumerator\<T>、IEnumerable\<T>インターフェースを実装したクラスについて説明をするのでここでは一旦、「こんな感じで計測したんだ」ぐらいの気持ちでみていただければと。

補足ですが、アンボックス化は参照型から値型に変換する処理のためEnemyクラスではなくEnemy構造体型を使用しています。

```cs
void CalculatePerformance()
{
    int count = 1000000;
    List<Enemy> enemies = new List<Enemy>();
    for (int i = 0; i < count; i++)
    {
        enemies.Add(new Enemy("dummy", i));
    }
    var enemyCollection = new EnemyCollection<Enemy>(enemies);
    // 計測開始
    var startTime = Time.realtimeSinceStartup;
    foreach (var data in enemyCollection)
    {
        // アンボックス化
        Enemy enemy = (Enemy) data;
    }
    // 計測終了
    var elapsedTime = Time.realtimeSinceStartup - startTime;
    Debug.Log($"処理時間: {elapsedTime}s");
}
```

```cs
// ※今回は値型であるStructを定義しています。
public struct Enemy
{
    public string Name { get; private set; }
    public int Hp { get; private set; }

    public Enemy(string name, int hp)
    {
        Name = name;
        Hp = hp;
    }
}
```

それぞれの処理時間の結果を比較すると、
|IEnumerator、IEnumerableの場合|IEnumerator\<T>、IEnumerable\<T>の場合|
|:-----------------:|:--------------------:|
|処理時間: 0.1573495s|処理時間: 0.06405878s|

となりました。

私の環境では2倍近いパフォーマンスの差が出ていますが、簡単な計測のため、「そうなんだ」ぐらいの認識でお願いします。

この結果から、アンボックス化がパフォーマンスに影響を与えることがわかりました。

実はIEnumerator\<T>、IEnumerable\<T>インターフェースを使用すると、下記の図のようにforeach文の"data"変数でobject型ではなく、Enemy構造体型が格納されていることがわかります。

そのため、アンボックス化によるパフォーマンスの低下が起こりません。

![型推定の比較](./Img/%E5%9E%8B%E6%8E%A8%E5%AE%9A%E6%AF%94%E8%BC%83.png "型推定の比較")

なぜ、"data"変数にobject型ではなくEnemy構造体型が格納されているのかはIEnumerator\<T>インターフェースの定義を見てみるとわかります。

```cs
public interface IEnumerator<out T> : IEnumerator, IDisposable
{
    // IEnumeratorインターフェースではobject型。
    T Current { get; }
}
```

IEnumerator\<T>インターフェースのCurrentプロパティの戻り値がobject型ではなく"T"型となっています。

そのため、呼び出し側で指定した型がCurrentプロパティの戻り値の型となります。今回はEnemy構造体型がCurrentプロパティから返されます。

以上を踏まえて、Sample5のEnemyCollection、EnemyEnumeratorクラスをIEnumerator\<T>、IEnumerable\<T>インターフェースを用いて書き直してみます。

**Sample6.EnemyCollection**
```cs
// 変更: IEnumerable -> IEnumerable<T>
// IEnumerable<T>にはジェネリックの型パラメータを設定。
public class EnemyCollection<T> : IEnumerable<T>
{
    private readonly List<T> enemies;

    public EnemyCollection(List<T> enemies)
    {
        this.enemies = new List<T>(enemies);
    }

    // 変更: IEnumerable<T>インターフェースのGetEnumerator()メソッドの追加
    public IEnumerator<T> GetEnumerator()
    {
        return new EnemyEnumerator<T>(enemies);
    }

    // 変更: IEnumerableインターフェースのGetEnumerator()メソッドが呼ばれたときの処理を追加
    IEnumerator IEnumerable.GetEnumerator()
    {
        return GetEnumerator();
    }
}
```

**Sample6.EnemyEnumerator**
```cs
// 変更: IEnumerator -> IEnumerator<T>
// IEnumerator<T>にはジェネリック型が適用されている。
public class EnemyEnumerator<T> : IEnumerator<T>
{
    private readonly List<T> enemies;
    private int index;

    public EnemyEnumerator(List<T> enemies)
    {
        this.enemies = new List<T>(enemies);
        index = -1;
    }

    public bool MoveNext()
    {
        index++;
        return index < enemies.Count;
    }

    public void Reset()
    {
        index = -1;
    }

    // 変更: IEnumerator<T>インターフェースのCurrentプロパティを追加
    public T Current => enemies[index];
    
    // 変更: IEnumeratorインターフェースのCurrentプロパティが呼ばれたときの処理を追加
    object IEnumerator.Current => Current;

    // 変更: リソースの解放メソッドを追加。今回はファイルやデータベース等は使用しないので中身はなしです。
    public void Dispose() { }
}
```

ここで抑えておきたいポイントは2つあります。
1つ目は、IEnumerator\<T>、IEnumerable\<T>インターフェースで定義されているメソッド、プロパティが追加されている点です。

IEnumerable\<T>インターフェースでは、IEnumerator\<T>型を戻り値とするGetEnumerator()メソッドが定義されており、IEnumerator\<T>インターフェースでは、\<T>型を戻り値とするCurrentプロパティが定義されています。

また、IEnumerator\<T>インターフェースにはIDisposableインターフェースも実装されているため、リソースの解放を行うDispose()メソッドも追加されています。

2つ目は同名のメソッドが定義されている点です。
EnemyCollection\<T>クラスでは、IEnumerable\<T>インターフェースのGetEnumerator()メソッドとIEnumerableインターフェースのGetEnumerator()メソッドの2つが実装されています。

EnemyEnumerator\<T>クラスでも、IEnumerator\<T>インターフェースのCurrentプロパティとIEnumeratorインターフェースのCurrentプロパティの2つが実装されています。

なぜ、わざわざ同名のプロパティ、メソッドが用意されているかというと、ジェネリック、非ジェネリックコレクションの両方に対応するためです。

例えば下記のように、IEnumerable\<T>オブジェクトをIEnumerable型にキャストした際に、IEnumerable\<T>.GetEnumerator()メソッドではなく、IEnumerable.GetEnumerator()メソッドが呼ばれます。

```cs
void Start()
{
    List<Enemy> enemies = new List<Enemy>()
    {
        new Enemy("ゴブリン", 5),
        new Enemy("ドラゴン", 200),
    };
    
    // ジェネリックコレクションに対する反復処理を非ジェネリックコレクションに対する反復処理にする。
    IEnumerable enemyCollection = (IEnumerable) new EnemyCollection<Enemy>(enemies);
    
    // IEnumerableインターフェースのGetEnumerator()メソッドが呼ばれる。
    IEnumerator enumerator = enemyCollection.GetEnumerator();
    
    // Currentプロパティでobject型のオブジェクトを取得する。
    foreach (var data in enemyCollection)
    {
        // 何かの処理
    }
}
```

"IEnumerable.GetEnumerator() {}"という書き方は見慣れないかもしれませんが、c#には同名のメソッドを定義するために[明示的に記述して区別する方法](https://learn.microsoft.com/ja-jp/dotnet/csharp/programming-guide/interfaces/explicit-interface-implementation)があります。

補足として、Sample6では呼び出し側で型を指定していましたが、Currentプロパティから返す型が決まっている場合は、下記のように反復処理クラスの定義で型パラメータを設定することもできます。

```cs
public class EnemyEnumerator : IEnumerator<Enemy> {}  
```

これでIEnumerator、IEnumerableインターフェースについての説明は終わりです。

"コルーチンがどう関係しているか"については後の章で触れますので一旦ここまでをまとめます。

* IEnumerator、IEnumerableインターフェースはコレクションの反復処理をするときに使われる。
* ジェネリックを使うことで任意の型で処理を行うことができる。（再利用性）
* 内部コレクションにジェネリックコレクション（List\<T>等）を使うことで型の安全性を高めることができる。
* ジェネリックコレクションに対して反復処理を行う場合は、IEnumerator\<T>、IEnumerable\<T>インターフェースを使用する。
* IEnumerator\<T>、IEnumerable\<T>インターフェースを実装したクラスや構造体はジェネリック、非ジェネリックコレクションの両方に対応している。

<br>

# yieldステートメントについて
## yieldステートメントを使うメリット
続いて、yieldステートメントについて説明していきます。

yieldステートメントの定義や挙動の説明の前に、まずはyieldステートメントを使うメリットについて見ていきます。

### メリット1: コードを簡潔にできる
これまで独自の反復処理を実装したクラスを作成するためにIEnumerable、IEnumeratorインターフェースを実装する必要がありました。

しかし簡単な反復処理をするだけなら冗長に感じてしまいます。そこで、C#には簡単に反復処理ができるように**yieldステートメント**が用意されています。

下記のようにyieldステートメントを使用すると、IEnumeratorインターフェースを実装したクラスを定義する必要がなくなります。

yieldステートメントについては後ほどするので、ここではEnemyEnumeratorクラスがなくなっていることだけ確認してください。

**Sample7.呼び出し部分**
```cs
public class Sample7 : MonoBehaviour
{
    void Start()
    {
        // EnemyA型を要素とするコレクションの反復処理
        List<EnemyA> enemiesA = new List<EnemyA>()
        {
            new EnemyA("ゴブリン", 5),
            new EnemyA("ドラゴン", 200),
        };
        
        var enemyCollectionA = new EnemyCollection<EnemyA>(enemiesA);
        foreach (var data in enemyCollectionA)
        {
            EnemyA enemy = (EnemyA)data;
            Debug.Log($"{enemy.Name}: {enemy.Hp}");
        }
        
        // EnemyB型を要素とするコレクションの反復処理
        List<EnemyB> enemiesB = new List<EnemyB>()
        {
            new EnemyB("ピクシー", 5),
            new EnemyB("オーガ", 100),
        };
        var enemyCollectionB = new EnemyCollection<EnemyB>(enemiesB);
        foreach (var data in enemyCollectionB)
        {
            EnemyB enemy = (EnemyB)data;
            Debug.Log($"{enemy.Name}: {enemy.Hp}");
        }
    }
}
```

**Sample7.EnemyCollection**
```cs
public class EnemyCollection<T> : IEnumerable<T>
{
    private readonly List<T> enemies;

    public EnemyCollection(List<T> enemies)
    {
        this.enemies = new List<T>(enemies);
    }

    public IEnumerator<T> GetEnumerator()
    {
        // IEnumeratorオブジェクトを生成して返すのではなく、yieldステートメントを使用。
        foreach (var enemy in enemies)
        {
            yield return enemy;
        }
    }

    IEnumerator IEnumerable.GetEnumerator()
    {
        return GetEnumerator();
    }
}
```

**Sample7.EnemyA、EnemyB**
```cs
public class EnemyA
{
    public string Name { get; private set; }
    public int Hp { get; private set; }

    public EnemyA(string name, int hp)
    {
        Name = name;
        Hp = hp;
    }
}

public class EnemyB
{
    public string Name { get; private set; }
    public int Hp { get; private set; }

    public EnemyB(string name, int hp)
    {
        Name = name;
        Hp = hp;
    }
}
```

### メリット2: 反復処理中のメモリ使用量を少なくできる
2つ目のメリットは、yieldステートメントを使うことで反復処理中のメモリ使用量を少なくすることができる点です。

下記は、ボタンをクリックした時に反復処理が行われるプログラムです。

それぞれyieldステートメントを使用した場合と使用しない場合でメモリ使用量を計測しました。（Enemyクラスは省略しています。）

**Sample8.yieldステートメントを使用しない場合**
```cs
public class Sample8 : MonoBehaviour
{
    // ボタンをクリックする度に反復処理を実行する。
    public void MyOnClick()
    {
        var myCollection = new MyCollection();
        foreach (var data in myCollection)
        {
            var i = data;
        }
    }
}

public class MyCollection : IEnumerable
{
    public IEnumerator GetEnumerator()
    {
        return new MyEnumerator();
    }
}

public class MyEnumerator : IEnumerator
{  
    // メモリ上に全ての要素を展開する。
    private readonly List<Enemy> enemies = new List<Enemy>();
    private int index;

    public MyEnumerator()
    {
        for (int i = 0; i < 2000000; i++)
        {
            enemies.Add(new Enemy("ゴブリン", 1));
        }
        index = -1;
    }
    public bool MoveNext()
    {
        index++;
        return index < enemies.Count;
    }
    public void Reset()
    {
        index = -1;
    }
    public object Current => enemies[index];
}
```

**Sample8.yieldステートメントを使用する場合**
```cs
public class Sample8 : MonoBehaviour
{
    public void MyOnClick()
    {
        var myYieldCollection = new MyYieldCollection();
        foreach (var data in myYieldCollection)
        {
            var i = data;
        }    
    }
}

public class MyYieldCollection : IEnumerable
{
    public IEnumerator GetEnumerator()
    {
        for (int i = 0; i < 2000000; i++)
        {
            // 必要な要素だけメモリ上に展開する。
            yield return new Enemy("ゴブリン", 1);
        }
    }
}
```

それぞれ計測すると、以下のような結果になりました。

|yieldステートメント使用しない|yieldステートメント使用する|
|:-----------------:|:--------------------:|
|常時: 500MB|常時: 442MB|
|ボタンクリック時: 660MB|ボタンクリック時: 590MB|

簡単な計測ではありますが、yieldステートメントを使用した場合の方が、常時、ボタンクリック時共にメモリ使用量が少なくなりました。

おそらく、yieldステートメントを使用しない場合は、ボタンをクリックするたびにコレクションの要素を全てメモリに展開しますが、yieldステートメントを使用する場合は適宜必要な要素のみメモリに展開するためメモリ使用量が少なくなるからだと考えられます。

## yieldステートメントと反復子の関係
yieldステートメントについて[公式ドキュメント](https://learn.microsoft.com/ja-jp/dotnet/csharp/language-reference/statements/yield)の説明を見てみると、

> 反復子で yield ステートメントを使うと、シーケンスを反復処理するときにシーケンスの次の値を指定できます。

と書かれています。

反復子とは、シーケンス（コレクションよりも順序を意識したデータ集合です）の要素を1つずつ見ていく機能を持つオブジェクトです。つまりIEnumeratorオブジェクトのことです。

そして、"シーケンスの次の値を指定できます"とはIEnumeratorオブジェクトのMoveNext()メソッドとCurrentプロパティの実装部分の事で、この部分がyieldステートメントで代用できます。そのため、EnemyEnumeratorクラスを定義する必要がありませんでした。

また、**yieldステートメントを使って反復子を返すメソッドのことを反復子メソッド**と呼びます。

下記のように、Mainクラス内に反復子メソッドを定義することでEnemyCollectionクラスもいらなくなります。

**Sample9**
```cs
// EnemyA、EnemyBクラスは省略しています。
public class Sample9 : MonoBehaviour
{   
    // 反復子メソッドの定義
    private IEnumerator<T> GetYieldEnumerator<T>(List<T> enemies)
    {
        foreach (var enemy in enemies)
        {
            yield return enemy;
        }
    }
    
    void Start()
    {
        List<EnemyA> enemiesA = new List<EnemyA>()
        {
            new EnemyA("ゴブリン", 5),
            new EnemyA("ドラゴン", 200),
        };

        var enumeratorA = GetYieldEnumerator<EnemyA>(enemiesA);
        while (enumeratorA.MoveNext())
        {
            Debug.Log($"HP: {enumeratorA.Current?.Hp}, {enumeratorA.Current?.Name}");
        }

        List<EnemyB> enemiesB = new List<EnemyB>()
        {
            new EnemyB("ピクシー", 5),
            new EnemyB("オーガ", 100),
        };

        var enumeratorB = GetYieldEnumerator<EnemyB>(enemiesB);
        while (enumeratorB.MoveNext())
        {
            Debug.Log($"HP: {enumeratorB.Current?.Hp}, {enumeratorB.Current?.Name}");
        }
    }
}
```

反復子メソッドの戻り値はIEnumerator\<T>型で定義されていますが、下記のようにIEnumerable\<T>型でも定義することができます。（IEnumerable、IEnumerator型も可能です。）

IEnumerable、IEnumerable\<T>型を戻り値とするメソッドを反復子メソッドと呼ぶかは公式ドキュメントを見てもハッキリとは分かりませんでした...

```cs
// IEnumerable<T>型を戻り値とするためforeach文で反復処理を行うことを想定。
private IEnumerable<T> GetYieldEnumerator<T>(List<T> enemies)
{
    foreach (var enemy in enemies)
    {
        yield return enemy;
    }
}
```

また、IEnumerator、IEnumerator\<T>型を戻り値とする場合は"foreach文が使えない"ことと"同じ反復子を使用する"ことに注意してください。

```cs
private IEnumerator<T> GetYieldEnumerator<T>(List<T> enemies)
{
    foreach (var enemy in enemies)
    {
        yield return enemy;
    }
}

// ✖: foreach文ではコンパイルエラーが出る。
foreach (var data in GetYieldEnumerator<EnemyB>(enemiesB)) { }

// 〇
var enumerator = GetYieldEnumerator<EnemyB>(enemiesB);
while (enumerator.MoveNext()) { }
```

下記のように、"enumerator"変数を用意せずに、GetYieldEnumerator()メソッドを呼ぶと、呼ばれる度に新しい反復子が生成されてしまい意図した結果になりません。

※下記のコードは無限ループしますので気を付けてください。

```cs
void Start() 
{
    // 新しい反復子を毎回生成するため、常にtrueになる。
    while (GetYieldEnumerator<EnemyB>(enemiesB).MoveNext())
    {
        // 新しい反復子を生成して、インデックスを進めずにCurrentプロパティを呼ぶため空が返る。
        Debug.Log($"{GetYieldEnumerator<EnemyB>(enemiesB).Current}");
    }
}
```

（反復子？）メソッドの戻り値の型をIEnumerable、IEnumerator型のどちらにするかは、

* foreach文を使用する: IEnumerable、IEnumerable\<T>型を戻り値にする。
* foreach文を使用しない: IEnumerator、IEnumerator\<T>型を戻り値にする。

を基準に考えると良いと思います。

## yieldステートメントの書き方
### yield return文とyield break文
続いては、yieldステートメントの書き方について説明します。

yieldステートメントにはyield return文とyield break文の2つがあります。それぞれ簡単に紹介します。

yield return文は、シーケンスで次にどの要素を返すのかを決めています。

例えば、下記のプログラムは1から3を要素として持つシーケンスの反復子を返すメソッド（反復子メソッド）を定義しています。

**Sample10**
```cs
public class Sample10 : MonoBehaviour
{
    private IEnumerator GetYieldEnumerator()
    {
        yield return 1;
        yield return 2;
        yield return 3;
    }
    
    void Start()
    {
        var enumerator = GetYieldEnumerator();
        while (enumerator.MoveNext())
        {
            Debug.Log($"{enumerator.Current}");
        }
    }
}

// 出力結果
1
2
3
```

yield return文で上から指定された要素の順にシーケンスの次の要素として返されます。

次に、yield break文は反復処理を途中でやめることができます。下記のプログラムは"3"の要素を返す前に反復処理をやめています。

**Sample11**
```cs
public class Sample11 : MonoBehaviour
{
    private IEnumerator GetYieldEnumerator()
    {
        yield return 1;
        yield return 2;
        // "3"の要素を返す前に反復処理をやめる。
        yield break;
        yield return 3;
    }

    private void Start()
    {
        var enumerator = GetYieldEnumerator();
        while (enumerator.MoveNext())
        {
            Debug.Log($"{enumerator.Current}");
        }
    }
}

// 出力結果
1
2
```

### yield return文で返すオブジェクトについて

Sample10、11ではyield return文でint型の要素を返していましたが他にもメソッドやインスタンスを指定できます。

**Sample12.メソッドの場合**
```cs
public class Sample12 : MonoBehaviour
{
    private IEnumerator GetYieldEnumerator()
    {
        yield return FuncA();
        yield return FuncB();
        yield return FuncC();
    }
    
    void Start()
    {
        var enumerator = GetYieldEnumerator();
        while (enumerator.MoveNext())
        {
            Debug.Log($"メソッドの戻り値は{enumerator.Current}");
        }
    }

    private int FuncA()
    {
        Debug.Log("FuncA()メソッドを実行");
        return 1;
    }

    private int FuncB()
    {
        Debug.Log("FuncB()メソッドを実行");
        return 2;
    }

    private int FuncC()
    {
        Debug.Log("FuncC()メソッドを実行");
        return 3;
    }
}

// 出力結果
FuncA()メソッドを実行
メソッドの戻り値は1
FuncB()メソッドを実行
メソッドの戻り値は2
FuncC()メソッドを実行
メソッドの戻り値は3
```

実行すると、FuncA()、FuncB()、FuncC()メソッドの順番で実行されます。そして、各メソッドの戻り値がdata変数に代入されて表示されます。

ただし、yield return文で実行するメソッドにvoid型を戻り値にするメソッドを指定することはできません。

反復子メソッドでvoid型の戻り値を持つメソッドを実行する場合は、Sample13のようにyield return文を使用せずに呼び出せば大丈夫です。

ただし、yield return文で返す値がないと困るため何も返すものがない場合はyield return null;を記述します。

**Sample13.メソッドの場合**
```cs
public class Sample13 : MonoBehaviour
{
    private IEnumerator GetYieldEnumerator()
    {
        FuncA();
        FuncB();
        FuncC();
        // nullを返す。
        yield return null;
    }
    
    void Start()
    {
        var enumerator = GetYieldEnumerator();
        while (enumerator.MoveNext())
        {
            Debug.Log($"メソッドの戻り値は{enumerator.Current}");
        }
    }
    
    private void FuncA()
    {
        Debug.Log("FuncA()メソッドを実行");
    }

    private void FuncB()
    {
        Debug.Log("FuncB()メソッドを実行");
    }

    private void FuncC()
    {
        Debug.Log("FuncC()メソッドを実行");
    }
}

// 出力結果
FuncA()メソッドを実行
FuncB()メソッドを実行
FuncC()メソッドを実行
メソッドの戻り値は
```

次は、yield return文にインスタンスを設定する場合です。

下記のプログラムではyield return文にIEnumerableインタフェースを実装したInnerSample14クラスのインスタンスを設定することで、InnerSample14.GetEnumerator()メソッドが実行されることが期待されています。

**Sample14**
```cs
public class InnerSample14 : IEnumerable
{
    private int InnerFuncA()
    {
        Debug.Log("InnerFuncA()メソッドを実行");
        return 2;
    }

    public IEnumerator GetEnumerator()
    {
        yield return InnerFuncA();
    }
}

public class Sample14 : MonoBehaviour
{
    void Start()
    {
        var enumerator = GetYieldEnumerator();
        while (enumerator.MoveNext())
        {
            Debug.Log($"メソッドの戻り値は{enumerator.Current}");
        }
    }

    private IEnumerator GetYieldEnumerator()
    {
        yield return FuncA();
        yield return new InnerSample14();
    }

    private int FuncA()
    {
        Debug.Log("FuncA()メソッドを実行");
        return 1;
    }
}

// 出力結果
FuncA()メソッドを実行
メソッドの戻り値は1
メソッドの戻り値はInnerSample14
```

一見上手く行きそうですが、出力結果を見るとInnerSample14オブジェクトのInnerFuncA()メソッドは呼ばれずにInnerSample14オブジェクト自体が返されています。

InnerFuncA()メソッドを実行するには下記のように反復処理を実行する処理を2つ記述する必要があります。

**Sample15**
```cs
// InnerSample15はInnerSample16と同じ内容のため省略。
public class Sample15 : MonoBehaviour
{
    void Start()
    {
        // 1つ目: 反復子を取得して反復処理を行う。
        var enumerator = GetYieldEnumerator();
        while (enumerator.MoveNext())
        {
            Debug.Log($"メソッドの戻り値は{enumerator.Current}");
        }
    }

    private IEnumerator GetYieldEnumerator()
    {
        yield return FuncA();
        // 2つ目: 反復子を取得して反復処理を行う。
        var enumerator = new InnerSample15().GetEnumerator();
        while (enumerator.MoveNext())
        {
            yield return enumerator.Current;
        }
    }

    private int FuncA()
    {
        Debug.Log("FuncA()メソッドを実行");
        return 1;
    }
}

// 出力結果
FuncA()メソッドを実行
メソッドの戻り値は1
InnerFuncA()メソッドを実行
メソッドの戻り値は2
```

また、IEnumeratorインタフェースを実装したクラスのインスタンスでも検証してみます。

**Sample16**
```cs
public class Sample16 : MonoBehaviour
{
    private void Start()
    {
        var enumerator = GetYieldEnumerator();
        while (enumerator.MoveNext())
        {
            Debug.Log($"メソッドの戻り値は{enumerator.Current}");
        }
    }

    private IEnumerator GetYieldEnumerator()
    {
        yield return FuncA();
        yield return new InnerSample16();
    }
    
    private int FuncA()
    {
        Debug.Log("FuncA()メソッドを実行");
        return 1;
    }
}

public class InnerSample16 : IEnumerator
{
    private int index;
    // デリゲートにメソッドを格納。
    private readonly Func<int>[] actions =
    {
        InnerFuncA,
    };
    
    private static int InnerFuncA()
    {
        Debug.Log("InnerFuncA()メソッドを実行");
        return 2;
    }

    public InnerSample16()
    {
        index = -1;
    }
    
    public bool MoveNext()
    {
        index++;
        return index < actions.Length;
    }

    public void Reset()
    {
        index = -1;
    }

    // Currentプロパティ呼び出し時にメソッドを実行して値を返す。
    public object Current => actions[index].Invoke();
}

// 出力結果
FuncA()メソッドを実行
メソッドの戻り値は1
メソッドの戻り値はInnerSample16
```

こちらも同様にInnerSample16オブジェクトのInnerFuncA()メソッドは呼ばれずにInnerSample16オブジェクト自体が返されています。そのため反復処理を実行する処理を2つ記述します。

**Sample17**
```cs
public class Sample17 : MonoBehaviour
{
    private void Start()
    {
        // 1つ目: 反復子を取得して反復処理を行う。
        var enumerator = GetYieldEnumerator();
        while (enumerator.MoveNext())
        {
            Debug.Log($"メソッドの戻り値は{enumerator.Current}");
        }
    }

    private IEnumerator GetYieldEnumerator()
    {
        yield return FuncA();
        // 2つ目: 反復子を取得して反復処理を行う。
        var enumerator = new InnerSample17();
        while (enumerator.MoveNext())
        {
            yield return enumerator.Current;
        }
    }
    
    private int FuncA()
    {
        Debug.Log("FuncA()メソッドを実行");
        return 1;
    }
}

// 出力結果
FuncA()メソッドを実行
メソッドの戻り値は1
InnerFuncA()メソッドを実行
メソッドの戻り値は2
```

# ここまでの内容でコルーチンの疑問を解消する
いよいよ本記事の冒頭で取り上げた下記の疑問について話していきます。

* なぜコルーチンの戻り値の型はIEnumerator型でなければならないのか？
* StartCoroutine()メソッドは何をしているのか？
* コルーチンでreturn文ではなく、yield return文を使う理由は？

## なぜコルーチンの戻り値の型はIEnumerator型でなければならないのか？
"非同期処理であるコルーチン"と"反復処理で使われるIEnumerator"がどう関係しているのかが鍵になります。

コルーチンは冒頭でも説明しましたが、**処理の途中で一時中断して、再度中断した位置から処理を再開できる機能を持つ関数**のことです。

この **"処理の途中で一時中断して、再度中断した位置から処理を再開できる機能"** の部分は、まさに反復処理で使われている反復子（要素を1つずつ見ていく機能を持つオブジェクト）の仕組みと同じです。

つまり、非同期処理を制御するためには「現在どこまで処理が行われているのか？」、「次に何の処理をするのか？」を保持する必要があるため、int型やstring型、void型は使用できません。

そして、MyCoroutine()メソッドの戻り値の型がIEnumerable型ではなくIEnumerator型である理由は、コルーチンを実行する時にMoveNext()メソッドを使用しているからです。

Unity-Technologiesさんが公開しているUnityのソースコード: [コルーチンの箇所](https://github.com/Unity-Technologies/UnityCsReference/blob/master/Runtime/Export/Scripting/Coroutines.cs)を参照させていただくと、InvokeMoveNext()メソッド内で引数として渡されたIEnumeratorオブジェクトのMoveNext()メソッドが呼ばれていることがわかります。

そのため、foreach文を使用するIEnumerable型ではなくIEnumerator型が使用されます。

```cs
// このプログラムは、Unity-Technologies社が公開しているソースコードの一部を引用しています。
[RequiredByNativeCode]
[System.Security.SecuritySafeCritical]
unsafe static public void InvokeMoveNext(IEnumerator enumerator, IntPtr returnValueAddress)
{
    if (returnValueAddress == IntPtr.Zero)
        throw new ArgumentException("Return value address cannot be 0.", "returnValueAddress");
    (*(bool*)returnValueAddress) = enumerator.MoveNext();
}
```

また、コルーチンの型をIEnumerator\<T>型にしても実行できますが、そもそもIEnumerator\<T>型は型安全のために使用しているので複数の型の値を返す場合は使うメリットがないのかなと...

**Sample18**
```cs
public class Sample18 : MonoBehaviour
{
    // 戻り値としてobject型を指定。
    IEnumerator<object> MyCoroutine()
    {
        Debug.Log("3秒待機処理開始");
        // Unity.WaitForSeconds型を返す。
        yield return new WaitForSeconds(3f);
        Debug.Log("3秒待機処理終了");
        // System.Int32型を返す。
        yield return FuncA();
    }

    private void Start()
    {
        StartCoroutine(MyCoroutine());
    }

    private int FuncA()
    {
        Debug.Log("FuncA()メソッドを実行");
        return 1;
    }
}
```

## StartCoroutine()メソッドは何をしているのか？
説明のために"3秒待機させて任意の処理を実行するコルーチン"のプログラムを用意しました。

**Sample19**
```cs
public class Sample19 : MonoBehaviour
{
    IEnumerator MyCoroutine()
    {
        Debug.Log("3秒待機処理開始");
        yield return new WaitForSeconds(3f);
        Debug.Log("3秒待機処理終了");
        yield return FuncA();
    }

    private void Start()
    {
        StartCoroutine(MyCoroutine());
        FuncB();
    }

    private int FuncA()
    {
        Debug.Log("FuncA()メソッドを実行");
        return 1;
    }

    private void FuncB()
    {
        Debug.Log("FuncB()メソッドを実行");
    }
}

// 出力結果
// [00:00:00]の右から2桁が秒数
[23:51:32] 3秒待機処理開始
[23:51:32] FuncB()メソッドを実行
[23:51:35] 3秒待機処理終了
[23:51:35] FuncA()メソッドを実行
```

StartCoroutine()メソッドが何をしているのか知るために、まずはStartCoroutine()メソッドを使用しないで、そのままMyCoroutine()メソッドを呼び出してみます。

**Sample20**
```cs
public class Sample20 : MonoBehaviour
{
    IEnumerator MyCoroutine()
    {
        Debug.Log("3秒待機処理開始");
        yield return new WaitForSeconds(3f);
        Debug.Log("3秒待機処理終了");
        yield return FuncA();
    }
    
    private void Start()
    {
        MyCoroutine();
        FuncB();
    }

    // FuncA()、FuncB()メソッドは同じ内容のため省略。
}

// 出力結果
FuncB()メソッドを実行
```

出力結果を見ると、MyCoroutine()メソッドが実行されていないことが分かります。つまり、StartCoroutine()メソッドを使用することでIEnumerator型を戻り値とするメソッドを実行することができます。

また下記のようにwhile文で反復子オブジェクトのMoveNext()メソッドを呼び出す方法でもMyCoroutine()メソッドを実行することができます。

**Sample21**
```cs
public class Sample21 : MonoBehaviour
{
    IEnumerator MyCoroutine()
    {
        Debug.Log("3秒待機処理開始");
        yield return new WaitForSeconds(3f);
        Debug.Log("3秒待機処理終了");
        yield return FuncA();
    }

    private void Start()
    {
        var enumerator = MyCoroutine();
        while (enumerator.MoveNext()) { }
        FuncB();
    }
    // FuncA()、FuncB()メソッドは同じ内容のため省略。
}

// 出力結果
// [00:00:00]の右から2桁が秒数
[21:23:03] 3秒待機処理開始
[21:23:03] 3秒待機処理終了
[21:23:03] FuncA()メソッドを実行
[21:23:03] FuncB()メソッドを実行
```

また上記の結果では、3秒待機処理（WaitForSecondsクラス）が実行されていません。一方でStartCoroutine()メソッドを使用した際は"yield return new WaitForSeconds(3f)"で実行することが出来ました。

ここで、「foreach文のように、IEnumerableオブジェクトのGetEnumerator()メソッドが呼ばれて、反復子のMoveNext()、Currentプロパティが自動で呼ばれているのでは？」と仮説が立ちます。さっそく仮説が正しいか確認してみます。

下記のプログラムはIEnumerator、IEnumerableインターフェースを実装したクラスを用意して、そのインスタンスをyield return文に設定します。そして各インスタンスのメソッドが呼ばれるかどうかを確認しています。

```cs
public class InnerSample22WithEnumerator
{
    private int index;
    // デリゲート変数にメソッドを格納。
    private readonly Func<int>[] actions =
    {
        InnerFuncA,
    };
    
    private static int InnerFuncA()
    {
        Debug.Log("InnerSample22WithEnumerator.InnerFuncA()メソッドを実行");
        return 2;
    }

    public InnerSample22WithEnumerator()
    {
        index = -1;
    }
    
    public bool MoveNext()
    {
        index++;
        return index < actions.Length;
    }

    public void Reset()
    {
        index = -1;
    }

    // Currentプロパティ呼び出し時にメソッドを実行して値を返す。
    public object Current => actions[index].Invoke();
}

public class InnerSample22WithEnumerable
{
    private int InnerFuncA()
    {
        Debug.Log("InnerSample22WithEnumerable.InnerFuncA()メソッドを実行");
        return 2;
    }

    public IEnumerator GetEnumerator()
    {
        yield return InnerFuncA();
    }
}

public class Sample22 : MonoBehaviour
{
    IEnumerator MyCoroutine()
    {
        // 各InnerクラスのInnerFuncA()メソッドが呼ばれるかどうか確認。
        yield return new InnerSample22WithEnumerator();
        yield return new InnerSample22WithEnumerable();
        yield return FuncA();
    }

    private void Start()
    {
        StartCoroutine(MyCoroutine());
        FuncB();
    }

    // FuncA()、FuncB()メソッドは同じ内容のため省略。
}

// 出力結果
[21:44:54] FuncB()メソッドを実行
[21:44:54] FuncA()メソッドを実行
```

仮説では各インスタンスのInnerFuncA()メソッドが呼ばれることを期待していましたが、出力結果を見ると呼ばれていません。

なので、StartCoroutine()メソッドで実行することができたWaitForSecondsクラスは他のIEnumerator、IEnumerableインターフェースを実装したクラスとは違う挙動をしていることが分かります。

もっと詳しく知るために、WaitForSecondsクラスの定義を確認していきます。ただ、WaitForSecondsクラスはライブラリクラスのため定義が分かりづらいです。そこで、WaitForSecondsクラスのような待機処理を独自に定義できるクラスを用意しました。全く同じではありませんが、雰囲気はつかめると思います。

**Sample23**
```cs
public class MyWaitForSeconds : CustomYieldInstruction
{
    private readonly float startTime;
    private readonly float waitSeconds;
    
    // インスタンスを生成した際に待機時間の計測が始まる。
    public MyWaitForSeconds(float waitSeconds)
    {
        startTime = Time.time;
        this.waitSeconds = waitSeconds;
    }

    // MonoBehaviour.Updateの後、MonoBehaviour.LateUpdateの前の各フレームで呼ばれる。
    public override bool keepWaiting => Time.time < startTime + waitSeconds;
}

public class Sample23 : MonoBehaviour
{
    IEnumerator MyCoroutine()
    {
        Debug.Log("3秒待機処理開始");
        yield return new MyWaitForSeconds(3f);
        Debug.Log("3秒待機処理終了");
        yield return FuncA();
    }

    private void Start()
    {
        // while文で反復子のMoveNext()メソッドを呼び出す方法では待機処理は行われない。
        StartCoroutine(MyCoroutine());
        FuncB();
    }

    // FuncA()、FuncB()メソッドは同じ内容のため省略。
}

// 出力結果
[22:08:21] 3秒待機処理開始
[22:08:21] FuncB()メソッドを実行
[22:08:24] 3秒待機処理終了
[22:08:24] FuncA()メソッドを実行
```

MyWaitForSecondsクラスのインスタンスを生成時に生成時の時刻と待機時間が保持されます。そして、keepWaitingプロパティがMonoBehaviour.Update後、MonoBehaviour.LateUpdate前の各フレームのタイミングで呼ばれます。

keepWaitingプロパティがtrueの場合は、まだ待機時間が完了していないためコルーチンを中断して、keepWaitingプロパティが呼ばれるタイミングで処理を再開します。falseの場合は、待機時間が完了したためコルーチンの次の処理を行います。

ちなみに、keepWaitingプロパティはMonoBehaviourのライフサイクル（Update）に従って呼ばれていますが、StartCoroutine()メソッドで呼び出す必要があります。

流れとしては、MyWaitForSecondsクラスで継承している下記のCustomYieldInstructionクラスで、keepWaitingプロパティがMoveNext()メソッドで呼ばれており、そのMoveNext()メソッドは、"なぜコルーチンの戻り値の型はIEnumerator型でなければならないのか？"の節で引用したInvokeMoveNext()メソッドから呼ばれていると思われます。
```cs
// このプログラムは、Unity-Technologies社が公開しているソースコードの一部を引用しています。
// https://github.com/Unity-Technologies/UnityCsReference/blob/master/Runtime/Export/Scripting/CustomYieldInstruction.cs
public abstract class CustomYieldInstruction : IEnumerator
{
    public abstract bool keepWaiting
    {
        get;
    }

    public object Current
    {
        get
        {
            return null;
        }
    }
    public bool MoveNext() { return keepWaiting; }
    public virtual void Reset() {}
}
```

StartCoroutine()メソッドについてまとめると、
* IEnumerator型の戻り値とするメソッドを実行することができる。
* 待機処理のような非同期処理に関係するクラスのインスタンスをyield return文で実行することができる。ただし、非同期処理に関係ないクラスのインスタンスではyield return文で実行できない。

## コルーチンでreturn文ではなく、yield return文を使う理由は？
結論から言うと**処理の途中で一時中断して、再度中断した位置から処理を再開できるようにする**ためにyield return文を使用します。

yieldステートメントについては前の章で説明しましたが同期処理で使われる場面を想定していました。そのため、今回はコルーチン（非同期処理）でreturn文とyield return文の挙動を見ていきます。

### コルーチンでreturn文を使用する
まずは、Sample19のyield return new WaitForSeconds(3f);の箇所をreturn new WaitForSeconds(3f);に変更してみましょう。

変更すると、コンパイルエラーが発生します。原因はWaitForSecondsクラスがIEnumeratorインターフェースを実装していないためIEnumerator型にキャストできないからです。

**Sample19.returnの場合**
```cs
public class Sample19 : MonoBehaviour
{
    IEnumerator MyCoroutine()
    {
        Debug.Log("3秒待機処理開始");
        // コンパイルエラー: IEnumerator型でなければならない。
        return new WaitForSeconds(3f);
        Debug.Log("3秒待機処理終了");
        yield return FuncA();
    }
}
```

そこで、Sample23で定義したMyWaitForSecondsクラスを使用します。MyWaitForSecondsクラスはIEnumeratorインターフェースを実装したCustomYieldInstructionクラスを継承しているため、IEnumerator型にキャストすることができます。これで戻り値のエラーは解決しましたが、下記のプログラムでは別のエラーが発生しています。

**Sample24**
```cs
// MyWaitForSecondsクラスは同じ内容のため省略

public class Sample24 : MonoBehaviour
{
    IEnumerator MyCoroutine()
    {
        Debug.Log("3秒待機処理開始");
        // コンパイルエラー: yield return文とreturn文は一緒に使用できない。
        return new MyWaitForSeconds(3f);
        Debug.Log("3秒待機処理終了");
        yield return FuncA();
    }

    private void Start()
    {
        StartCoroutine(MyCoroutine());
        FuncB();
    }

    // FuncA()、FuncB()メソッドは同じ内容のため省略。
}
```

実はyield return文を用いて反復子を使用する場合は、return文が使えません。そのため、下記のように"yield return FuncA();"を削除してreturn文のみにします。

**Sample25**
```cs
// MyWaitForSecondsクラスは同じ内容のため省略

public class Sample25 : MonoBehaviour
{
    IEnumerator MyCoroutine()
    {
        Debug.Log("3秒待機処理開始");
        return new MyWaitForSeconds(3f);
        Debug.Log("3秒待機処理終了");
    }

    private void Start()
    {
        StartCoroutine(MyCoroutine());
        FuncB();
    }

    // FuncA()、FuncB()メソッドは同じ内容のため省略。
}

// 出力結果
[17:26:51] 3秒待機処理開始
[17:26:51] FuncB()メソッドを実行
```

実行して出力結果を確認すると、"3秒待機処理終了"の出力されていないことに気づきます。これは当たり前ですがreturn文以降の処理が実行されないためです。

また、意外だったのは"return new MyWaitForseconds(3f);"と書いても、yield return文と同じようにkeepWaitingプロパティが呼ばれます。つまり、return文でも待機処理の計測自体はできるみたいです。

しかし、待機処理は開始後にreturnされるため待機処理完了後に行いたい処理がある場合は、待機処理完了を知る必要があります。完了タイミングを知る方法として下記のように、MyWaitForSecondsクラスのインスタンスを取得してkeepWaitingプロパティを呼ぶ方法もあります。

**Sample26**
```cs
// MyWaitForSecondsクラスは同じ内容のため省略

public class Sample26 : MonoBehaviour
{
    private MyWaitForSeconds wait = null;
    
    IEnumerator MyCoroutine()
    {
        Debug.Log("3秒待機処理開始");
        wait = new MyWaitForSeconds(3f);
        return wait;
    }

    private void Update()
    {
        if (wait != null)
        {
            // keepWaitingがfalseの場合は待機処理完了
            if (!wait.keepWaiting)
            {
                Debug.Log("3秒待機処理終了");
            }
        }
    }

    private void Start()
    {
        StartCoroutine(MyCoroutine());
        FuncB();
    }

    // FuncA()、FuncB()メソッドは同じ内容のため省略。
}

// 出力結果
[18:10:39] 3秒待機処理開始
[18:10:39] FuncB()メソッドを実行
[18:10:42] 3秒待機処理終了
```

上記の方法を使えばreturnでも待機処理における非同期処理を実装することはできますが処理が複雑になるため拡張性や可読性が低いです。

### コルーチンでyield return文を使用する
前回のreturn文を使用した書き方では、コルーチン内に処理時間がかかる処理を1つだけしか記述することが出来ない点や処理が完了したことを知るために毎フレーム確認する必要がある点から複雑になっていました。

理想としては、コルーチン内で複数の処理時間がかかる処理をまとめて書くことができて、前の処理時間が完了したら次の処理を実行するようにしたいです。そこで登場するのがyield return文です。

yield return文を使用することで、時間のかかる処理の途中で一時中断して、再度中断した位置から処理を再開できます。そのため、コルーチン内の処理を順番に実行することができます。

# まとめ
かなり長い記事になってしまいましたが、ここまでお付き合いいただきありがとうございました。

本記事では、コルーチンで使用されている戻り値の型（IEnumerator）やyieldステートメント、StartCoroutine()メソッドの3つについて取り上げました。

正直、これを知っているからと言って非同期処理がスラスラかけるようにはなりませんが、コルーチンについて少しでも理解するきっかけになれば嬉しいです。

それでは。
