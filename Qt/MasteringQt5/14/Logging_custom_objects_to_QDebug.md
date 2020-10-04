# カスタムオブジェクトをQDebugにロギングする

複雑なオブジェクトをデバッグしているときに、現在のメンバーの値を qDebug() に出力するといいでしょう。他の言語 (Java など) では、toString() メソッドまたはそれに相当するメソッドに遭遇したことがあるかもしれませんが、これは非常に便利です。

確かに、以下のような構文でコードを書くために、ログを記録したい各オブジェクトに関数 void toString() を追加することができます。

```C++
qDebug() << "Object content:" << myObject.toString()
```

C++にはもっと自然な方法があるはずです。さらに、Qtはすでにこの種の機能を提供しています。

```C++
QDate today = QDate::currentDate();
qDebug() << today;
// Output: QDate("2016-10-03")
```

これを実現するには、C ++演算子のオーバーロードに依存します。これは、「第10章、IPCが必要ですか？ミニオンを機能させる」でQDataStreamオペレーターを使用して行ったものと非常によく似ています。

struct Personを考えてみましょう。

```C++
struct Person {
    QString name;
    int age;
};
```

QDebugに適切に出力する機能を追加するには、次のようにQDebugとPersonの間の<<演算子をオーバーライドするだけです。

```C++
#include <QDebug>

struct Person {
    ...
};

QDebug operator<<(QDebug debug, const Person& person)
{
    QDebugStateSaver saver(debug);
    debug.nospace() << "("
                    << "name: " << person.name << ", "
                    << "age: " << person.age
                    << ")";
    return debug;
}
```

QDebugStateSaverは、QDebugの設定を保存し、破壊時に自動的に復元する便利なクラスです。<<演算子のオーバーロードでQDebugを壊さないように常に使用するのが良いでしょう。

あとはQDebugを使って、最後に修正したdebug変数を返すといういつものやり方です。これでPersonはこんな感じで使えるようになりました。

```C++
Person person = { "Lenna", 64 };
qDebug() << "Person info" << person;
```

toString() 関数は必要ありません。単に person オブジェクトを使用してください。疑問に思った人のために、はい、レンナはwrrting（2016年）の時点で本当に64です。

***

**[戻る](../index.html)**
