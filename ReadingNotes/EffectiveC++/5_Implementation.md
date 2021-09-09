# Effective C++ メモ

## 第 5 章 実装

### 26 項 変数の定義は可能な限り先延ばししよう

以下のコードを考えます。

```C++
// この関数で、encrypted の定義は早すぎる
std::string enctyptPassword(const std::string& password)
{
  using namespace std;
  string encrypted;       // 暗号化した文字列を格納するための変数
  if (password.length() < MinimumPasswordLength) {
    throw logic_error("Password is too short");
  }
  ...                     // password の文字列を暗号化したものを encrypted に格納する
  return encrypted;
}
```

上記のコードでは、encrypted は例外が投げられると使われません。
なので、実際に使われることがはっきりしてから encrypted を定義するほうがよいでしょう。

```C++
// この関数は本当に必要になるまで encrypted の定義をしない
std::string encryptPassword(const std::string& password)
{
  using namespace std;
  if (password.length() < MinimumPasswordLength) {
    throw logic_error("Password is too short");
  }
  string enctypted;
  ...                   // password の文字列を暗号化したものを encrypted に格納する
  return encrypted;
}
```

しかし、encrypted は引数なしで生成しているので、まだ無駄なコードがあります。
encrypted は、下記の関数で使用されるとします。

```C++
void encrypt(std::string& s); // s を暗号化する
```

すると、encryptedPassword の実装は、下記のようになるでしょう。

```C++
std::string encryptPassword(const std::string& password)
{
  ...                   // password の長さをチェック
  string encrypted;     // encrypted を引数なしに生成
  encrypted = password; // encrypted に password を代入

  encrypt(encrypted);
  return encrypted;
}
```

さらに、よりよい方法は、encrypted を password で初期化するというものです。

```C++
std::string encryptPassword(const std::string& password)
{
  ...                         // password の長さをチェック
  string encrypted(password); // コピーコンストラクタを使って
                              // encrypted を定義し、初期化する
  encrypt(encrypted);
  return encrypted;
}
```

しかし、ループ内で使用する場合はどうでしょうか？

```C++
// 方法 A: ループの外で定義           // 方法 B: ループの中で定義
Widget w;                       
for (int i = 0; i < n; ++i) {         for (int i = 0; i < n; ++i) {
  w = i に依存する値;                   Widget w(iに依存する値);
  ...                                   ...
}                                     }
```

上記の場合

* 方法 A: 生成 1 回、破棄 1 回、代入 n 回
* 方法 B: 生成 n 回、破棄 n 回

となります。

一般的には、方法 B を使うべきですが、「(1) 1 回の代入が 1 組の生成と破棄よりコストがかからない。かつ、(2) その部分では、効率が重要である」という場合を除きます。

#### 覚えておくこと

* **変数の定義は可能な限り先延ばししよう。それにより、プログラムは明快になり、効率的にもなる。**

### 27 項 キャストは最小限にしよう

下記のコードを考えます。

```C++
class Base {...};
class Derived: public Base {...};
Derived d;
Base *pb = &d;                    // 暗黙に(非明示的に) Derived を Base* に変換
```

最後の行で、派生クラスのオブジェクトを指す基底クラスのポインタを作っています。
このとき、ポインタに格納される値はが**型の変換前と変換後で同じだとは限りません**。

このようなことは、C や Java や C# では起こりません。しかし、C++ では多重継承を使用するとほぼそうなります。単一継承でも起こる場合もあります。

例えばウィンドウアプリケーションのフレームワークがあったとして、そのフレームワークでは、仮想関数を実装する場合、まずはじめに基底クラスの関数を呼び出す必要があるとします。

```C++
class Window {                              // 基底クラス
public:
  virtual void onResize() {...}             // 基底クラスの onResize の実装
  ...
};

class SpecialWindow : public Window {       // 派生クラス
public:
  virtual void onResize() {                 // 派生クラスの onResize の実装
    static_cast<Window>(*this).onResize();  // *this を Window にキャストし
                                            // その onResize を呼び出す
                                            // しかし、これはうまく行かない！

    ...                                     // SpecialWindow 独自のコード
  }
  ...
};
```

上のコードの static_cast は、*this を Window にキャストしています。なので、Window::onResize を呼び出しますが、このとき呼び出される onResize は、このオブジェクトのものではなく、「\*this の基底クラス部分」のコピーとして、新しい一時オブジェクトを生成してしまいます。
この問題を避ける方法は、キャストを行わないことです。

```C++
class SpecialWindow : public Window {
public:
  virtual void onResize() {
    Window::onResize();               // Window::onResize を *this に対して呼び出す
    ...
  }
  ...
};
```

dynamic_cast の使用には、更に注意が必要です。dynamic_cast は非常に遅くなります。
間違っても、下記のような連続した dynamic_cast の使用は避けるべきです。

```C++
class Window {...};
...                                                                       // ここで派生クラスを定義
typedef std::vector<std::shared_ptr<Window>> VPW;
VPW winPtrs;
...
for (VPW::iterator iter = winPrts.begin(); iter != winPtrs.end(); ++iter)
{
  if (SpecialWindow1 *psw1 =
      dynamic_cast<SpecialWindow1*>(iter->get())) {...}
  else if (SPecialWindow2 psw2 =
      dynamic_cast<SpecialWindow2*>(iter->get())) {...}
  else if (SPecialWindow3 psw3 =
      dynamic_cast<SpecialWindow3*>(iter->get())) {...}
  ...
}
```

#### 覚えておくこと

* **可能な限りキャストを避けよう。特に、効率が重要な場合、dynamic_cast を避けよう。もし、設計上キャストが必要と感じたら、キャストを使わない代替案を考えてみよう。**
* **キャストが避けられない場合、それは関数の中に隠蔽してしまおう。そうすれば、その関数のクライアントは、自分のコード内でキャストを使わなく済むようになる。**
* **古い C スタイルのものより C++ スタイルのキャストを使おう。C++ スタイルのキャストは、見やすく、何をするかについても、よりはっきりしているから。**

### 28 項 オブジェクト内部のデータへの「ハンドル」を戻さないようにしよう
