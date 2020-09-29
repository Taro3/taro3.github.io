# バイナリ形式でのオブジェクトのシリアライズ

XMLシリアライゼーションは完全に機能しています。ここで、この章で取り上げた最後のタイプのシリアライズに切り替えましょう。

バイナリシリアライズはQtが直接行う方法を提供しているので簡単です。Serializerを継承したBinarySerializerクラスを作ってください。ヘッダは共通で、オーバーライド関数はsave()とload()だけです。save()関数の実装です。

```C++
void BinarySerializer::save(const Serializable& serializable,
    const QString& filepath, const QString& /*rootName*/)
{
    QFile file(filepath);
    file.open(QFile::WriteOnly);
    QDataStream dataStream(&file);
    dataStream << serializable.toVariant();
    file.close();
}
```

第10章「IPCが必要ですか？」で使用されているQDataStreamクラスを認識していただけたでしょうか。ミニオンを働かせましょう。今回はこのクラスを使用して、バイナリデータを宛先のQFileにシリアライズします。QDataStreamクラスは、<<演算子でQVariantクラスを受け入れます。バイナリ・シリアライザではrootName変数が使用されていないことに注意してください。

ここではload()関数を使用しています。

```C++
void BinarySerializer::load(Serializable& serializable, const QString&
    filepath)
{
    QFile file(filepath);
    file.open(QFile::ReadOnly);
    QDataStream dataStream(&file);
    QVariant variant;
    dataStream >> variant;
    serializable.fromVariant(variant);
    file.close();
}
```

QVariantとQDataStreamメカニズムのおかげで、タスクは簡単です。ソースファイルパスでQFileを開きます。このQFileからQDatastreamクラスを構築します。次に、ルートのQVariantを読み込むために >> 演算子を使用します。最後に、Serializable::fromVariant()関数でソースSerializableを埋めます。

BinarySerializerクラスでシリアル化されたTrackクラスの例は載せませんのでご安心ください。

シリアライズ部分が完成しました。このサンプルプロジェクトの GUI 部分は、この本のこれまでの章で何度も取り上げてきました。以下のセクションでは、MainWindow と SoundEffectWidget クラスで使用されている特定の機能についてのみ説明します。完全な C++ クラスが必要な場合は、ソースコードを確認してください。

***

**[戻る](../index.html)**
