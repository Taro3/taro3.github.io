# Windows用パッケージ

Windows上のスタンドアロンアプリケーションをパッケージ化するには、実行ファイルのすべての依存関係を提供する必要があります。gallery-core.dllファイル、Qtライブラリ(例: Qt5Core.dll)、コンパイラ固有のライブラリ(例: libstdc++-6.dll)は、実行ファイルが必要とする依存関係の例です。ライブラリの提供を忘れると、gallery-desktop.exeプログラムの実行時にエラーが表示されます。

***

## Info

Windows では、Dependency Walker (depends) というユーティリティを使うことができます。これは、あなたのアプリケーションが必要とするすべてのライブラリのリストを表示します。こちらからダウンロードできます: www.dependencywalker.com。

***

このセクションでは、コマンドラインインターフェースを介してプロジェクトをビルドするスクリプトを作成します。次に、Qtツールwindeployqtを使用して、アプリケーションに必要なすべての依存関係を収集します。この例は MinGW コンパイラのものですが、MSVC コンパイラにも簡単に適応できます。

Windows上でgallery-desktopを適切に実行するために、winqtdeployによって収集された必要なファイルとフォルダのリストは以下の通りです。

* iconengines:
  * qsvgicon.dll
* imageformats:
  * qjpeg.dll
  * qwbmp.dll
  * ...
* Platforms:
  * qwindows.dll
* translations:
  * qt_en.qm
  * qt_fr.qm
  * ...
* D3Dcompiler_47.dll
* gallery-core.dll
* gallery-desktop.exe
* libEGL.dll
* libgcc_s_dw2-1.dll
* libGLESV2.dll
* libstdc++-6.dll
* libwinpthread-1.dll
* opengl32sw.dll
* Qt5Core.dll
* Qt5Gui.dll
* Qt5Svg.dll
* Qt5Widgets.dll

環境変数が正しく設定されていることを確認してください。

![image](img/1.png)

scripts ディレクトリに package-windows.bat というファイルを作成します。

```bat
@ECHO off
set DIST_DIR=dist\desktop-windows
set BUILD_DIR=build
set OUT_DIR=gallery
mkdir %DIST_DIR% && pushd %DIST_DIR%
mkdir %BUILD_DIR% %OUT_DIR%
pushd %BUILD_DIR%
%QTDIR%\bin\qmake.exe ^
 -spec win32-g++ ^
 "CONFIG += release" ^
 ..\..\..\ch13-gallery-packaging.pro
%MINGWROOT%\bin\mingw32-make.exe qmake_all
pushd gallery-core
%MINGWROOT%\bin\mingw32-make.exe && popd
pushd gallery-desktop
%MINGWROOT%\bin\mingw32-make.exe && popd
popd
copy %BUILD_DIR%\gallery-core\release\gallery-core.dll %OUT_DIR%
copy %BUILD_DIR%\gallery-desktop\release\gallery-desktop.exe %OUT_DIR%
%QTDIR%\bin\windeployqt %OUT_DIR%\gallery-desktop.exe %OUT_DIR%\gallerycore.dll
popd
```

実行されたステップについてお話しましょう。

1. メインパスの変数を設定します。出力ディレクトリはdist_dirです。すべてのファイルは dist/desktop-windows/build ディレクトリに生成されます。
2. すべてのディレクトリを作成し、dist/desktop-windows/buildを起動します。
3. Win32プラットフォーム用のリリースモードでqmakeを実行し、親プロジェクトのMakefileを生成します。win32-g++仕様はMinGWコンパイラ用です。MSVCコンパイラを使用する場合はwin32-msvc仕様を使用してください。
4. mingw32-make qmake_all コマンドを実行して、サブプロジェクトの Makefiles を生成します。MSVC コンパイラでは、mingw32-make を nmake または jom に置き換える必要があります。
5. 必要な各サブプロジェクトを構築するために mingw32-make コマンドを実行します。
6. 生成されたファイル、gallery-desktop.exeとgallery-core.dllをgalleryディレクトリにコピーします。
7. 両方のファイルでQtツールwindeployqtを呼び出し、必要な依存関係（例えば、Qt5Core.dll、Qt5Sql.dll、libstdc++-6.dll、qwindows.dllなど）をすべてコピーします。

***

**[戻る](../index.html)**
