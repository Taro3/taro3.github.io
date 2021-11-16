# Effective C++ メモ

## 第 4 章 デザインと宣言

### 18 項 インターフェースは、正しく使うときには使いやすく、間違った使い方では使いにくいものにしよう

例えば日付を表すクラスのコンストラクタを作る場合

```C++
class Date {
public:
  Date(int month, int day, int year);
  ...
};
```

これは合理的に見えますが、このクラスのユーザは、以下のような 2 つの間違いを起こすかもしれません。

* 引数の順番を間違える

  ```C++
  Date d(30, 3, 1995);  // おっと！「30, 3」ではなく「3, 30」であるべきだ
  ```

* 無効な数値を入れてしまう
  
  ```C++
  Date d(3, 40, 1995);  // おっと！「3, 40」ではなく「3, 30」であるべきだ
  ```

これらのミスは、新しい型を使うことで防げます。

```C++
struct Day {
  explicit Day(int d) : val(d) {}
  int val;
};

struct Month {
  explicit Month(int m) : val(m) {}
  int val;
};

struct Year {
  explicit Year(int y) : val(y) {}
  int val;
};

class Date {
public:
  Date(const Month& m, const Day& d, const Year& y);
  ...
};

Date d(30, 3, 1995);                    // エラー!型が違う
Date d(Day(30), Month(3), Year(1995));  // エラー!型が違う
Date d(Month(3), Day(30), Year(1995));  // OK 正しい型
```

また、値を制限したい場合、例えば Month をすべて定義してしまうという方法があります。

```C++
class Month {
public:
  static Month Jan() { return Month(1); }   // すべての有効な Month を返す
  static Month Feb() { return Month(2); }   // 関数群を考える
  ...
  static Month Dec() { return Month(12); }
  ...                                       // 他のメンバ関数
private:
  explicit Month(int m);                    // 新しい Month の値が生成される
                                            // ことを防ぐ
  ...                                       // データメンバ
};

Date d(Month::Mar(), Day(30), Year(1995));
```

ミスを防ぐ別の方法には「その型でできることを制限する」というものがあります。よくあるのは const を付けることです。例えば、* 演算子の戻り値に const を付けることで、下記のようなミスを防ぐことができます。

```C++
if (a * b = c) ...  // おっと。比較のつもりだった！
```

クライアントが何かを覚えておかなければならないようなインターフェースは、誤用される可能性が高くなります。

```C++
Investment* createInstance(); // 13 項で紹介したもの
                              // 簡単にするために、引数は省略
```

この関数を使ってリソースリークを起こさないためには、戻り地であるポインタに、delete を適用しなければなりません。
これは、delete を忘れるとか、2回 delete を適用してしまうといったミスが考えられます。
これを避けるために、下記のようにスマートポインタを使用します。

```C++
std::shared_ptr<Investment> createInvestment();
```

これは、利用者がスマートポインタを使用することを強制しますので、Investment の開放忘れを考えなくて良くなります。
std::shared_ptr は、メモリ以外のリソース開放に関してもミスを防ぐのに役立ちます。リソースの開放に使うデリータを指定可能だからです。

スマートポインタとデリータを使用することで、クロス DLL 問題を回避することができます。クロス DLL 問題とは、ある DLL で確保されたリソースが別の DLL のデリータで開放されてしまう事による問題です。
しかし、スマートポインタとデリータを使用することで、確実に同じ DLL のデリータを使用してリソースが開放されることが保証されます。

**覚えておくこと**

* **よいインターフェースは、正しく使うときには使いやすく、間違った使い方では使いにくい。自分でインターフェースを書く場合、常にそのように書くべき。**
* **インターフェースをクライアントに正しく使わせるには、新しい型を定義し、型に作用する演算子の機能や、オブジェクトの値を制限すると良い。そして、クライントにリソースの管理を任せないのか良い。**
* **std::shared_ptr では、デリータを設定できる。これにより、クロス DLL 問題を防ぎ、また、「ミューテックスを自動的にアンロックする」などということができる。**

### 19 項 クラスのデザインを型のデザインとして考えよう

下記のことを考慮してクラスを設計しよう。

* 新しい型のオブジェクトはどのように生成され、破棄されるのか？
* オブジェクトの初期化と代入は、異なる操作になるか？
* 新しい型のオブジェクトの値渡しは、具体的にどのようなものになるか？
* 新しい型のオブジェクトが持てる有効な値はどのようなものになるか？
* 新しい型は継承の階層の中にうまくあてはまるか？
* 新しい型はどのような型変換を受けるか？
* 新しい型で意味のある演算子や関数は何か？
* 標準的な関数で禁止すべきものは何か？
* 新しい型のメンバにアクセスできるものは何か？
* 新しい型の「宣言されていないインターフェース」は何か？
* 新しい型はどのくらい一般的か？
* 本当に新しい型が必要なのか？

**覚えておくおこと**

* **クラスのデザインは、型のデザイン。新しい型を定義する前に、この項に挙げたすべての問いを考えてみよう。**

### 20 項 値渡しより const 参照渡しを使おう

以下のクラスを考える。

```C++
class Person {
public:
  Person();           // 簡単にするため、引数なしとした
  virtual ~Person();  // 仮想にする理由は 7 項参照のこと
  ...
private:
  std::string name;
  std::string address;
};

class Student : public Person {
public:
  Student();          // ここでも仮引数を省略している
  virtual ~Student();
  ...
private:
  std::string schoolName;
  std::string schoolAddress;
};
```

ここで、次のようなコードを考えます。

```C++
bool validateStudent(Student s);        // Student オブジェクトを値渡しで受け取る関数
Student plato;                          // ソクラテスの学生プラトン
bool platoOK = validateStudent(plato);  // 関数の呼び出し
```

上記のコードでは、コンストラクタが 6 回、デストラクタも 6 回呼び出されています！
これを避ける方法が、const 参照渡しです。

```C++
bool validateStudent(const Student& s);
```

これは、前のコードよりかなり効率的です。
さらに、引数を参照渡しにすると、「スライス問題」も避けられます。以下のようなことです。

```C++
class Window {
public:
  ...
  std::string name() const;     // ウィンドウの名前を返す
  virtual void display() const; // ウィンドウとその内容を描画する
};

class WindowWithScrollBars : public Window {
public:
  ...
  virtual void display() const;
};
```

ここで、ウィンドウの名前を出力してからウィンドウを表示する関数を書くとします。

```C++
void printNameAndDisplay(Window w)  // 誤り！ 引数がスライスされる
{
  std::cout << w.name();
  w.display();
}
```

この値渡しの関数を下記のように呼び出します。

```C++
WindowWithScrollBars wwsb;
printNameAndDisplay(wwsb);
```

この呼出では、WindowWithScrollBars オブジェクトから、仮引数 w が Window オブジェクトとして生成されます。そして、wwsb の持つ WindowWithScrollBars に特有の部分は切り捨てられてしまうのです。
つまり、WindowWithScrollBars::display ではなく、Window::display が呼び出されてしまいます。
ここで、w を const 参照にすれば、このスライス問題は避けられます。

```C++
void printNameAndDisplay(const Window w)  // 問題なし。引数はスライスされない
{
  std::cout << w.name();
  w.display();
}
```

**覚えておくこと**

* **値渡しより const 参照渡しを使おう。そうすれば、一般的に、効率的で、スライス問題が起こらない。**
* **ただし、これは、組み込み型と STL の反復子・関数オブジェクトには適用されない。これらについては、普通、値渡しが適当。**

### 21 項 オブジェクトを戻すべき時に参照を返そうとしないこと

以下のコードを考えます。

```C++
class Rational {
public:
  Rational(int numerator = 0,                             // なで explicit にしないかは 24 項を参照のこと
    int denominator = 1);                                 // numerator は分子、denominator は分母

  ...
private:
  int n, d;
  friend const Rational operator*(const Rational& lhs,  // なで戻り値を const にしたかは
    const Rational& rhs);                               // 3 項を参照のこと
}
```

このコードでもし参照を返すことができれば、値渡しのコストを削減できます。
しかし、参照は何かの別名なので、必ず実態が必要です。

```C++
Rational a(1, 2);   // a は 1/2 を表す
Ratiobal b(3, 5);   // b は 3/5 を表す
Rational c = a * b; // c は 3/10 になる
```

上のコードでは、はじめから 3/10 を持つオブジェクトはないのでどこかに実態がなければなりません。
関数がオブジェクトを生成する方法は、スタック上かヒープ上の 2 つしかありません。
スタック上に生成する場合は以下のようになります。

```C++
const Rational& operator*(const Rational& lhs,  // 警告！悪いコード！
                          const Rational& rhs)
{
  Rational result(lhs.n * rhs.n, lhs.d * rhs.d);
  return result;
}
```

上のコードはローカルのオブジェクトの参照を返すので、動作未定義になってしまいます。
では、ヒープ上に生成する方法はどうでしょうか？

```C++
const Rational& operator*(const Rational& lhs,  // 警告！まだ、悪いコード！
                          const Rational& rhs)
{
  Rational *result = new Rational(lhs.n * rhs.n, lhs.d * rhs.d);
  return *result;
}
```

上の方法では new で生成されたオブジェクトを delete する方法がないという問題があります。

上記の 2 つの方法は共にコンストラクタが実行されてしまうので、static な Rational オブジェクトの参照を返す方法を考えます。

```C++
const Rational& operator*(const Rational& lhs,  // 警告！まだ、悪いコード！
                          const Rational& rhs)
{
  static Rational result;                       // static なオブジェクトの参照を返す

  result = ...;                                 // lhs と rhs の積を result に格納する
  return result;
}
```

上の処理ではスレッド安全性が問題になりそうです。また、以下のような間違いを誘発します。

```C++
bool operator==(const Rational& lhs,  // Rational のための operator==
                const Rational& rhs);
Rational a, b, c;
...
if ((a * b) == (c * d)) {
  積が等しいときの処理;
} else {
  積が等しくないときの処理;
}
```

上のコードでは、((a \* b) == (c \* d))は常に true になってしまいます。

結局正しいコードは以下のようなものになります。

```C++
inline const Rational operator*(const Rational& lhs, const Rational& rhs)
{
  return Rational(lhs.n * rhs.n, lhs.d * rhs.d);
}
```

operator* の戻り値に使われるオブジェクトの生成と破棄というコストが掛かりますが、結局これが正しいということになります。


**覚えておくこと**

* **スタック上のローカルなオブジェクトを指し示すポインタや参照、ヒープ上のオブジェクトへの参照を戻してはいけない。ローカルに static なオブジェクトを作って、そのオブジェクトを指し示すポインタや参照を戻すのも、そのオブジェクトが複数個必要なら、よくない。(4 項では、「ローカルに static なオブジェクト」への参照を返す、少なくとも単一スレッドの場合には合理的なコード例を紹介しました。ただし、それは、「ただ 1 つのオブジェクト」だけが必要な場合でした。)**

### 22 項 データメンバは private 宣言しよう

データメンバを private のみにした、以下のコードを考えます。

```C++
class AccessLevel {
public:
  ...
  int getReadOnly() const { return readOnly; }
  void setReadWrite(int value) { readWrite = value; }
  int getReadWrite() const { return readWrite; }
  void setWriteOnly(int value) { writeOnly = value; }
private:
  int noAccess;   // アクセス不可
  int readOnly;   // 読み出しのみ可能
  int readWrite;  // 読み書き可能
  int writeOnly;  // 書き込みのみ可能
}
```

上記のように、データメンバを private にすることで、自由にアクセスを制御可能になります。
さらに、カプセル化を考えると、あとからアクセス関数内で計算したものを返すといった変更も可能になります。
protected メンバにすると、そこから派生したクラスに変更を加える必要が発生し、どれぐらいの修正が必要かはわからなくなります。
データメンバが、 private でも protected でも、そのデータメンバが変更されると、大量のコードが使えなくなります。

**覚えておくこと**

* **データメンバは private 宣言しよう。それにより、データアクセスにおける構文の一貫性、制度の良いアクセス制御、不変な条件の保証、クラスの製作者が実装を変更できる柔軟性が得られる。**
* **protected は public よりカプセル化を進めるものではない。**

### 23 項 メンバ関数より、メンバでも friend でもない関数を使おう

以下のコードを考えます。

```C++
class WebBrowser {
public:
  ...
  void clearCache();
  void clearHistory();
  void removeCookies();
  ...
};
```

これらの関数を一度に実行したい場合

```C++
class WebBrowser {
public:
  ...
  void clearEverything(); // clearCache、clearHistory、
                          // removeCookie を呼び出す関数
  ...
};
```

と

```C++
void clearBrowser(WebBrowser& wb)
{
  wb.clearCache();
  wb.clearHistory();
  wb.removeCookie();
}
```

のどちらが良いのでしょうか？

カプセル化の観点から考えると、private な項目に一切アクセスできない後者のほうが好ましいのです。
これは、「メンバでも friend でもない関数」のほうがカプセル化をより強く維持できるからです。

また、機能ごとにヘッダを分割し、それぞれ独立したヘッダファイルで宣言しましょう。

```C++
// WebBrowser.h WebBrowser クラスのヘッダ
// WebBrowser 関連の便利関数のコア部分も含める
namespace WebBrowserStuff {
  class WebBrowser {...};
  ...                             // ほとんどのクライアントが必要とする
                                  // 非メンバ関数など、便利関数のコア部分
}

// webBrowserbookmarks.h
namespace WebBrowserStuff {
  ...                             // ブックマーク関連の便利機能
}

// webbrowsercookies.h
namespace WebBrowserStuff {       // クッキー関連の便利関数
  ...
}
...
```

これは、C++ の標準ライブラリの構成方法にもなっています。

**覚えておくこと**

* **メンバ関数より、メンバでも friend でもない関数を使おう。それによって、カプセル化の度合いを増し、柔軟性と機能拡張性をもたせることになる。**

### 24 項 すべての引数に型変換が必要なら、メンバでない関数を宣言しよう

以下のコードを考えます。

```C++
class Rational {
public:
  Rational(int numerator = 0,     // 意図的に explicit にせず、int から
           int denominator = 1);  // Rational への暗黙の型変換を許す
  int numerator() const;          // 分子と分母へのアクセス関数
  int denominator() const;        // 22 項を参照のこと
private:
  ...
};
```

さらに算術演算もサポートしたいのですが、この場合にメンバ関数にした場合

```C++
class Rational {
public:
  ...
  const Rational operator*(const Rational& rhs) const;
};
```

のようになります。
これで、オブジェクト同士の積の計算がとても簡単になります。

```C++
Rational oneEighth(1, 8);
Rational oneHalf(1, 2);
Rational result = oneHalf * oneEighth;  // OK
result = result * oneEighth;            // OK
```

しかし、Rational と int の積だと

```C++
result = oneHalf * 2; // OK
result = 2 * oneHalf; // エラー！
```

となってしまします。
これは下記のコードと同じ意味になります。

```C++
result = oneHalf.operator*(2);  // OK
result = 2.operator*(oneHalf);  // エラー！
```

このとき、コンパイラは

```C++
result = operator*(2, oneHalf); // エラー！
```

という呼び出しが可能かを調べますが、そのような operator\* は定義されていないのでエラーになります。
エラーにならない「result = oneHalf \* 2;」「result = oneHalf.operator\*(2);」について考えます。
これは

```C++
const Rational temp(2);   // Rational の一時オブジェクトを生成
result = oneHalf + temp;  // oneHalf.operator*(temp); と同じ
```

という処理になります。
これは、Rational のコンストラクタが explicit ではないから可能になるもので、もし、explicit 宣言されていたら両方共エラーになります。

```C++
result = oneHalf * 2; // エラー！explicit なコンストラクタは
                      // 呼び出されないので、2 が Rational
                      // オブジェクトに変換されない

result = 2 * oneHalf; // 前と同様に、エラー！
```

この問題を解決するには、operator* を非メンバ関数にすれば解決します。

```C++
class Rational {
  ...                                         // ここで operator* は定義しない
};
const Rational operator*(const Rational& lhs, // メンバ関数ではない
                         const Rational& rhs)
{
  return Rational(lhs.numerator() * rhs.numerator(),
                  lhs.denominator() * rhs.denominator());
}
Rational oneFourth(1, 4);
Rational result;
result = oneFourth * 2;                       // OK
result = 2 * oneFourth;                       // やった！大丈夫だ
```

ここで、operator* は、Rational の friend にするべきかを考えます。
公開されたインターフェースのみで実現できているため、答えは No です。
関数を friend にしないで済むのなら、するべきではないのです。

**覚えておくこと**

* **関数を呼び出すオブジェクト(this の指し示すオブジェクト)が暗黙の型変換で生成されることはない。そのような場合にも暗黙の型変換を利用したいなら、その関数を非メンバ関数にしよう。**

### 25 項 例外を投げない swap を考えよう

標準ライブラリの swap 処理は下記のようなものです。

```C++
namespace std {
  template<typename T>    // std::swap の典型的な実装
  void swap(T& a, T& b);  // a と b の値を交換する
  {
    T temp(a);
    a = b;
    b = temp;
  }
}
```

しかし、「データを保持するオブジェクトを指し示すポインタ」を持つ型ではこのようなコピーは不要だし、速度を落とします。
このような型のデザインは、「pimpl イディオム」と呼ばれます。

```C++
class WidgetImpl {                      // Widget クラスのデータ
public:                                 // ここで詳細は重要ではない
  ...
private:
  int a, b, c;                          // たくさんのデータがあり
  std::vector<double> v;                // コピーにコストがかかる
  ...
};

class Widget {                          // pimpl イディオムを使う
public:
  Widget(const Widget& rhs);
  Widget& operator=(const Widget& rhs)  // Widget のコピーは WidgetImpl のコピー
  {
    ...                                 // operator= の実装の詳細については
    *pImpl = *(rhs.pImpl);              // 10 項、11 項、12 項を参照のこと
    ...
  }
  ...
private:
  WidgetImpl *pImpl;                    // 実際のデータへのポインタ
};
```

この場合、「2 つの Widget オブジェクトの値の交換」で必要なのは、「ポインタ pImpl の交換」だけですが、std の swap では 3 つの Widget オブジェクトを WidgetImpl ごとコピーしてしまいます。
これは非常に非効率であるため、「pImpl を交換するだけでよい」ということを std::swap に明示的に指示する方法があります。
std::swap を Widget の場合に特化(特殊化)する方法です。

```C++
namespace std {
  tamplate<>                    // T が Widget の場合に使われる
  void swap<Widget>(Widget& a,  // std::swap の特別版
                    Widget& b)  // ただし、コンパイルできない
  {
    swap(a.pImpl, b.pImpl);     // Widget の交換では、それぞれの
  }                             // pImplを交換すればよい
}
```

一般的に、std の名前空間の内容を変えることは許されていませんが、自分で作った型に対してテンプレートを特化することは許されています。
しかし、上記のコードはコンパイルできません。a と b のプライベートメンバである pImpl にアクセスしているためです。
friend にすることで、一応回避はできますが、一般的には Widget に public なメンバ関数として swap を定義して、それを上記の swap で呼び出すようにします。

```C++
class Widget {                    // メンバ関数 swap を定義した
public:                           // それ以外は前と同じ
  ...
  void swap(Widget% other)
  {
    using std::swap;              // この宣言を個々に書く理由は
                                  // 本文を参照のこと

    swap(pImpl, other.pImpl);     // pImpl を交換する
  }
  ...
};

namespace std {
  template<>                      // std::swap の特化
  void swap<Widget>(Widget& a,
                    Widget& b)
  {
    a.swap(b);                    // Widget の交換に、メンバ関数の
  }                               // swap を呼び出す
}
```

これでコンパイルできるようになりました。

しかし、Widget と WidgetImpl がクラスではなく、データの型を示すテンプレートの場合を考えると問題があります。

```C++
template<typename T>
class WidgetImpl {...};

template<typename T>
class Widget {...};
```

この場合、std::swap の特化で問題にぶつかります。

```C++
namespace std {
  template<typename T>
  void swap<Widget<T> >(Widget<T>& a, // エラー！不正なコード！
                        Widget<T>& b)
  { a.swap(b); }
}
```

C++では、関数テンプレートの部分的な特化はできないためです。なので、このコードはコンパイルできません。
関数テンプレートの「部分的な特化」が必要な場合、普通はオーバーロードを使います。

```C++
namespace std {
  template<typename T>    // std::swap のオーバーロード
  void swap(Widget<T>& a, // swap の後に <...> がないことに注意
            Widget<T>& b) // しかし、このコードは有効ではない
  { a.swap(b); }          // 本文を参照のこと
}
```

std に新しいテンプレートを追加することはできないためです。

解決方法は、メンバ関数の swap を呼び出す非メンバの swap を、std::swap の特化やオーバーロードではないものにすることです。

```C++
namespace WidgetStuff {
  ...                     // テンプレート WidgetImpl など。
  template<typename T>    // 前と同じく、swap をメンバ関数
  class Widget {...};     // にする。
  ...
  template<typename T>    // メンバ関数でない swap
  void swap(Widget<T>& a, // std 名前空間に属さない
            Widget<T>& b)
  {
    a.swap(b);
  }
}
```

これで、C++ の名前検索ルールに従って、WidgetStuff 名前空間の swap が呼び出されることになります。
これはテンプレート以外でも動作しますが、残念ながらクラスの場合は、std::swap を特化した方が良い理由があります。
なので、特定のクラスに関して、可能な限り最良の swap を呼び出したい場合は、同じ名前空間に非メンバの swap を作成し、さらに std::swap の特化もする必要があります。

クライアント側(使用者側)の視点から考えてみます。

```C++
template<typename T>
void doSomething(T& obj1, T& obj2)
{
  ...
  swap(obj1, obj2);
  ...
}
```

上記のように書くと、どの swap が呼び出されるのでしょうか？
std 内の swap、std 内の特定クラスに特化したもの、std 外で定義された特定の型の swap などがあります。
特定の T に対して定義された swap があればそれを使い、無ければ一般の swap を使うようにしたい場合は、以下のようにします。

```C++
template<typename T>
void doSomething(T& obj1, T& obj2)
{
  using std::swap;    // std::swap を利用可能にする
  ...
  swap(obj1, obj2);   // 最適な swap を呼び出す
  ...
}
```

これで、C++ の名前検索ルールに従って正しい swap を使用するようになります。
ただし、下記のように呼び出す関数を std:: で修飾してはいけません。

```C++
std::swap(obj1, obj2);  // 誤った swap 呼び出し
```

上記の場合、std の swap を使用するように指示したことになります。
上記のように書くプログラマは多いので、自作クラスのために swap を書く場合、「std::swap を特化した swap」を書くことが重要になります。

まとめとしては、以下のようになります。

1. 効率的にオブジェクトを交換する public なメンバ関数として、そのクラスやクラステンプレートに、swap を書く。すぐあとで各理由のため、この関数は例外を投げないものにする。
2. メンバ関数でない swap を、そのクラスやテンプレートと同じ名前空間に書く。そして、その内部で、メンバ関数の swap を呼び出すようにする。
3. もし、クラステンプレートではなくクラスの場合は、そのクラスに対して特化した std::swap を書く。そして、その内部でも、メンバ関数の swap を呼び出すようにする。

メンバ関数の swap が、なぜ例外を投げてはいけないかについては、強い例外安全性をクラス(やテンプレート)に提供する助けになることが、swap の最も有用な役割の 1 つだからです。

**覚えておくこと**

* **自分の型について std::swap が非効率的な場合、メンバ関数として swap を書こう。そして、この swap は例外を投げないようにしよう。**
* **メンバ関数の swap を書いたら、その関数を呼び出す、メンバ関数でない swap を書こう。クラスについては(クラステンプレートは違う)、std::swap の特化も行おう。**
* **swap を呼び出すときは、std::swap の using 宣言をしよう。それから、名前空間の修飾なしに swap を呼び出すコードを書こう。**
* **std のテンプレートを自分のクラス用に特化(クラスを部分的にではなく、完全に指定する特化)することに問題はない。しかし、std にまったく新しいものを付け加えてはいけない。**

***

**[戻る](./index.md)**
