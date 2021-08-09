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

***覚えておくこと***

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

***覚えておくおこと***

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

***覚えておくこと***

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


***覚えておくこと***

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

***覚えておくこと***

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

***覚えておくこと***
* **メンバ関数より、メンバでも friend でもない関数を使おう。それによって、カプセル化の度合いを増し、柔軟性と機能拡張性をもたせることになる。**

### 24 項 すべての引数に型変換が必要なら、メンバでない関数を宣言しよう
