# Effective C++ メモ

## 第 1 章 C++ に慣れよう

### 1 項 C++ を複数の言語の連合と見なそう

C++ は

1. C
2. オブジェクト指向 C++
3. テンプレートC++
4. STL

という4つのサブセットの連合と考える。

### 2 項 #define より、const、enum、inline を使おう

* #define マクロでは、定義名がシンボルテーブルに乗らないため、デバッグがやりにくくなる。
* #define マクロではすべての場所に変数が置かれるが、 const ならば変数は 1 箇所にしか存在しないため、無駄なメモリを消費しない。

なので、 const に置き換えよう。

さらに 2 つの注意点がある。

1. 定数ポインタには 2 つの const が必要。

    ```C++
    const char* const name = "xxxxx";
    ```

    通常は

    ```C++
    const std::string name("xxxxx");
    ```

    とする。

2. クラスのメンバ減数を定数にする場合は、 static にすること。

    ```C++
    class MyClass {
    private:
        static const int Num = 5;
        int values[Num];
        ...
    }
    ```

アドレスの取得などを制限したい場合は、 enum ハックが使える。

```C++
class MyClass {
private:
    enum {Num = 5}; // 宣言時初期化ができないコンパイラでも使用でき、マクロのようにアドレス取得されることもないしメモリも無駄に消費しない
    int Values[Num];
}
```

マクロとしても使用しないほうがいい。

悪い例

```C++
// a、b の大きい方を使って f を呼び出す
#define CALL_WITH_MAX(a,b) f((a) > (b) ? (a):(b))
```

上記のコードでは、値によって a が 2 回インクリメントされる場合と 1 回インクリメントされる場合が発生してしまう。

テンプレートに置き換えた例

```C++
template<typename T>
inline void callWithMax(const T& a, const T& b)
{
    f(a > b ? a : b);
}
```

***まとめ***

**単純な定数には、 #define より、 const か enum を使うようにしよう。**
**#define で定義するマクロより、インライン関数を使うように使用。**

***

### 3 項 可能ならいつでも const を使おう

ポインタの const の位置と意味

```C++
char greeting[] = "Hello";
cha *p = greeting;                  // ポインタは非 const
                                    // データも非 const

const char *p = greeting;           // ポインタは非 const
                                    // データは const

char * const p = greeting;          // ポインタは const
                                    // データは非 const

const char * const p = greeting;    // ポインタは const
                                    // データも const
```

つまり

* const が * の左にあれば「ポインタが指し示すデータ」が不変
* const が * の右にあれば「ポインタそのもの」が不変

になる。

ポインタが指し示すものを不変にする場合、 const は型名の前でも後ろでも意味は同じ。

```C++
void f1(const Widget *pw);  // f1 の引数は「変更不可の Widget オブジェクト」へのポインタ

void f2(Widget const *pw);  // f2 も同じ
```

STL の iterator はポインタをモデルにしているので以下の様になる。

```C++
std:: vector<int> vec;
...

const std::vector<int>::iterator iter = vec.begein();   // T* const のように振舞う反復子
*iter = 10;                                             // iter が示すものの内容を変更しても問題ない
++iter;                                                 // エラー！ iter は const だから

std::vector<int>::const_iterator cIter = vec.begin();   // cIter は const T* のように振舞う
*cIter = 10;                                            // エラー！ *cIter は const だから
++cIter;                                                // cIter を変えても問題ない
```

関数の戻り値を const にすると、安全性や効率を犠牲にせずに、関数利用者が誤用する可能性を下げることができる。

```C++
class Rational {...};
const Rational operator*(const Rational& lhs, const Rational& rhs);
```

このとき、戻り値が const でないと、この関数の利用者は次のようなコードを書けてしまう。

```C++
Rational a, b, c;
...
(a * b) = c;    // a * b の結果に対して = が使われる
```

また、以下のようなタイプミスを許容してしまいます。

```C++
if (a * b = c) ...  // おっと、比較のつもりだったのに！
```

const をつけることで、上記のようなミスを避けられるのです。

#### const なメンバ関数

メンバ関数に const を使う理由は下記の 2 つ。

* どのメンバ関数がオブジェクトを変更し、どのメンバ関数が変更しないかを容易に判断できるようにする。
* const なオブジェクトに対して使えるため。

```C++
class TextBlock {
public:
...
    const char& operator[](std::size_t position) const  // const なオブジェクトのための [] 演算子
    {return text[position];}

    char& operator[](std::size_t position)              // const でないオブジェクトのための [] 演算子
    {return text[position];}

private:
    std::string text;
};
```

これで、 const でない TextBlock に対しても、 const な TextBlock に対しても、 [] 演算子を次のように使える。

```C++
TextBlock tb("Hello");
std::cout << tb[0];             // const でない TextBlock::operator[] を呼び出す
const TextBlpock ctb("World");
std::cout << ctb[0];            // cont な TextBlock::operator[] を呼び出す
```

もっと現実的なコードでは

```C++
void print(const TextBlock& ctb)    // この関数内で ctb は const
{
    std::cout << ctb[0];            // const な TextBlock::operator[] を呼び出す
    ...
}
```

のように使う。

const 付き [] オペレータには、戻り値にも const を付けているので、 const な TextBlock を const でない TextBlock とは全く違うように扱えます。

```C++
std::cout << tb[0];     // 問題なし
                        // const でない TextBlock からの読み出し(と出力)

tb[0]='x';              // 問題なし
                        // const でない TextBlock への書き込み

std::cout << ctb[0];    // 問題なし
                        // const な TextBlock からの読み出し(と出力)

ctb[0]='x';             // エラー！
                        // const な TextBlock への書き込み
```

メンバ関数の const には

* ビットレベルの不変性
* 論理的な不変性

という 2 つの考え方がある。

const は、ビットレベルの不変性を示しているが、以下のような場合は？

```C++
class CTextBlock {
public:
    ...
    char& operator[](std::size_t position) const    // 不適切な宣言だが、
    {return pText[position];}                       // 「ビットレベルの不変性」はある
private:
    char *pText;
};
```

この場合、 pText が指し示すデータは変更されてしまう。

```C++
const CTextBlock cctb("Hello"); // const なオブジェクトの宣言

char *pc = &cctb[0];            // const な [] 演算子を呼び出し、 cctb のデータポインタを得る

*pc = 'J';                      // ここで cctb の文字列は "Jello" になる
```

const なメンバ関数呼び出しなのにデータが変更されてしまいます。
このような事実から「論理的な不変性」が生まれた。

```C++
class CTextBlock {
public:
    ...
    std::size_t length() const;
private:
    char *pText;
    std::size_t textLength;     // 最後に調べられたときの文字列の長さ
    bool lengthIsValid;         // 長さが変更されていないか
};

std::size_t CTextBlock::length() const
{
    if (!lengthIsValid) {
        textLength = std::strlen(pText);    // エラー！ const なメンバ関数で
        lengthIsValid = true;               // textLength と lengthIsValid に代入はできない
    }
    return textLength;
}
```

上記の例では、メンバ関数 length が、 textLength と lengthIsValid の値を変えてしまうので、「ビットレベルの不変性」を持っていません。しかし、外部から変更は見えないの const な CTextBlock オブジェクトであっても良いように思えます。

では、どうするか？
**mutable** を使います。 mutable は static でないデータメンバを「ビットレベルの不変性」の制約から開放します。

```C++
class CTextBlock {
public:
    ...
    std::size_t length() const;
private:
    char *pText;
    mutable std::size_t textLength; // これらのデータメンバはどこでも
    mutable bool lengthIsValid;     // (const なメンバ関数でも)変更できる
};

std::size_t CTextBlock::length() const
{
    if (!lengthIsValid) {
        textLength = std::strlen(pText);    // 問題なし
        lengthIsValid = true;               // こちらも問題なし
    }
    return textLength;
}
```

#### const なメンバ関数と非 cont なメンバ関数の重複を取り除く

すべての関数に const と、非 const の両方の関数を用意し、さらにそれをインライン化するとクラス定義がひどいことになります。

```C++
class TextBlock {
public:
    ...
    const char& operator[](std::size_t position) const
    {
        ... // 境界超えのチェック
        ... // アクセスログを取る
        ... // データの整合性をチェック
        return text[position];
    }
    char& operator[](std::size_t position)
    {
        ... // 境界超えのチェック
        ... // アクセスログを取る
        ... // データの整合性をチェック
        return text[position];
    }
private:
    std::string text;
};
```

上記のコードは、コードがかなり重複しています。
これを解決するために、非 const な関数から const な関数を呼び出します。

```C++
class TextBlock {
public:
    ...
    const char& operator[](std::size_t position) const  // 前と同じ
    {
        ...
        ...
        ...
        return text[position];
    }
    char& operator[](std::size_t position)          // 単に const な [] を呼び出すだけ
    {
        return
          const_cast<char&>(                        // [] の戻り値から const をキャストではずす
            static_cast<const TextBlock&>(*this)    // const を *this に付けて const な [] を呼び出す
            [position]
          );
    }
    ...
};
```

見栄えはあまり良くないですが、重複は避けられました。
ここでは、 const な関数を非 const な関数から呼び出していますが、その逆はNGです。
非 const な関数は、オブジェクトを変更する可能性があるため、 const な関数から呼び出すのはおかしいからです。

***まとめ***

* **const を付けて宣言すると、コンパイラがそのオブジェクトの誤用を検出してくれる。 const は、あらゆるスコープのオブジェクト、関数の仮引数と戻り値、メンバ関数自体につけることができる。**
* **コンパイラは const に対し、「ビットレベルの不変性」を保証する。しかし、「論理的不変性(概念的不変性)」を保証するようなコードを書くべき。**
* **const と非 const なメンバ関数で、本質的に同じ実装をする必要がある場合、非 const なメンバ関数内で const なメンバ関数を呼び出し、コードの重複を避けることができる。**

### 4 項 オブジェクトは、使う前に初期化しよう

オブジェクトは、使う前に必ず初期化するようにしましょう。

クラスのメンバでない組み込み型の初期化は以下のようになります。

```C++
int x = 0;                              // int の初期化

const char* text = "A C-style string";  // ポインタの初期化

double d;                               // 入力ストリームからの
std::cin >> d;                          // 読み込みによる初期化
```

オブジェクト内のすべてのデータはコンストラクタで初期化しましょう。

```C++
class PhoneNumber {...};

class ABEntry {             // Address Book Entry (アドレス帳のデータ) を表す
public:
    ABEntry(cont std::string& name, const std::string& address, const std::list<PhoneNumber>& phone);

private:
    std::string theName;
    std::string theAddress;
    std::list<PhoneNumber> thePhone;
    int numTimesConsulted;
};

ABEntry::ABEntry(const std::string& name, const std::string& address, const std::list<PhoneNumber>& phones)
{
    theNumber = name;           // これらは代入であって初期化ではない
    theAddress = address;
    thePhones = phones;
    numTImesConsulted = 0;
}
```

上記のコンストラクタで行っているのは**変数への代入であって初期化ではありません**。

一般には、以下のようにメンバ初期化子リストを使います。

```C++
ABEntry::ABEntry(const std::string& name, const std::string& address, const std::list<PhoneNumber>& phones)
:   theName(name),          // データメンバのすべてを初期化する
    theAddress(address),
    thePhones(phones),
    numTimesCOnsulted(0)
{}                          // コンストラクタ本体は空になる
```

デフォルトコンストラクタで初期化するデータメンバに対しても、以下のように初期化子を使用できます。

```C++
ABEntry::ABEntry()
:   theName(),              // theName に関してデフォルトコンストラクタを呼び出す
    theAddress(),           // theAddress でも同様
    thePhones(),            // thePhones でも同様
    numTimesConsulted(0)    // ここでは明示的に 0 に初期化
{}
```

オブジェクトのデータ初期化の順番は、基底クラス→派生クラスの順に初期化され、オブジェクト内での順番はデータメンバが宣言された順に初期化されます。

ちょっと複雑ですが、「異なる翻訳単位で定義されたローカルでない静的オブジェクトの初期化の順番は決められていない」という問題があります。

```C++
class FileSystem {                  // このクラスはライブラリ内にあるとする
public:
    ...
    std::size_t numDisks() const;   // たくさんあるメンバ関数のうちの 1 つ
    ...
};

extern FileSystem tfs;              // クライアントが使うためのオブジェクトの宣言
                                    // tfs は「the file system」の意
```

上記の tfs が「ローカルでない静的オブジェクト」の宣言です。
さらに以下のクラスがあるとします。

```C++
class Directory {                       // ライブラリのクライアントが書いたクラス
public:
    Directory(params);
    ...
};

Directory::Directory(params)
{
    ...
    std::size_t disks = tfs.numDisk();  // tfs を使う
}
```

さらに、クライアントは、テンポラリのディレクトリを以下のように作成したとします。

```C++
Directory tempDir(params);
```

tempDir も「ローカルでない静的オブジェクト」です。
この場合、 tfs と tempDir は「異なる翻訳単位にあるローカルでない静的オブジェクト」です。
このとき、 **tfs が tempDir よりも先に初期化される保証はないのです**。

この問題を解決するには、以下のように「ローカルでない静的オブジェクト」を「ローカルな static オブジェクト」に変更します。

```C++
class FileSystem {...};                     // 以前と同じ
FileSystem& tfs()                           // この関数が tfs の代わりになる
{                                           // FileSystem クラスの static な関数にしてもよい

    static FileSystem fs;                   // fs をローカルな static オブジェクトとして定義・初期化
    return fs;                              // その参照を返す
}

class Directory {...};                      // 以前と同じ

DIrectory::Directory(params)                // 前と同じだが、 tfs を tfs() に置き換えた
{
    ...
    std::size_t disks = tfs().numDisks();
    ...
}

Directory& tempDir()                        // この関数が tempDir の代わりになる
{                                           // Directory クラスの static 関数としてもよい

    static Directory td(params);            // td をローカルな static オブジェクトとして定義・初期化
    return td;                              // その参照を戻す
}
```

***まとめ***

* **組み込み型のオブジェクトは自動で初期化されるとは限らない。そこで、プログラマが初期化するコードを書こう。**
* **コンストラクタでは、代入を行うより、初期化子リストを使うようにしよう。その際、初期化子の順番は、クラス内でデータメンバを宣言するのと同じ順番にしよう。**
* **異なる翻訳単位にある「ローカルでない静的オブジェクト」の初期化の順番は決められていない。「ローカルでない静的オブジェクト」を「関数に対する static なオブジェクト」に置き換えることで、この問題を避けることができる。**

***

[戻る](./index.md)
