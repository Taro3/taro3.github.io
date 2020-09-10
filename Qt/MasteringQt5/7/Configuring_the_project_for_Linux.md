# Linux用のプロジェクトを設定する

OpenCV のバイナリは，確かに公式のソフトウェアリポジトリで入手できます．ディストリビューションやパッケージマネージャにもよりますが、以下のようなコマンドでインストールすることができます。

    apt-get install libopencv
    yum install opencv

OpenCV が Linux にインストールされている場合，このスニペットを filter-plugindesigner.pro ファイルに追加することができます．

```QMake
linux {
target.path = $$(QTDIR)/../../Tools/QtCreator/lib/Qt/plugins/designer/

    CONFIG += link_pkgconfig
    PKGCONFIG += opencv
}
```

今回はLIBS変数ではなく、pkg-configに依存するpkgconfigを使用します。これは、コンパイル時のコマンドラインに正しいオプションを挿入してくれるヘルパーツールです。今回のケースでは、プロジェクトをOpenCVとリンクさせるためにpkg-configを要求します。

***

## Info

pkg-config が管理している全ての libs を pkg-config --list-all コマンドで一覧表示することができます。

***

**[戻る](../index.html)**
