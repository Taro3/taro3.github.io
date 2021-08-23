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
