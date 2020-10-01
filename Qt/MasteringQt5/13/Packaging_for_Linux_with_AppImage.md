# AppImageを使ったLinuxのパッケージング

Windows や Mac では、アプリケーションは自己完結型で、実行に必要なすべての依存関係を含んでいます。一方で、これはファイルの重複を増やし、開発者のためのパッケージングを簡素化します。

この前提に基づいて、Linux上で同じパターンを持つための努力がなされてきました(リポジトリ/ディストリビューションに特化したパッケージとは対照的です)。今日では、いくつかのソリューションがLinux上で自己完結型のパッケージを提供しています。これらのソリューションの一つを検討することをお勧めします。AppImageです。この特定のツールは、Linux コミュニティで人気を集めています。アプリケーションのパッケージ化とデプロイに AppImage を利用する開発者が増えています。

AppImageは、すべてのライブラリを含むアプリケーションを含むファイル形式です。AppImage ファイルをダウンロードして実行すると、アプリケーションが実行されます。裏では、AppImageはステロイドのISOファイルであり、実行するとその場でマウントされます。AppImageファイル自体は読み取り専用で、Firejail（アプリケーションの実行環境を制限することでセキュリティ侵害のリスクを軽減するSUIDサンドボックスプログラム）のようなサンドボックスでも動作します。

***

## Info

AppImage の詳細については、http://appimage.org/ をご覧ください。

***

gallery-desktopをAppImageにパッケージ化するには、大きく分けて2つのステップがあります。

1. gallery-desktopの依存関係をすべて集める。
2. gallery-desktopとその依存関係をAppImage形式でパッケージ化します。

幸いなことに、このプロセスは linuxdeployqt という便利なツールを使って行うことができます。これは趣味のプロジェクトとして始まったものですが、AppImageのドキュメントでQtアプリケーションをパッケージ化する公式な方法になりました。

***

## Info2

linuxdeployqtは、https://github.com/probonopd/linuxdeploy qt/から取得してください。

***

これから書くスクリプトは、linuxdeployqtのバイナリが$PATH変数にあることを前提としています。環境変数が正しく設定されていることを確認してください。

![image](img/3.png)

scripts/package-linux-appimage.shを作成し、このように更新します。

```shell
#!/bin/bash

DIST_DIR=dist/desktop-linux
BUILD_DIR=build

mkdir -p $DIST_DIR && cd $DIST_DIR
mkdir -p $BUILD_DIR

pushd $BUILD_DIR

$QTDIR/bin/qmake \
    -spec linux-g++ \
    "CONFIG += release" \
    ../../../ch13-gallery-packaging.pro
make qmake_all
pushd gallery-core && make ; popd
pushd gallery-desktop && make ; popd
popd

export QT_PLUGIN_PATH=$QTDIR/plugins/
export LD_LIBRARY_PATH=$QTDIR/lib:$(pwd)/build/gallery-core

linuxdeployqt \
    build/gallery-desktop/gallery-desktop \
    -appimage

mv build/gallery-desktop.AppImage .
```

第1部は、その集大成です。

1. メインパスの変数を設定します。出力ディレクトリはdist_dirです。全てのファイルは dist/desktop-linux/build フォルダに生成されます。
2. 全てのディレクトリを作成し、dist/desktop-linux/build に移動します。
3. Linuxプラットフォームのリリースモードでqmakeを実行して、親プロジェクトのMakefileを生成します。
4. make qmake_all コマンドを実行して、サブプロジェクトの Makefiles を生成します。
5. 必要な各サブプロジェクトを構築するために make コマンドを実行します。

スクリプトの2番目の部分は、linuxdeployqtに関するものです。まず、linuxdeployqtがgallery-desktopのすべての依存関係（Qtライブラリとgallery-coreライブラリ）を適切に見つけるために、いくつかのパスをエクスポートしなければなりません。

その後、作業するソースバイナリとターゲットファイルタイプ（AppImage）を指定してlinuxdeployqtを実行します。結果として、gallery-desktop.AppImageという単一のファイルが作成され、Qtパッケージがインストールされていなくても、ユーザーのコンピュータ上で起動できるようになります！

***

**[戻る](../index.html)**
