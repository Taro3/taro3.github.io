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

