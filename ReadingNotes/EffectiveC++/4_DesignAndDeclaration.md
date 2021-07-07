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
