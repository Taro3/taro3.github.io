# Qt6のQListの変更点について

Qt6 では、QList と QVector が統合され、に QVector がベースになったようです。

Qt6 の QList には、新たに *capacity()* と *squeeze()* という関数が追加されました。*capacity()* は現在のリストのアイテム確保サイズを返し、*squeeze()* は、実際にリスト内にセットされているアイテムの数に確保サイズを切り詰める関数です。

さらに、 QList への格納できるアイテムのメモリ制限の 2GiB がなくなりました。これに伴って、 QList::size() の戻り値などが int から *qsizetype* という型に変更されました。

```C++
QList<int> list;

list.reserve(100);              // あらかじめ100個分のサイズを確保
qDebug() << list.capacity();

list << 0 << 1 << 2;            // 3個のアイテムを追加
qsizetyp size = list.size();    // size()の戻り値はqsizetypeになった
qDebug() << size;

list.squeeze();                 // 100個分のサイズを実データサイズ(3個)に切り詰め
qDebug() << list.capacity();
```

これを実行すると下記のような出力になります。

```shell
100
3
3
```

この変更によって、アイテムを追加/削除した際にリストサイズを自動で調整する事によるオーバーヘッドを回避することができるようになりました。

***

**[戻る](../Qt.md)**
