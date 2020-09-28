# QVariant を使ってオブジェクトをシリアル化する

ここでは、何をどのようにシリアル化するのかを考えてみましょう。ユーザーは、記録して再生するデータをすべて含んだTrackクラスと対話します。

ここから、シリアル化されるオブジェクトはTrackであると想定できます。これにより、SoundEventインスタンスのリストを含むmSoundEventsが何らかの形でもたらされるはずです。これを実現するために、QVariantクラスに大きく依存します。

QVariant を使ったことがあるかもしれません。これは、あらゆるプリミティブ型（char、int、doubleなど）だけでなく、複雑な型（QString、QDate、QPointなど）の汎用的なプレースホルダです。

***

## Info

QVariantがサポートしている型の完全なリストは、<http://doc.qt.io/qt-5/qmetatype.html#Type-enum> で入手できます。

***

QVariantの簡単な例を挙げます。

```C++
QVariant variant(21);

int answer = variant.toInt() * 2;

qDebug() << "what is the meaning of the universe,
 life and everything?"
 << answer;
```

21 を variant に格納します。ここから、 variant に値のコピーを希望の型にキャストしてもらうことができます。ここでは int 値が欲しいので、 variant.toInt() を呼び出します。variant.toX() 構文では、すでに多くの変換が可能です。

QVariantのカーテンの向こう側で何が起こっているのか、とても簡単に覗いてみることができます。QVariantはどのようにして、私たちが与えたすべての情報を保存しているのでしょうか？その答えはC++のunion型にあります。QVariantクラスは、一種のスーパーunionです。

unionは、一度に1つの非静的データメンバのみを保持できる特殊なクラス型です。短いスニペットでこれを説明します。

```C++
union Sound
{
    int duration;
    char code;
};

Sound s = 10;
qDebug() << "Sound duration:" << s.duration;
// output= Sound duration: 10

s.code = 'K';
qDebug() << "Sound code:" << s.code;
// output= Sound code: K
```

まず、unionクラスをstructのように宣言します。デフォルトでは、すべてのメンバはpublicとなっています。このunionの特異性は、メモリ内の最大のメンバサイズだけを取るということです。ここでは、Sound はメモリ内の int duration のスペース分だけを取ります。

unionはこの特定の空間だけを取るので、すべてのメンバ変数は同じメモリ空間を共有しています。したがって、未定義の動作をさせたい場合を除いて、一度に利用できるのは1つのメンバだけです。

Soundスニペットを使用する場合、値10で初期化することから始めます（デフォルトでは最初のメンバーが初期化されます）。ここから、s.durationにアクセスできますが、s.codeは未定義とみなされます。

s.codeに値を代入すると、s.durationは未定義になり、s.codeにアクセスできるようになります。

unionクラスを使うことで、メモリ使用量を非常に効率的にすることができます。QVariantでは、値を格納するときはプライベートのunionに格納されます。

```C++
union Data
{
    char c;
    uchar uc;
    short s;
    signed char sc;
    ushort us;
    ...
    qulonglong ull;
    QObject *o;
    void *ptr;
    PrivateShared *shared;
} data;
```

プリミティブ型のリストと、最後に複合型のQObject*とvoid*があることに注意してください。

Dataの他に、QMetaTypeオブジェクトが初期化され、格納されているオブジェクトの型を知ることができます。unionとQMetaTypeの組み合わせにより、QVariantはどのDataメンバを使用して値をキャストして呼び出し元に返すべきかを知ることができます。

これで、unionとは何か、そしてQVariantがどのようにそれを使うのかがわかったと思いますが、なぜQVariantクラスを作ったのかと疑問に思うかもしれません。単純なunionだけでは十分ではなかったのでしょうか？

答えはノーです。それはunionクラスがデフォルトのコンストラクタを持たないメンバを持つことができないからです。これはunionに入れるクラスの数を大幅に減らしてしまいます。Qtの人々は、unionにデフォルトのコンストラクタを持たないクラスをたくさん入れたいと考えていました。これを緩和するためにQVariantが誕生しました。

QVariantの面白いところは、カスタム型を格納できることです。SoundEventクラスをQVariantクラスに変換したい場合、SoundEvent.hに以下のように追加しています。

```C++
class SoundEvent
{
    ...
};
Q_DECLARE_METATYPE(SoundEvent);
```

Q_DECLARE_METATYPE マクロはすでに第 10 章「IPC は必要ですか？ミニオンを動作させる」で既に使用しています。このマクロは、SoundEventをQMetaTypeレジスタに効果的に登録し、QVariantで利用できるようにします。QDataStreamはQVariantに依存しているので、最後の章でこのマクロを使用しなければなりませんでした。

今度はQVariantで前後に変換します。

```C++
SoundEvent soundEvent(4365, 0);
QVariant stored;
stored.setValue(soundEvent);

SoundEvent newEvent = stored.value<SoundEvent>();
qDebug() << newEvent.timestamp;
```

お察しの通り、このスニペットの出力は、SoundEventに保存されているオリジナルのタイムスタンプである4365です。

このアプローチは、バイナリのシリアル化だけをしたい場合には完璧だったでしょう。データの書き込みや読み込みは簡単にできる。しかし、Track と SoundEvents を標準フォーマットに出力したい。JSON と XML です。

Q_DECLARE_METATYPE/QVariant コンボには大きな問題があります。これはシリアライズされたクラスのフィールドのためのキーを保存しません。SoundEvent クラスの JSON オブジェクトが次のようになることは、すでに予見できます。

```C++
{
    "timestamp": 4365,
    "soundId": 0
}
```

QVariantクラスがタイムスタンプ・キーが欲しいことを知る方法はありません。QVariantクラスは生のバイナリデータを保存するだけです。同じ原理がXMLにも適用されます。

そのため、QVariantのバリエーションとしてQVariantMapを使うことになります。QVariantMapクラスは、QMap\<QString, QVariant\>上のtypedefのみです。このマップは、QVariantクラスのフィールドのキー名と値を格納するために使用されます。そして、これらのキーはJSONやXMLのシリアライズシステムで使用され、きれいなファイルが出力されます。

柔軟なシリアライズシステムを目指しているので、このQVariantMapを複数の形式でシリアライズ/デシリアライズできるようにしなければなりません。これを実現するために、クラスがその内容をQVariantMapでシリアライズ/デシリアライズする機能を与えるインターフェースを定義します。

このQVariantMapは中間フォーマットとして使用され、最終的なJSON、XML、またはバイナリに依存しない。

Serializer.hという名前のC++ヘッダーを作成します。内容はこんな感じです。

```C++
#include <QVariant>

class Serializable {
public:
    virtual ~Serializable() {}
    virtual QVariant toVariant() const = 0;
    virtual void fromVariant(const QVariant& variant) = 0;
};
```

この抽象基底クラスを実装することで、クラスはSerializableになります。仮想純粋関数は2つしかありません。

* 関数 toVariant() は、クラスが QVariant (より正確には QMetaType システムのおかげで QVariant にキャストできる QVariantMap) を返さなければなりません。
* fromVariant() 関数は、パラメータとして渡された variant からクラスのメンバを初期化しなければなりません。

そうすることで、最終的なクラスにその内容をロードして保存する責任を与えます。結局のところ、SoundEvent 自体よりも優れた SoundEvent を知っているのは誰でしょうか？

SoundEventでSerializableの動作を見てみましょう。SoundEvent.hをこのように更新します。

```C++
#include "Serializable.h"

class SoundEvent : public Serializable
{
    SoundEvent(qint64 timestamp = 0, int soundId = 0);
    ~SoundEvent();

    QVariant toVariant() const override;
    void fromVariant(const QVariant& variant) override;

    ...
};
```

SoundEventクラスがSerializableになりました。実際の作業はSoundEvent.cppでやってみましょう。

```C++
QVariant SoundEvent::toVariant() const
{
    QVariantMap map;
    map.insert("timestamp", timestamp);
    map.insert("soundId", soundId);
    return map;
}

void SoundEvent::fromVariant(const QVariant& variant)
{
    QVariantMap map = variant.toMap();
    timestamp = map.value("timestamp").toLongLong();
    soundId = map.value("soundId").toInt();
}
```

toVariant()では、timestampとsoundIdで埋められるQVariantMapを宣言しています。

一方、fromVariant()では、バリアントをQVariantMapに変換し、その内容をtoVariant()で使用したのと同じキーで取得します。これはとてもシンプルなことです。

次に Serializable にしなければならないクラスは Track です。TrackをSerializableを継承させた後、Track.cppを更新します。

```C++
QVariant Track::toVariant() const
{
    QVariantMap map;
    map.insert("duration", mDuration);
    QVariantList list;
    for (const auto& soundEvent : mSoundEvents) {
        list.append(soundEvent->toVariant());
    }
    map.insert("soundEvents", list);

    return map;
}
```

少し複雑ですが、原理は同じです。mDuration変数は、SoundEventで見たようにマップオブジェクトに格納されます。mSoundEventsの場合は、各項目がSoundEventキーの変換されたQVariantバージョンであるQVariantのリスト（QVariantList）を生成しなければなりません。

これを行うには、単純にmSoundEventsをループさせ、listに先ほど説明したsoundEvent->toVariant()の結果を入力します。

次は fromVariant() です。

```C++
void Track::fromVariant(const QVariant& variant)
{
    QVariantMap map = variant.toMap();
    mDuration = map.value("duration").toLongLong();

    QVariantList list = map.value("soundEvents").toList();
    for(const QVariant& data : list) {
        auto soundEvent = make_unique<SoundEvent>();
        soundEvent->fromVariant(data);
        mSoundEvents.push_back(move(soundEvent));
    }
}
```

ここでは、キーとなるSoundEventsの各要素について、新しいSoundEventを作成し、dataの内容でロードし、最後にベクトルmSoundEventsに追加します。

***

**[戻る](../index.html)**
