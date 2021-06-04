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

***覚えておくこと***

* **リソース漏れを避けるために、RAII オブジェクトを使おう。RAII オブジェクトはコンストラクタでリソースを受け取り、デストラクタでそれを破棄する。**
* **unique_ptr と shared_ptr は、一般的に有用な RAII クラス。ただし、「コピーが自然なもの」が必要なら、unique_ptr より shared_ptr がよい。unique_ptr をコピーすると、コピー元はヌルになる。**

### 14 項 リソース管理クラスのコピーの振る舞いはよく考えて決めよう
