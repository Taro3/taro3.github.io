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

以下のコードを考えます。

```C++
class Point {             // 点を表すクラス
public:
  Point(int x, int y);
  ...
  void setX(int newVal);
  void setY(int newVal);
  ...
};

struct RectData {         // Ractangle のデータ
  Point ulhc;             // 左上の点(upper left-hand corner)
  Point lrhc;             // 右下の点(lower right-hand corner)
};

class Rectangle {
public:
  ...
  Point& upperLeft() const { return pData->ulhc; }
  Point& lowerRight() const { return pData->lrhc; }
  ...
private:
  std::shared_ptr<RectData> pData;
}
```

このコードでは、矩形の左上座標と右下座標をクラス外(Rectangle クラス外)に持っています。
そして、その座標を取得する upperLeft と lowerRight 関数があります。

しかし、upperLeft と lowerRight は const 宣言されているにも関わらず、呼び出し元で変更できてしまうという問題があります。

```C++
Point coord1(0, 0);
Point coord2(100, 100);
const Rectangle rec(coord1, coord2);  // rec は const なオブジェクト
                                      // 左上の点が(0, 0)、右下の点が(100, 100)

rec.upperLeft().setX(50);             // これで、rec の左上の点が(50, 0)になる
```

rec が const であるにも関わらずデータを変更できてしまいます。

このようにメンバ関数が「内部」のデータへの参照を返す場合、そのクラスはあまり強くカプセル化されていないということになります。
また、const なメンバ関数が「実際にはオブジェクトの外部に置いたデータオブジェクト」への参照を返す場合、その関数の呼び出し元でデータを変更できてしまうということです。これは、ポインタや反復子の場合も同じ問題があります。
参照、ポインタ、反復子はハンドルと呼ばれます。
また、public でないメンバ関数へのハンドルも戻さないことが重要です。そのポインタを使って「公開されない(はずの)関数」も呼び出されてしまうためです。

先程の Rectangle の問題は、戻り値に const を付けることで解決できてしまいます。

```C++
class Rectangle {
public:
  ...
  const Point& upperLeft() const { return pData->ulhc; }
  const Point& lowerRight() const { return pData->lrhc; }
  ...
}
```

このようにすれば、呼び出し側での変更は行えなくなります。
しかし、これでも「内部」へのハンドルを返すため、別の問題が発生する可能性があります。特に「どこも指し示さないハンドル」の問題です。
下記のコードを考えます。

```C++
class GUIObject {...};
const Rectangle boundingBox(const GUIObject& obj);  // Rectangle を値で返す
```

上記の関数は次のように使用できます。

```C++
GUIObject *pgo;
...                                     // pgo が GUIObject を指し示すようにする

const Point *pUpperLeft =               // GUI オブジェクトの範囲を表す矩形の
    &(boundingBox(*pgo).upperLeft());   // 左上の点へのポインタを取得
```

このとき、boundingBox は Rectangle の一時オブジェクトを生成します。upperLeft は、その一時オブジェクトに対して呼ばれます。
一時オブジェクトの左上の座標が返されるわけですが、この行の実行が終わると、当然ですが、一時オブジェクトは破棄されます。
結果として、pUpperLeft は、すでに破棄された座標を指し示すことになります。
これが、「内部」へのハンドルを返す関数が危険だという理由です。

#### 覚えておくこと

* **オブジェクト内部のデータへのハンドル(参照、ポインタ、反復子)を返す関数は避けよう。そうすることで、カプセル性を高め、const なメンバ関数を概念的にも const にし、「どこも指さないハンドル」を生成してしまう可能性を減らすことができる。**

### 29 項 コードを例外安全なものにしよう

以下のコードを考えます。

```C++
class PrettyMenu {
public:
  ...
  void changeBackground(std::istream& imgSrc);  // 背景の画像を変える
  ...
private:
  Mutex mutex;      // このオブジェクトのミューテックス
  Image *bgImage;   // 現在の背景画像
  int imageChanges; // 画像が変更された回数
};
```

changeBackground の実装は下記のようになるでしょう。

```C++
void PrettyMenu::changeBackground(std::istream& imgSrc)
{
  lock(&mutex);                 // ミューテックスのロック
  delete bgImage;               // 古い背景画像を破棄
  ++imageChanges;               // 変更回数を増やす
  bgImage = new Image(imgSrc);  // 新しい背景画像に変える
  unlock(&mutex);               // ミューテックスのアンロック
}
```

しかし、上記のコードは、下記の 2 つの例外安全性をどちらも満たしていません。

* **リソース漏れをおこさない** 上のコードはこの条件を満たしていません。「new Image(imgSrc)」が例外を投げると、ミューテックスの開放が行われません。
* **データ構造が無効な状態(正しくない状態)になることを防ぐ** 「new Image(imgSrc)」が例外を投げると、bgImage が「破棄されたオブジェクト」を指し示したままになります。また、新しい画像に変わっていないにもかかわらず、imageChanges がインクリメントされます。

リソース漏れに関しては、リソース管理オブジェクトを使用することで解決できます。

```C++
void PrettyMenu::changeBackground(std::istream& imgSrc)
{
  Lock ml(&mutex);  // ミューテックスをロックし、
                    // あとで確実にアンロック
  delete bgImage;
  ++imageChanges;
  bgImage = new Image(imgSrc);
}
```

次に、データ保護について考えます。
データ構造の保護に関して、例外安全な関数は、次の 3 つのうちの 1 つを保証しなければなりません。

* **関数が例外に対し基本保証(basic guarantee)をする** これは、「例外が投げられても、プログラム中のすべてのものの状態が有効に保たれる」ということです。たとえば、「changeBackground で例外が投げられたら、PrettyMenu は古い背景画像かデフォルトの背景画像を持つことになる。ただし、どちらになるかは状況次第」になります。
* **関数が例外に対し強い保証(strong guarantee)をする** これは、「例外が投げられた場合、プログラムの状態は、その関数を呼び出し前のものに戻される」ということです。
* **関数が例外に対し投げない保証(nothrow grarantee)をする** これは、「必ず予定の仕事を実行し、例外を投げない」ということです。

ほぼ強い保証を行うには下記のようにします。

```C++
class PrettyMenu {
  ...
  std::shared_ptr<Image> bgImage;
  ...
};

void PrettyMenu::changeBackground(std::istream& imgSrc)
{
  Lock ml(&mutex);
  bgImage.reset(new Image(imgSrc)); // bgImage の内部ポインタを
                                    // 「new Image」の戻り値に変える
  ++imageChanges;
}
```

これで、古い画像を破棄するコードを書く必要はありません。これで「ほぼ強い保証」ができるようになりました。
しかし、「ほぼ」です。問題は imgSrc です。Image のコンストラクタが例外を投げるかもしれないのです。

ここでは、Image のコンストラクタも強い保証をするものとしましょう。
強い保証をするための、一般的な設計に「コピーと交換(スワップ)」と呼ばれるものです。

```C++
struct PMImpl {                   // PrettyMenu の実装オブジェクト
  std::shared_ptr<Image> bgImage; // 構造体にした理由は後述
  imt imageChanges;
};

class PrettyMenu {
  ...
private:
  Mutex mutex;
  std::shared_ptr<PMImpl> pImpl;
};

void PrettyMenu::changeBackground(std::istream& imgSrc)
{
  using std::swap;
  Lock ml(&mutex);                        // ミューテックスロック
  std::shared_ptr<PMImpl>                 // データのコピー
      pNew(new PMImpl(*pImpl));
  pNew->bgImage.reset(new Image(imgSrc)); // コピーの変更
  ++pNew->imageChanges;
  swap(pImpl, pNew);                      // オブジェクトの交換
}                                         // ミューテックスのアンロック
```

「コピーと交換」は、優れた方法ですが、残念ながら、関数全体を強い保証にできるとは限りません。

```C++
void someFunc()
{
  ...   // ローカルな状態のコピーを作る
  f1();
  f2();
  ...   // 変更したコピーと元のものを交換する
}
```

f1 と f2 が例外安全性に関して強い保証をしない関数なら、someFunc が強い保証をするのは難しいでしょう。実際には両関数が強い保証をしたとしても、f2 が例外を投げると元の状態には戻せなくなります。また、必ずコピーが発生するため、パフォーマンスの問題もあるでしょう。
結論として、例外安全性に関する保証をまったくしないコードを書くべきではありません。できる範囲で保証されたコードを書きましょう。

#### 覚えておくこと

* **例外安全な関数とは、例外が投げられても、リソース漏れを起こさず、データを有効に保つ関数のこと。そのような関数には、基本保証をするもの、強い保証をするもの、投げない保証をするものがある。**
* **つよう保証は、しばしば、「コピーと交換」で実装できる。しかし、強い保証が、すべての関数で実用的であるわけではない。**
* **ある関数内で別の関数を呼び出す場合、「呼び出される関数」よりよい保証を「呼び出す関数」ですることは、通常、とても困難である。**

### 30 項 インラインをよく理解しよう

#### 覚えておくこと

* **普通、インライン化を要請する関数は、小さくてしばしば呼び出されるものに限定しよう。それにより、デバッガを使いやすくし、バイナリによるアップデートをしやすくし、コードの膨張を防ぎ、プログラムの効率をよくすることができる。**

  これは、インライン化には様々なコストが掛かるので当然ですね。

* **定義がヘッダファイル内にあるからといって、それだけの理由で、関数テンプレートをインライン宣言しないこと。**
  当然テンプレート宣言がヘッダにあっても、インライン化を望まないならば inline キーワードを使用するべきではないということです。

### 31 項 ファイル間のコンパイル依存性をなるべき小さくしよう

#### 覚えておくこと

* **コンパイル依存性をできるだけ小さくするための一般的な考え方は、「定義ではなく宣言に依存するようにコードを書く」というもの。この考えに基づいて、ハンドルクラスの方法とインターフェースクラスの方法がある。**
  これは、pImpl などを使って定義ではなく宣言依存にする方法(ハンドルクラス)と、仮想関数を使用したインターフェースを使う方法のこと。実態ではなく、ポインタや参照を使うことで、定義ではなく宣言に依存した実装にすることが可能になる。

* **ライブラリのヘッダファイルは、すべてのコンポーネントに関して、「宣言のみ」を含むようにするべき。これはテンプレートに関しても可能。**
  ライブラリでは特に宣言依存にしておくことが重要ということ。

**[戻る](./index.md)**
