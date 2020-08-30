# データクラスの定義

私たちはギャラリーを一から構築します。データベース層を適切に記述できるように、データクラスの実装から始めます。このアプリケーションは、写真をアルバムに整理することを目的としています。したがって、2つの明白なクラスは、AlbumとPictureです。この例では、アルバムは単に名前を持っています。Pictureクラスは、Albumクラスに属していなければならず、ファイルパス（元のファイルがあるファイルシステム上のパス）を持っていなければなりません。

Albumクラスは、プロジェクトの作成時にすでに作成されています。 Album.hファイルを開き、次の実装が含まれるように更新します。

```C++
#include <QString>

#include "gallery-core_global.h"

class GALLERYCORE_EXPORT Album
{
public:
    explicit Album(const QString& name = "");

    int id() const;
    void setId(int id);
    QString name() const;
    void setName(const QString& name);

private:
    int mId;
    QString mName;
};
```

見た通り、AlbumクラスにはmId変数（データベースID）とmName変数のみが含まれています。典型的なOOP（オブジェクト指向パラダイム）の方法では、AlbumクラスにはQVector \<Picture\> mPicturesフィールドがあるはずです。わざとやったわけではありません。これら2つのオブジェクトを分離することで、関連する何千もの画像を引き出さずにアルバムをロードしたい場合に、より柔軟になります。AlbumクラスにmPicturesを設定する際のもう1つの問題は、このコードを使用する開発者（あなたまたは他の誰か）が自問することです。mPicturesはいつ読み込まれるのか？Albumの部分的なロードを実行して、不完全なアルバムを作成するべきか、それとも常にすべての画像を含むAlbumをロードするべきか？

フィールドを完全に削除することで、問題は存在しなくなり、コードはよりシンプルに把握できるようになります。開発者は、画像が欲しい場合は明示的に画像をロードしなければならないことを直感的に知っています; そうでなければ、このシンプルな Album クラスを続けることができます。

ゲッタとセッタは十分に明解です。私たちは、見なくても実装できます。Album.cpp の Album クラスのコンストラクタだけを見てみましょう。

```C++
Album::Album(const QString &name) :
    mId(-1),
    mName(name)
{
}
```

デフォルトでは無効なidが使用されていることを確認するためにmId変数を-1に初期化し、mName変数にはnameの値を代入しています。

これで、Pictureクラスに進むことができます。Pictureという名前のC++クラスを新規作成し、Picture.hを開いて以下のように修正します。

```C++
#include <QUrl>
#include <QString>

#include "gallery-core_global.h"

class GALLERYCORE_EXPORT Picture
{
public:
    Picture(const QString& filePath = "");
    Picture(const QUrl& fileUrl);
    
    int id() const;
    void setId(int id);
    
    int albumId() const;
    void setAlbumId(int albumId);
    QUrl fileUrl() const;
    void setFileUrl(const QUrl& fileUrl);
    
private:
    int mId;
    int mAlbumId;
    QUrl mFileUrl;
};
```

ライブラリからクラスをエクスポートするには、classキーワードの直前にGALLERYCORE_EXPORTマクロを追加することを忘れないでください。データ構造体として、PictureはmId変数を持ち、mAlbumId変数に属し、mFileUrl値を持ちます。プラットフォーム（デスクトップかモバイルか）に応じてパス操作がしやすくなるように、QUrl型を使用しています。

Picture.cppを見てみましょう。

```C++
#include "picture.h"

Picture::Picture(const QString &filePath) :
    Picture(QUrl::fromLocalFile(filePath))
{
}

Picture::Picture(const QUrl &fileUrl) :
    mId(-1),
    mAlbumId(-1),
    mFileUrl(fileUrl)
{
}

QUrl Picture::fileUrl() const
{
    return mFileUrl;
}

void Picture::setFileUrl(const QUrl &fileUrl)
{
    mFileUrl = fileUrl;
}
```

最初のコンストラクタでは、静的関数 QUrl::fromLocalFile が呼び出され、QUrl パラメータを取る他のコンストラクタに QUrl オブジェクトを提供します。

他のコンストラクタを呼び出す機能は、C++11では素晴らしい追加機能です。

***
**[戻る](../index.html)**
