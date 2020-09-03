# Q_OBJECTとシグナル/スロットのもとで

ここで、Qtの構築システムがより明確になるはずです。それでも、Q_OBJECTマクロとsignal / slot / emitキーワードはブラックボックスのままです。 Q_OBJECTを調べましょう。

真実はソースコードにあります。 Q_OBJECTはファイルqobjectdefs.hで定義されています（Qt 5.7）：

```C++
#define Q_OBJECT \
public: \
    // skipped details
    static const QMetaObject staticMetaObject; \
    virtual const QMetaObject *metaObject() const; \
    virtual void *qt_metacast(const char *); \
    virtual int qt_metacall(QMetaObject::Call, int, void **); \
    QT_TR_FUNCTIONS \
private: \
    // skipped details
qt_static_metacall(QObject *, QMetaObject::Call, int, void **);
```

このマクロは、いくつかの静的関数と static QMetaObject を定義します。これらの静的関数の本体は、生成された moc ファイルに実装されています。QMetaObject クラスの詳細については、ここでは説明しません。このクラスの役割は、QObject サブクラスのすべてのメタ情報を格納することです。また、あなたのクラスのシグナルとスロット、そして接続されたクラスのシグナルとスロットの間の対応表を保持します。各シグナルと各スロットには一意のインデックスが割り当てられます。

* metaObject()関数は、通常のQtクラスの場合は&staticMetaObjectを、QMLオブジェクトを扱う場合はdynamicMetaObjectを返します。
* qt_metacast() 関数は、クラス名を使用して動的キャストを実行します。Qtでは、オブジェクトまたはクラスに関するメタデータの取得に標準のC++ RTTI（Runtime Type Information）を使用しないため、この関数が必要です。
* qt_metacall()は、内部シグナルまたはスロットをインデックスで直接呼び出します。ポインタではなくインデックスが使用されるため、ポインタの派生参照がなく、生成されたスイッチケースはコンパイラによって大幅に最適化されます（コンパイラは、特定のケースへのジャンプ命令を非常に早い段階で直接含めることができ、多くの分岐評価を避けることができます）。このように、シグナル/スロット機構の実行はかなり高速である。

Qt では、シグナル/スロットの仕組みを管理するために、signals、slots、emitという非標準の C++ キーワードも追加されています。それぞれのキーワードの背後にあるものを見て、connect() 関数の中にどのように収まるのかを見てみましょう。

slots と signals キーワードも qobjectdefs.h で定義されています。

```C++
# define slots
# define signals public
```

見ての通りです。slotsは何も指しておらず、signalsキーワードはpublicキーワードのプレースホルダに過ぎません。あなたの signals/slots はすべて、ただの...関数です。シグナル関数がクラスの外から見えるようにするために、signalsキーワードは強制的にpublicにされています（そもそもprivate signalのポイントは何なのでしょうか？)Qtマジックとは、このslotを実装しているクラスの詳細を知らなくても、接続されたslotキーワードにsignalキーワードを出力する機能のことです。すべては moc ファイルの QMetaObject クラスの実装を通して行われます。signalキーワードがエミットされると、変更された値とsignalsのインデックスを指定して関数QMetaObject::activate()が呼び出されます。

勉強する最後の定義はemitです。

```C++
# define emit
```

無意味なものの定義が多すぎて、ほとんど無茶苦茶です。emitキーワードはコード的には何の役にも立ちません。moc はそれを無視しているだけで、その後は特に何も起こりません。これは、開発者が単純な関数ではなく、シグナル/スロットを使っていることに気づくためのヒントに過ぎません。

スロットをトリガするには、signalキーワードを接続して QObject::connect() 関数を使用します。この関数はqobject_p.hで定義されている新しいConnectionインスタンスを作成します。

```C++
struct Connection
{
    QObject *sender;
    QObject *receiver;
    union {
        StaticMetaCallFunction callFunction;
        QtPrivate::QSlotObjectBase *slotObj;
    };
    // The next pointer for the singly-linked ConnectionList
    Connection *nextConnectionList;
    //senders linked list
    Connection *next;
    Connection **prev;
    ...
};
```

Connectionインスタンスには、シグナルエミッタクラス(sender)、スロットレシーバクラス(receiver)へのポインタ、接続されたsignalキーワードとslotキーワードのインデックスが格納されています。
シグナルがエミットされると、接続されているすべてのスロットが呼び出されなければなりません。これを実現するために、すべてのQObjectは、signalのそれぞれに対してConnectionインスタンスのリンクされたリストを持ち、slotキーワードのそれぞれに対しても同じConnectionのリンクされたリストを持っています。

このリンクされたリストのペアにより、Qt は各依存する slot/signal カップルを適切に捜査し、インデックスを使用して適切な関数をトリガーすることができます。同じ推論が receiver の破棄を処理するために使用されます。Qt はダブルリンクされたリストを捜査し、接続されていた場所からオブジェクトを削除します。

この処理は有名な UI スレッドで行われ、メッセージループ全体が処理され、接続されたすべてのシグナル/スロットが可能なイベント（マウス、キーボード、ネットワークなど）に応じてトリガーされます。QThread クラスは QObject を継承しているため、どの QThread でもシグナル/スロット機構を使用することができます。さらに、signals キーワードを他のスレッドにポストすることができ、受信側のスレッドのイベントループで処理されます。

***

## まとめ

本章では、クロスプラットフォームのSysInfoアプリケーションを作成しました。プラットフォーム固有のコードできちんとしたコード構成を持つために、シングルトンとストラテジーパターンを取り上げました。Qt Charts モジュールを使ってシステム情報をリアルタイムで表示することを学びました。最後に、Qt がシグナル/スロットメカニズムをどのように実装しているのか、また Qt 固有のキーワード (emit, signals, slots) の背後に何が隠されているのかを知るために、qmake コマンドを深く掘り下げてみました。

これまでに、Qt がどのように動作し、どのようにしてクロスプラットフォームのアプリケーションに取り組むことができるのか、明確なイメージを持っているはずです。次の章では、メンテナとしての正気を保つために、大きなプロジェクトを分割する方法を見ていきます。Qt の基本的なパターンである Model/View を学び、Qt でデータベースを使う方法を調査します。

***
**[戻る](../index.html)**
