# Effective C++ メモ

## 第 2 章 コンストラクタ、デストラクタ、コピー代入演算子

### 5 項 C++ が自動で書き、自動で呼び出す関数を知ろう

プログラマが中身を書かなくても、クラスの中身は空ではありません。
コピーコンストラクタ、コピー代入演算子、デストラクタはコンパイラが自動で生成します。
また、コンストラクタを 1 つも書かなければ、コンパイラはデフォルトコンストラクタを作成します。
**これらの自動で作成される関数は、すべて public で inline になります。**
つまり

```C++
class Empty{}
```

と書いても、実際は

```C++
class Empty {
public:
    Empty() {...}                               // デフォルトコンストラクタ
    Empty(const Empty& rhs) {...}               // コピーコンストラクタ
    ~Empty() {...}                              // デストラクタ
                                                // 仮想かどうかは後述

    Empty& operator=(const Empty& rhs) {...}    // コピー代入演算子が必要になる
};
```

というものが実際は作成されます。
しかし、これらの関数は、必要なときだけ生成されます。
次のようなコードで必要になります。

```C++
Empty e1;       // デフォルトコンストラクタと
                // デストラクタが必要

Empty e2(e1);   // コピーコンストラクタが必要になる
e2 = e1;        // コピー代入演算子が必要になる
```

基底クラスや非 staic なデータメンバのコンストラクタとデストラクタは見えないところで呼び出されます。
関数の仮想性は継承されるので、基底クラスが仮想デストラクタを持っていれば、派生クラスのデストラクタも仮想になります。
コピーコンストラクタとコピー代入演算子は、コピー元の非 static なデータメンバをコピー先のオブジェクトに単純にコピーします。
例えば

```C++
template<typename T>
class NamedObject {
public:
    NamedObject(const char *name, const T& value);
    NamedObject(const std::string& name, const T& value);
    ...
private:
    std::string nameValue;
    T objectValue;
};
```

この NamedObject では、コンストラクタが明示的に宣言されているので、**コンパイラはデフォルトコンストラクタの生成は行いません**。
コピーコンストラクタもコピー代入演算子も宣言されていないので、必要な場合はコンパイラがそれらを生成します。

```C++
NamedObject<int> no1("Smallest Prime Number", 2);
NamedObject<int> no2(no1);                          // コピーコンストラクタの呼び出し
```

nameValue と objectValue は、 no1 から no2 にコピーされます。
コピー代入演算子もコピーコンストラクタとほぼ同様に動作しますj。ただし、コードがエラーにならず、合理的に動作する場合だけです。そうでない場合は、コンパイラはコピー代入演算子を生成しません。
例えば

```C++
template<class T>
class NamedObject {
public:
    // nameValue は const でない string への参照
    // そのため、このコンストラクタの仮引数 name は const を指定しない
    // また、 nameValue が string への参照なので char * を引数に取るコンストラクタはない
    NamedObject(std::string& name, const T& value);
    ...                         // 前と同様 = 演算子は宣言されていないとする
private:
    std::string& nameValue;     // 今度は参照
    const T objevtValue;        // 今度は const
};
```

この場合

```C++
std::string newDog("Persephone");
std::string oldDog("Satch");
NamedObject<int> p(newDog, 2);
NamedObject<int> s(oldDog, 36);
p = s;                          // p のデータメンバはどうなるでしょう
```

nameValue は、それぞれ別の string への参照です。「参照変数は、参照するものを変更できない」という C++ の規則があるので、コンパイラはコードの生成を拒否します。必要であれば自分で作成するしかありません。
const データメンバの場合も同様で、 const なデータメンバは変更できないためです。
また、**コピー代入演算子を private にしているクラスの派生クラスに対して、コンパイラはコピー代入演算子を作成しません**。コンパイラが生成するコピー代入演算子は、オブジェクトの基底クラス部分も扱うことになっていますが、基底クラスの private 関数を、派生クラスから呼び出せないからです。

***まとめ***

* **コンパイラはクラスのデフォルトコンストラクタ、コピーコンストラクタ、コピー代入演算子、デストラクタを暗黙に(非明示的に)生成するかもしれない。**

### 6 項 コンパイラが自動生成することを望まない関数は、使用を禁止しよう

次のようなオブジェクトをコピーするコードをコンパイルできなくしたいというケースがあります。

```C++
HomeForSale h1;
HomeForSale h2;
HomeForSale h3(h1); // h1 をコピーコンストラクタでコピーしようとしている
                    // これをコンパイルできないようにしたい

h1 = h1;            // h2 を代入によってコピーしようとしている
                    // これもコンパイルできないようにしたい
```

コピーコンストラクタやコピー代入演算子は、作成していない場合はコンパイラが自動生成してしまいます。
コピーをできなくするには、コピーコンストラクタやコピー代入演算子を private にします。
ただ、他のメンバ関数やフレンド関数は private な関数を呼び出し可能なので、これだけでは不十分です。
なので、「メンバ関数を private に宣言し、意図的にその定義を書かない」という手法を使います。
実際には下記のようになります。

```C++
class HomeForSale {
public:
    ...
private:
    ...
    HomeForSale(const HomeForSale&);            // 宣言のみ
    HomeForSale& operator=(const HomeForSale&);
}
```

これで、うっかり他のメンバ関数やフレンド関数がコピーをしようとしてもリンクエラーで知ることができます。
さらに、リンクエラーをコンパイラエラーにするには、コピーコンストラクタとコピー代入演算子を private にしたクラスを作り、それを HomeForSale の基底クラスにします。

```C++
class Uncopyable {
protected:                                      // 派生クラスのオブジェクトの
    Uncopyable() {}                             // 派生と破棄は許可する
    ~Uncopyable() {}
private:
    Uncopyable(const Uncopyable&);              // しかし、コピー(代入を含む)は禁止する
    Uncopyable& operator=(const Uncopyable&);
}
```

上記のようなクラスを定義し、HomeForSale は下記のようにします。

```C++
class HomeForSale : private Uncopyable {    // このクラスは
    ...                                     // コピーコンストラクタや
};                                          // コピー代入演算子を宣言できない
```

これで、コンパイル時にエラーにすることができます。
Uncopyable の継承は public ではなくてもよく、デストラクタも仮想でなくても良いのです。
Uncopyable はデータを持っていないので、「空の基底クラスの最適化」を考えるとこのほうが望ましいのです。
ただし、多重継承を行うと「空のクラスの最適化」が適用されないかもしれませんが、上記のように使用する分には問題ありません。

***まとめ***

* **コンパイラが自動でコードを生成することを禁止するために、「対応するメンバ関数を private に宣言し、定義を書かない」という方法がある。また、Uncopyable のうようなクラスを使う方法もある。**

### 7 項 ポリモーフィズムのための基底クラスには仮想デストラクタを宣言しよう

下記のような基底クラスと派生クラスがあったとします。

```C++
class TimeKeeper {
public:
    TimeKeeper();
    ~TimeKeeper();
    ...
};
class AtomicClock : public TimeKeeper {...};
class WaterClock : public TimeKeeper {...};
class WristWatch : public TimeKeeper {...};
```

このような場合、ファクトリ関数が役に立ちそうです。
ファクトリ関数は、派生クラスのオブジェクトを生成し、それを指し示す基底クラスのポインタを返す関数です。

```C++
TimeKeeper* getTimeKeeper();    // TimeKeeper の適当な派生クラスのオブジェクトを
                                // 動的に生成し、そのオブジェクトを指し示す
                                // ポインタを返す
```

オブジェクトがヒープに確保される場合、下記のようにオブジェクトを破棄します。

```C++
TimeKeeper *ptk = getTimeKeeper();  // TimeKeeper の派生クラスの
                                    // オブジェクトを動的に生成する
...                                 // それを使う
delete ptk;                         // オブジェクトを破棄し、メモリを開放する
```

上記のコードは、動作が未定義になってしまいます。基底クラスが、仮想でないデストラクタを持つのが原因です。
派生クラスのオブジェクトを破棄するときに、「仮想デストラクタを持たない基底クラス」のポインタに delete を行うと、結果が未定義になるためです。
オブジェクトの派生クラス部分が動的に開放されない」ということが起こる可能性があります。
この問題を解決するのは簡単で、基底クラスのデストラクタを仮想関数にします。

```C++
class TimeKeeper {
public:
    TimeKeeper();
    virtual ~TimeKeeper();
    ...
};

TimeKeeper *ptk = getTimeKeeper();
...
delete ptk;                         // 今度は正しく動作する
```

一般に基底クラスは仮想関数を持っている場合が多いので、そういったクラスのデストラクタは仮想にするべきです。
逆に、仮想関数を持たないクラスの場合、そのクラスは基底クラスとして使用される前提ではないかもしれません。
そういったクラスのデストラクタを仮想にするのはよくありません。
次のようなクラスがあった場合

```C++
class Point {                       // 2D(平面)の点
public:
    Point(int xCoord, int yCoord);
    ~Point();
private:
    int x, y;
};
```

int が 32 ビットの環境では、Point オブジェクトは、64 ビットのレジスタに収まります。
しかし、デストラクタを仮想にした場合、vtbl を保持する必要があるため、32 ビット環境なら 96 ビットに、64 ビット環境なら 128 ビットになってしまい 64 ビットレジスタには入らなくなってしまいます。
また、Point オブジェクトは、C のような vptr を持たない言語では異なるものになってしまいます。
結論というと、「理由もなくすべてのデストラクタを仮想にするのは、すべてのデストラクタを非仮想にするのと同様に誤り」ということになります。
しかし、仮想関数を持たないクラスでも、「仮想でないデストラクタ」で問題が起こる場合があります。
例えば、標準の string は仮想関数を持っていませんが、string を基底クラスにしたクラスを作ってしまうかもしれません。

```C++
class SpecialString : public std::string {  // 悪い考え！ std::string は
    ...                                     // 仮想でないデストラクタを持つ
};
```

この場合、当然 SpecialString へのポインタを、string のポインタに変換して delete すると未定義の動作になってしまいます。

```C++
SpecialString *pss = new SpecialString("Impending Doom");
std::string *ps;
...
ps = pss;       // SpecialString* => std::string*
...
delete ps;      // 未定義！よくあるケースでは、ps が指していた
                // SpecialString のデストラクタが呼ばれず、
                // そのリソースが開放されないことになる
```

デストラクタを純粋仮想関数にすると良い場合があります。
純粋仮想関数を持たないクラスでも、設計上抽象クラスにしたい場合があります。
そういた場合、抽象クラスにしたいクラスに、純粋仮想デストラクタを宣言します。

```C++
class AWOV {                // デストラクタ以外に仮想関数を持たない抽象クラス
public:
    virtual ~AWOV() = 0;    // 純粋仮想デストラクタ
}
```

このクラスは、純粋仮想関数を持っているので抽象クラスです。デストラクタが仮想なので、ここまでのような問題もありません。
ただし、一点注意が必要で、純粋仮想デストラクタの定義を書く必要があります。

```C++
AWOV::~AWOV() {};           // 純粋仮想デストラクタの定義
```

まとめると、「仮想デストラクタを宣言すべきなのは、ポリモーフィズムのための基底クラス」ということになります。
しかし、すべての基底クラスがポリモーフィズムのために使われるわけではありません。 6 項の Uncopyable や標準ライブラリの input_iterator_tag などです。なので、これらのクラスに仮想デストラクタは必要ないのです。

***まとめ***

* **ポリモーフィズムのための基底クラスには仮想デストラクタを宣言しよう。特に、クラスが仮想関数を持つなら、仮想デストラクタも持たせよう。**
* **基底クラスとしてデザインされていないクラス、あるいは、基底クラスとしてデザインされていても、それがポリモーフィズム的な使われ方をされないクラスには、仮想デストラクタを宣言すべきではない。**

### 8 項 デストラクタから例外を投げないようにしよう

次のようなコードを考えてみます。

```C++
class Widget {
public:
    ...
    ~Widget() {...}             // 例外を投げるかもしれないとする
};
void doSomething()
{
    std::vector<Widget> v;
    ...
}                               // ここで自動的に v が破棄される
```

この場合、v にオブジェクトが複数あり、途中で例外が発生した場合の動作は未定義になります。
このように、デストラクタが例外を投げる場合、プログラムの中断や未定義動作になることがあります。

しかし、デストラクタで例外が発生するかもしれない処理をする場合はどうすればよいでしょうか？

```C++
class DBConnection {
public:
    ...
    static DBConnection create();   // DBConnection オブジェクトを戻す関数
                                    // 引数は説明を簡単にするため省略した

    void close();                   // 接続を切る関数
                                    // 失敗時は例外を投げることにする
};
```

これを使用するクラスを下記のようにします。

```C++
class DBConn {          // DBConnection オブジェクトを
public:                 // 管理するクラス
    ...
    ~DBConn()           // デストラクタで接続を切り、
    {                   // 切り忘れを防ぐ
        db.close();
    }
private:
    DBConnection db;
};
```

これを使うと次のようなクラスが作成できます。

```C++
{                                       // ブロックのはじめ
    DBConn dbc(DBConnection::create()); // DBConnection オブジェクトを生成し
                                        // それを管理するため、
                                        // DBConn オブジェクトに渡す

    ...                                 // DBConn を通して DBConnection
                                        // オブジェクトを使う

}                                       // ブロックの終わり
                                        // ここで DBConn オブジェクトが破棄され、
                                        // 自動的に、DBConnection の持つ接続が
                                        // DBConn のデストラクタで切られる
```

上記のコードの場合、データベース接続の切断で例外が発生した場合にデストラクタ内から例外を投げてしまいます。

この問題を避けるには 2 つの方法があります。

* **デストラクタ中で、プログラムを中止してしまう**
  例外が投げられたら、abort などでプログラムを強制終了してしまいます。

  ```C++
  DBConn::~DBConn()
  {
      try { db.close(); }
      catch (...) {
          close の失敗を記録する;
          std::abort();
      }
  }
  ```

  オブジェクトの破棄に失敗したらプログラム継続の意味がない場合は上記のような処理で OK でしょう。

* **デストラクタが飲み込んでしまう**
  例外が投げられたら、それをデストラクタが飲み込むという方法もあります。

  ```C++
  DBConn::~DBConn()
  {
      try { db.close(); }
      catch (...) {
          close の失敗を記録する;
      }
  }
  ```

  エラーが起こっても、プログラムを継続したい場合はこのようにします。

上記の 2 つの方法は例外発生時に具体的な対応を行っていないためあまり良い方法とは言えません。
よりよい方法は、DBConn のクライアント(利用者)に「問題に対処する機会」を与えるような設計です。

```C++
class DBConn {
public:
    ...

    void close()                        // このクラスのクライアントのための関数
    {
        db.close();
        closed = true;
    }

    ~DBConn()
    {
        if (!closed) {
            ...
        }
        try {                           // クライアントが接続を切っていなければ、
            db.close();                 // ここで切る
        }
        catch(...) {                    // 切るのに失敗したら記録して、
            close の失敗を記録する;     // 中止するか、飲み込む
            ...
        }
    }
private:
    DBConnection db;
    bool closed;
}
```

これでクライアントは、自分で例外を処理する機会を得ることができます。

***まとめ***

* **デストラクタは例外を投げてはいけない。デストラクタ内で例外を投げる関数を呼び出す場合、デストラクタがプログラムを中止させるか、その例外を補足し飲み込む(処理する)ようにする。**
* **クラスのクライアントが例外に対処する必要があるなら、クラスに「例外を投げるかもしれない処理」をする通常の関数(つまり、デストラクタ出ない関数)を付ける。

### 9 項 コンストラクタやデストラクタ内では決して仮想関数を呼び出さないようにしよう

コンストラクタやデストラクタから仮想関数を呼び出してはいけません。
例えば

```C++
class Transaction {                             // すべての取引の基底クラス
public:
    Transaction();
    virtual void logTransaction const = 0;      // 型ごとにログを取る関数
    ...
};

Transaction::Transaction()                      // 基底クラスのコンストラクタの定義
{
    ...
    logTransaction();                           // コンストラクタの最後で
}                                               // この取引のログを取る

class BuyTransaction : public Transaction {     // 派生クラス「買い」
public:
    virtual void logTransaction() const;        // 「買い」のログ
    ...
};

class SellTransaction : public Transaction {    // 派生クラス「売り」
public:
    virtual void logTransaction() const;        // 「売り」のログ
    ...
};
```

この時、次のようなコードが合った場合

```C++
BuyTransaction b;
```

この時、BuyTransaction の前に Transaction のコンストラクタが呼ばれます。
Transaction のコンストラクタの最後で、仮想関数 logTransaction を呼び出しています。
この時呼び出されるのは、BuyTransaction のものではなく、**基底クラス**である Transaction クラスの BuyTransaction です。
デストラクタでも、まず派生クラスの部分が破棄され、派生クラスのメンバ変数は不定になります。その後、基底クラスの破棄を行いますが、その際に呼び出される仮想関数は**基底クラスのも**のになります。
コンパイラによっては警告を出す場合があります。

また

```C++
class Transaction {
public:
    Transaction()
    { init(); }                                 // 仮想でない関数の呼び出しですが、
    virtual void logTransaction() const = 0;
    ...
private:
    void init()
    {
        ...
        logTransaction();                       // 中で仮想関数を呼び出している！
    }
};
```

上記の場合は、直接仮想関数を呼び出していないため、コンパイラはエラーを出しません。

この問題の解決方法はいくつかありますが、その 1 つは「logTransaction を仮想ではない関数にする」というものです。

```C++
class Transaction {
public:
    explicit Transaction(const std::string& logInfo);
    void logTransaction(const std::string& logInfo) const;  // 非仮想関数
    ...
};
Transaction::Transaction(const std::string& logInfo)
{
    ...
    logTransaction(logInfo);                                // 非仮想関数の呼び出し
}
class BuyTransaction : public Transaction {
public:
    BuyTransaction(params)
     : Transaction(createLogString(params))                 // ログ情報を基底クラスの
     { ... }                                                // コンストラクタに渡す
     ...
private:
    static std::string createLogString(params);
};
```

基底クラスのコンストラクタ内で仮想関数を呼び出しても、派生クラスの関数が呼び出されないため、派生クラスのコンストラクタが基底クラスのコンストラクタに必要な情報を渡すようにしたのです。
この例ではBuyTransactionで、private な static 関数 createLogString を使用しています。関数を static にすることで「まだ初期化されていないデータメンバを使う」という危険を避けることができます。

***覚えておくこと***

* **オブジェクトの生成や破棄の間に仮想関数を呼び出してはいけない。そのような呼び出しで実行されるのは、そのときの(生成屋は期の途中の)オブジェクトの型のものになる。**

### 10 項 代入演算子は*thisへの参照を戻すようにしよう

コピー演算子は下記のように繋げて使えます。

```C++
int x, y, z;
x = y = z = 15; // 代入を繋げる
```

また、コピー演算子は右結合です。なので上記のような代入は下記のように解釈されます。

```C++
x = (y =(z = 15));
```

これは、コピー代入演算子が「左辺への参照」を返すことで実現されています。
自作のクラスにコピー代入演算子を定義する場合にもこの原則を守るべきです。

```C++
class Widger {
public:
    ...
    Widget& operator=(const Widget& rhs)    // 戻り値型はこのクラスへの参照
    {
        ...
        return *this;                       // 左辺(自分自身)への参照を返す
    }
    ...
};
```

この原則は単純代入に限らずすべての代入演算子に適用されます。

```C++
class Widget {
public:
    ...
    Widget& operator+=(const Widget& rhs)   // C=、-=、*= などでも同じように
    {
        ...
        return *this;
    }
    Widget& operator=(int rhs)              // 仮引数が型が違っても原則に従う
    {
        ...
        return *this;
    }
    ...
};
```

この原則に従わなくてもコンパイルエラーにはなりませんが、すべての組み込み型と標準ライブラリで使用されています。
なので、特別な理由がない限り従いましょう。

***覚えておくこと***

* **代入演算子は *this への参照を返すようにしよう。**

### 11 項 operator= の実装では、自己代入に備えよう

自己代入は次のようなものです。

```C++
class WIdget{...};
Widget w;
...
w = w;              // 自己代入
```

次のような場合もあります。

```C++
a[i] = a[j];    // 自己代入になるかもしれない
```

```C++
*px = *py;      // 自己代入になるかもしれな
```

```C++
class Base {...};
class Derived : public Base {...};
void doSomething(const Base& rb,    // rb と *pd は同じオブジェクト
                    Derived* pd);   // になることもある
```

また、自分でリソースを管理したい場合「自己代入で、使う前のデータを破棄してしまう危険」に注意しなければなりません。

```C++
class Bitmap {...};

class Widget {
    ...
private:
    Bitmap *pb;     // ヒープ上においたオブジェクトへのポインタ
}

Widget::operator=(const Widget& rhs)    // operator= の危険な実装
{
    delete pb;                          // 現在のビットマップを破棄
    pb = new Bitmap(*rhs.pb);           // 右辺のビットマップをコピー

    return *this;                       // 10 項を参照
}
```

このような処理では、*this (代入式の左辺にある代入先)と rhs (代入式の右辺にある代入元)が同じオブジェクトの場合、つまり、自己代入の場合に問題が起こります。

この問題を避ける伝統的な方法は、operator= の処理のはじめで、自己代入かどうかをチェックすることです。

```C++
Widget& Widget::operator=(const Widget& rhs)
{
    if (this == &rhs) return *this; // 同一性テスト
                                    // 自己代入では何もしない
    delete pb;
    pb = new Bitmap(*rhs.pb);
    return *this;
}
```

上記のコードにはまだ問題があります。new Bitmap で例外が発生した場合、Widget オブジェクトは「破棄された Bitmap オブジェクトへのポインタ」を保持することになります。
これを解決するには、pb に delete を行う前に、pb が指し示すべきオブジェクトをコピーで生成しておきます。

```C++
Widget& Widget::operator=(const Widget& rhs)
{
    Bitmap *pOrig = pb;         // 元の pb の値を記録
    pb = new Bitmap(*rhs.pb);   // pb が *rhs.pb のコピーを指し示すようにする
    delete pOrig;               // 元の pb が指し示すオブジェクトを破棄
    return *this;
}
```

上記のコードは自己代入に関してはあまり効率がよくありませんが正常に動作します。効率を考える場合は、operator= の最初で同一性テストを行っても良いでしょう。しかし、自己代入が発生する確率がどの程度あるかを考慮して自己テストを行うかを決めましょう。

また、例外にも自己代入にも安全なコードにするために、「コピーと交換」というテクニックがあります。

```C++
class Widget {
    ...
    void swap(Widget& rhs);                     // *this と rhs のデータを交換する関数
    ...
};

Widget& Widget::operator=(const Widget& rhs)
{
    Widget temp(rhs);                           // rhs のデータのコピーを作る

    swap(temp);                                 // *this とコピーデータを交換する

    return *this;
}
```

さらに

* コピー代入演算子の仮引数は参照にしなくても良い
* 仮引数を参照にしなければ、引数はコピーされる

という事実を使って下記のように書き換えることもできます。

```C++
Widget& Widget::operator=(Widget rhs)   // rhs は渡される引数のコピーになる
{                                       // つまり、仮引数が参照でないことに注意

    swap(rhs);                          // *this のデータと rhs のデータを交換する

    return *this;
}
```

***覚えておくこと***

* **operator= の定義は、自己代入に対して安全に振舞うように書こう。そのテクニックには、コピー元とコピー先のオブジェクトのアドレスを比較する方法、コード(ステートメント)の順序を注意深く決める方法、「コピーと交換」の方法がある。**
* **複数のオブジェクトを操作する場合、異なるオブジェクトと考えられていたものが、実は同じオブジェクトであることもある。そのような場合でも、正しく動作するように関数を書こう。**

### 12 項 コピーするときは、オブジェクトの全体をコピーしよう

```C++
void logCall(const std::strinng& funcName);             // ログを取る

class Customer {
public:
    ...
    Customer(const Customer& rhs);
    ...
private:
    std::string name;
};

Customer::Customer(const Customer& rhs)
: name(rhs.name)                                        // rhs のデータをコピー
{
    logCall("Customer copy constructor");
}

Customer& Customer::operator=(const Customer& rhs)
{
    logCall("Customer copy assignment operator");
    name = rhs.name;                                    // rhs のデータをコピー
    return *this;
}
```

上記のコードは問題ありません。しかし、次のように Customer に新しいデータメンバを追加すると問題が発生します。

```C++
class Date {...};           // 日時を表すクラスとする

class Customer {
public:
    ...                     // 前と同じ
private:
    std::string name;
    Date lastTransaction;   // 追加された
};
```

この段階で、前に書いたコピー関数は、部分的なコピーしかしない関数になります。しかし、コンパイラーは何も言いません。
なので、「クラスにデータメンバを追加したなら、コピー関数も更新しなければならない」のです。

継承を考えると更に複雑になります。

```C++
class PriorityCustomer : public Customer {                                      // 派生クラス
public:
    PriorityCustomer(const PriorityCustomer& rhs);
    PriorityCustomer& operator=(const PriorityCUstomer& rhs);
    ...
private:
    int priority;
};

PriorityCustomer::PriorityCustomer(const PriorityCustomer& rhs)
    : priority(rhs.priority)
{
    logCall("PriorityCustomer copy constructor");
}

PriorityCustomer& PriorityCustomer::operator=(const PriorityCustomer& rhs)
{
    logCall("PriorityCustomer copy assignment operator");
    priority = rhs.priority;
    return *this;
}
```

上記のコードでは、PriorityCustomer のデータメンバしかコピーしていません。Customer のデータメンバはコピーされていません。
上記のコードは、Customer のコンストラクタを明示的に呼び出していないので、引数なしのコンストラクタが呼ばれています。
これによって、name と lastTransaction はデフォルト値に初期化されることになります。
コピー代入演算子でも、同様のことが起こります。
つまり、派生クラスにコピー関数を書く場合は、基底クラスの部分もコピーする必要があるのです。
通常、基底クラスのメンバは、private なデータメンバなので、派生クラスから直接アクセスできません。派生クラスのコピー関数では、基底クラスの対応するコピー関数を呼び出すようにします。

```C++
PriorityCustomer::PriorityCustomer(const PriorityCustomer& rhs)
    : Customer(rhs),                                            // 基底クラスのコピーコンストラクタを呼び出す
    priority(rhs.priority)
{
    logCall("PriorityCustomer copy constructor");
}

PriorityCustomer&
PriorityCustomer::operator=(const PriorityCustomer& rhs)
{
    logCall("PriorityCustomer copy assignment operator");
    Customer::operator=(rhs);                                   // 基底クラスのコピー代入演算子を呼び出す
    priority = rhs.priority;
    return *this;
}
```

つまり、(1) そのクラスで宣言しているすべてのデータメンバをコピーし、(2) 基底クラスの適当なコピー関数を呼び出すということです。

2 つの関数には共通する部分がありますが、コンストラクタからコピー代入演算子を呼び出したり、その逆を行ってはいけません。かわりに、共通する部分を別の関数に抜き出して、その関数を呼び出すようにしましょう。

***覚えておくこと***

* **コピー関数は、オブジェクトのデータメンバと基底クラスのすべてをコピーするように書かなければならない。**
* **一方のコピー関数から他方のコピー関数を呼び出すようなコードを書いてはならない。かわりに、両者の共通部分を別の関数として定義し、それを呼び出すようにする。

***

[戻る](./index.md)
