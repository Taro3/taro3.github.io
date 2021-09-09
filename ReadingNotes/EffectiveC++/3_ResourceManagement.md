# Effective C++ メモ

## 第 3 章 リソース管理

### 13 項 リソース管理にオブジェクトを使おう

下記のようなファクトリ関数があるとします。

```C++
Investment* createInvestment(); // Investment の派生クラスのオブジェクトを
                                // 生成し、そのオブジェクトへのポインタを返す
                                // そのオブジェクトは呼び出し元で破棄する
                                // (引数は簡単にするために省略)
```

これを下記のように使うとします。

```C++
void f()
{
    Investment *pInv = createInvestment();  // ファクトリ関数の呼び出し

    ...                                     // pInv を使う

    delete pInv;                            // オブジェクトを破棄
}
```

このコードは安全そうですが、必ず pInv が delete されるとは限りません。... の部分の処理中に return してしまったり、例外が発生する可能性があるためです。

このような場合、リソースをオブジェクトの中に置き、デストラクタで破棄します。
標準ライブラリの unique_ptr は、そのようなメモリ操作を行うためのスマートポインタです。
unique_ptr を使用すると下記のようになります。

```C++
void f()
{
    std::unique_ptr<Investment> pInv(createInvestment());   // ファクトリ関数の呼び出し

    ...                                                     // 前と同様に pInv を使う

}                                                           // 自動的に pInv を unique_ptr の
                                                            // デストラクタで破棄
```

これは、リソース管理を行うオブジェクトの、2 つの重要な事実を示しています。

* **リソースを確保したらすぐにリソース管理オブジェクトに渡す**
  これをRAIIといいます。
* **リソース管理オブジェクトは、リソースを確実に開放するため、デストラクタを使う**

1 つのリソースを指し示すために、複数の unique_ptr は使用できません。

```C++
std::unique_ptr<Investment>                 // pInv1 は createInvestment が生成した
    pInv1(createInvestment());              // オブジェクトを指し示すようになる

std::unique_ptr<Investment> pInv2(pInv1);   // pInv2 がそのオブジェクトを指し示すように
                                            // なり、pInv1 はヌルになる

pInv1 = pInv2;                              // 今度は、pInv1 がそのオブジェクトを指し示し、
                                            // pInv2 がヌルになる
```

1 つのリソースを複数から指し示したいときは、shared_ptr を使用します。

```C++
void f()
{
    ...
    std::shared_ptr<Investment>
        pInv(createInvestment());   // ファクトリ関数の呼び出し

    ...                             // 前と同じように pInv を使う

}                                   // pInv の指すオブジェクトは
                                    // shared_ptr デストラクタで
                                    // 自動的に破棄される
```

上記のコードは、unique_ptr と殆ど同じに見えますが、次のようなコードを書くことができます。

```C++
viod f()
{
    ...
    std::shared_ptr<Investment>     // pInv1 は createInvestment の
        pInv1(createInvestment());  // 生成したオブジェクトを指す

    std::shared_ptr<Investment>     // これで pInv1 と pInv2 が同じ
        pInv2(pInv1);               // オブジェクトを指すようになる

    pInv1 = pInv2;                  // これは何も変わらない
    ...
}                                   // pInv1 と pInv2 は破棄され、
                                    // これらが指していたオブジェクトも
                                    // 自動的に破棄される
```

ここでのアドバイスは、「リソースを開放するコードを直接書かなければならないなら(つまり、リソース管理オブジェクト以外の場所に delete 文を書かなければならないなら)、何かが間違っている」ということです。
ここでの、createInvestment のように、生のポインタを返す関数は、クライアント側でリソース漏れを起こすコードにつながりやすいと指摘しておきます。

#### 覚えておくこと

* **リソース漏れを避けるために、RAII オブジェクトを使おう。RAII オブジェクトはコンストラクタでリソースを受け取り、デストラクタでそれを破棄する。**
* **unique_ptr と shared_ptr は、一般的に有用な RAII クラス。ただし、「コピーが自然なもの」が必要なら、unique_ptr より shared_ptr がよい。unique_ptr をコピーすると、コピー元はヌルになる。**

### 14 項 リソース管理クラスのコピーの振る舞いはよく考えて決めよう

下記のように使用する Mutex クラスを考えてみます。

```C++
void lock(Mutex *pm);   // pm の指すミューテックスをロックする
void unlock(Mutex *pm); // pm の指すミューテックスをアンロックする
```

ここで、アンロックを忘れないように、ロック管理を行うクラスを RAII で作るとします。

```C++
class Lock {
public:
    explicit Lock(Mutex *pm)
    : mutexPtr(pm)
    { lock(mutexPtr); }             // リソースの確保(ミューテックスのロック)
    ~Lock() { unlock(mutexPtr); }   // リソースの開放(ミューテックスのアンロック)
private:
    Mutex *mutexPtr;
}
```

Lock の利用者は、RAII の使用方法にに従って Lock を使います。

```C++
Mutex m;            // これから使うミューテックスの定義
...
{                   // クリティカルセクションのブロックを生成
    Lock ml(&m);    // ミューテックスをロック
    ...             // クリティカルセクションの実行
}                   // ブロックの最後で自動的にミューテックスを開放
```

ここまでは問題ありませんが、ここでミューテックスがコピーされたらどうなるでしょうか？

```C++
Lock ml1(&m);   // m をロック
Lock ml2(ml1);  // ml2 を ml1 にコピー。すると、どうなるでしょう
```

ここでは狭い意味のコピーが発生します。「RAII オブジェクトがコピーされるとき、どうすべきか」という問題が発生します。
たいていは、以下の中の 1 つを選ぶことになります。

* **コピーを禁止する** 以下のように、コピー関数を private 宣言します。
  
  ```C++
  class Lock : private Uncopyable { // コピーの禁止
  public:                           // 6 項を参照
    ...                             // 前と同じ
  };
  ```

* **リソースへの参照を数える** std::shared_ptr と同じように参照回数をカウントします。以下のように、std::shared_ptrをそのまま使用することもできます。
  
  ```C++
  class Lock {
  public:
    explicit Lock(Mutex *pm)            // Mutex ポインタと unlock で
    : mutexPtr(pm, unlock)              // shared_ptr を初期化
    {                                   // unlock がデリータになる

        lock(mutexPtr.get());           // get に関しては 15 項を参照
    }
  private:
    std::shared_ptr<Mutex> mutexPtr;    // 生のポインタではなく
  };                                    // shared_ptr を使う
  ```

* **管理しているリソースをコピーする** RAII オブジェクトの中身を「深いコピー(指し示しているリソース自体をコピー)」します。
* **管理しているリソースの管理者を変更する** std::unique_ptr のように、リソースの所有権を委譲します。

コピー関数(コピーコンストラクタとコピー代入演算子)は、コンパイラが自動的に生成することもありますが、自分が期待したものではない場合は、独自に実装する必要があるわけです。

***覚えておくこと***

* **RAII オブジェクトのコピーでは、そのオブジェクトが管理するリソースのコピーが問題になる。コピーにおけるリソースの扱いを決めることが、RAII オブジェクトの振る舞いを決めることになる。**
* **一般的な RAII オブジェクトのコピーでは、コピーを禁止するか、参照を数えるようにする。しかし、他の扱いを考えることもある。**

### 15 項 リソース管理クラスには、リソースそのものへのアクセス方法を付けよう

下記のようなスマートポインタがあった場合

```C++
std::shared_ptr<Investment> pInv(createInvestment());
```

これを使う関数は次のような感じです。

```C++
int dayHeld(const Investment *pi);  // 投資されてから経った日数を返す
```

すると、次のように使いたくなります。

```C++
int days = daysHeld(pInv);  // エラー
```

このようなときのために、uniqut_ptr も shared_ptr も、内部で保持しているポインタを取り出す get というメンバ関数を持っています。

```C++
int days = daysHeld(pInv.get());  // 問題なし
                                  // pInv が保持するポインタを daysHeld に渡している
```

ほとんどのスマートポインタは(unique_ptr や shared_ptr も)、ポインタの逆参照演算子(-> 演算子と * 演算子)をオーバーロードしています。
これにより次のような使い方ができます。

```C++
class Investment {                                      // すべての「投資」を表すクラスの基底クラス
public:
  bool isTaxFree() const;
  ...
};

Investment* createInvestment();                         // ファクトリ関数

std::shared_ptr<Investment>                             // shared_ptr でリソース管理
  pi1(createInvestment());
bool taxable1 = !(pi1->isTaxFree());                     // -> 演算子を通してリソースにアクセス
...
std::unique_ptr<Investment> pi2(createInvestment());    // unique_ptr でリソース管理
bool taxable2 = !((*pi2).isTaxFree());                   // * 演算子を通してリソースにアクセス
...
```

RAII クラスの中には、内部のリソースにアクセスするための暗黙の型変換を持つものがあります。

```C++
FontHandle getFont();             // C スタイルの関数
                                  // 簡単にするため、引数は省略

void releaseFont(FontHandle fh);  // 同じく C スタイルの関数

class Font {                      // RAII クラス
public:
  explicit Font(FontHandle fh)    // リソースの確保
    : f(fh)                       // C スタイルの API を使うため値渡し
    {}
    ~Font() { releaseFont(f); }   // リソースの開放
private:
  FontHandle f;                   // 生のフォントリソース
};
```

フォントを扱うために FontHandle を使う C スタイルの関数がたくさんある場合、Font オブジェクトを頻繁に FontHandle に変換する必要が出てきます。そのために、**明示的なリソースアクセス**のための get を、Font クラスに定義することもできます。

```C++
class Font {
public:
  ...
  FontHandle get() const { return f; }  // 明示的にアクセスを与える関数
  ...
};
```

しかし、これだと、C スタイルのフォント関数を呼び出すたびに、get を使わなければなりません。

```C++
void changeFontSize(FontHandle f, int newSize); // C スタイルの関数

Font f(getFont());
int newFontFize;

...
changeFontSize(f.get(), newFontSize);           // Font から FontHandle を得るため get を使う
```

そこで、Font に、FontHandle への**暗黙の型変換**をもたせる方法もあるのです。

```C++
class Font {
public:
  ...
  operator FontHandle() const // 暗黙の型変換
  { return f; }
  ...
};
```

この演算子があると、C スタイルの「**FontHandle** を引数に取る関数」を、以下のように簡単に呼び出すことができるようになります。

```C++
Font f(getFont());
int newFontSize;
...
changeFontSize(f, newFontSize); // Font が FontHandle に自動的に変換される
```

この方法の欠点は、**暗黙の型変換はエラーを引き起こしやすい**ことです。

```C++
Font f1(getFont());
...
FontHandle f2 = f1; // おっと！Font オブジェクトをコピーするつもり
                    // だったのに、f1 を FontHandle オブジェクト
                    // に変換してからコピーしてしまった
```

RAII オブジェクトの内部リソースへのアクセスは、カプセル化を破壊するという意見もありますが、適材適所でしょう。

***覚えておくこと***

* **利用する API によっては、生のりソースにアクセスする必要がある。そのため、RAII クラスは、管理するリソースにアクセスする方法を提供すべき。**
* **そのアクセス方法には、明示的なもの(get のような関数)と非明示的なもの(暗黙の型変換)がある。一般には、明示的なものが安全だが、暗黙の型変換が使えると、クライアントにはより便利になる。**

### 16 項 対応する new と delete は同じ型のものを使おう

以下のコードは間違いです。

```C++
std::string *stringArray = new std::string[100];
...
delete stringArray;
```

配列を new で作成しているのに、delete は単一のオブジェクトに対して行われているためです。このときの動作は未定義になります。

```C++
std::string *stringPtr1 = new std::string;
std::string *stringPtr2 = new std::string[100];
...
delete stringPtr1;                              // 単独オブジェクトを破棄
delete [] stringPtr2;                           // オブジェクトの配列を破棄
```

上記のように、単独のオブジェクトと配列で **delete** と **delete []** を使い分ける必要があります。
必ず対になるように使用しなければなりません。

また、typedef を使用するときも気をつけなければなりません。

```C++
typedef std::string AddressLines[4];  // string が 4 つの配列
```

上記のような typedef をした場合

```C++
std::string *pal = new AddressLines;  // new AddressLines は string* を戻す
                                      // これは new string[4] と同じ意味になる
```

この new に対する delete は同じ形式のものでなければなりません。

```C++
delete pal;     // 未定義！
delete [] pal;  // 問題なし
```

このような混乱を避けるため、配列を typedef することは避けたほうが良いでしょう。
代わりに、string や vector などで動的配列を確保しましょう。
今の例では、AddressLines は string の vector 、つまり、 vector\<string\> になります。

***覚えておくこと***

* **オブジェクトを new で生成するときに [] を使ったなら、対応する delete でも [] を使おう。逆に、オブジェクトを new で生成するときに [] を使っていないなら、対応する delete でも [] うぃ使わないように。**

### 17 項 new で生成したオブジェクトをスマートポインタに渡すのは、独立したステートメントで行うようにしよう

整数を返す、priority という関数があり、動的に確保した Widget オブジェクトと priority の戻り値を引数に取る processWidget が下記のようにあった場合

```C++
int priority();
void processWidget(std::shared_ptr<Widget> pw, int priority);
```

ここでは、processWidget は、スマートポインタを受け取るようになっています。
ここで、processWidget は次のようには呼び出すことはできません。(コンパイルできません)

```C++
processWidget(new Widget, priority());
```

std::shared_ptr の「ポインタを受け取るコンストラクタ」は explicit で宣言されているので、new Widget で生成されるポインタが暗黙の型変換で std::shared_ptr に変換されることはないからです。
しかし、次のようなコードなら**コンパイルは**通ります。

```C++
processWidget(std::shared_ptr<Widget>(new Widget), priority());
```

しかし、上記のコードではリソース漏れの可能性があります。
コンパイラが、引数を評価する順番はコンパイラに任せられているため、new Widget のあとに priority() が呼び出され、priority() で例外が発生した場合、new Widget で生成されたポインタが**行方不明になってしまう**からです。

これを避けるのは簡単で、Widget を生成したポインタをスマートポインタに引き渡す処理を、独立したステートメントに分ければよいのです。

```C++
std::shared_ptr<Widget> pw(new Widget); // 独立したステートメントで
                                        // オブジェクトのポインタを
                                        // スマートポインタに渡す

processWidget(pw, priority());          // これでリソース漏れは起きない
```

これで、必ず new Widget で生成されたポインタが、スマートポインタに引き渡される処理と、priority() の呼び出しが分けられるため、ポインタが行方不明になることがなくなるためです。

#### 覚えておくこと

* **new で生成したオブジェクトをスマートポインタに渡すのは、独立したステートメントで行うようにしよう。そうしないと、例外が投げられと時に、リソース漏れが起こるかもしれない。**

***

**[戻る](../index.md)**
