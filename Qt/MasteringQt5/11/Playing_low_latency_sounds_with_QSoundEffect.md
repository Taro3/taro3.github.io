# QSoundEffectで低レイテンシーのサウンドを再生する

プロジェクトアプリケーション ch11-drum-machine は、kickWidget、snareWidget、hihatWidget、crashWidget の 4 つの SoundEffectWidget ウィジェットを表示します。

各 SoundEffectWidget ウィジェットは QLabel と QPushButton を表示します。ラベルにはサウンド名が表示されます。ボタンがクリックされると、サウンドが再生されます。

Qt Multimediaモジュールは、オーディオファイルを再生する主な方法を2つ提供しています。

* QMediaPlayer: このファイルは、様々な入力形式で曲や映画、インターネットラジオを再生することができます。
* QSoundEffect: このファイルは、低レイテンシーの.wavファイルを再生することができます。

このプロジェクトの例では、仮想ドラムマシンなので、QSoundEffectオブジェクトを使用しています。QSoundEffectを使用するための最初のステップは、次のように.proファイルを更新することです。

```QMake
QT += core gui multimedia
```

その後、サウンドを初期化します。ここに例を示します。

```C++
QUrl urlKick("qrc:/sounds/kick.wav");
QUrl urlBetterKick = QUrl::fromLocalFile("/home/better-kick.wav");

QSoundEffect soundEffect;
QSoundEffect.setSource(urlBetterKick);
```

最初のステップは、サウンドファイルに有効なQUrlを作成することです。urlKickは.qrcリソースのファイルパスから初期化し、urlBetterKickはローカルのファイルパスから作成します。そして、QSoundEffectを作成し、QSoundEffect::setSource()関数で再生するURLの音を設定します。

これでQSoundEffectオブジェクトが初期化されたので、以下のコードスニペットを使ってサウンドを再生することができます。

```C++
soundEffect.setVolume(1.0f);
soundEffect.play();
```

***

**[戻る](../index.html)**
