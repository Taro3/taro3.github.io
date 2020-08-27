# Windowsの実装を追加する

この章の最初の方にあったUML図を覚えていますか？SysInfoWindowsImplクラスは、SysInfoクラスから派生したクラスの1つです。このクラスの主な目的は、CPUとメモリ使用量を取得するためのWindows固有のコードをカプセル化することです。

SysInfoWindowsImplクラスを作成するときが来ました。そのためには、以下の手順を実行する必要があります。

1. 階層ビューのch02-sysinfoプロジェクト名を右クリックします。
2. **Add New**|**C++クラス**|**選択**をクリックします。
3. クラス名のフィールドをSysInfoWindowsImplに設定します。
4. 基底クラスフィールドを<カスタム>に設定し、下のフィールドにSysInfoを入力します。
5. **次へ**をクリックしてから**完了**をクリックして、空のC++クラスを生成します。

作成されたファイルは良い出発点ですが、調整が必要です。

```C++
#include "sysinfo.h"

class SysInfoWindowsImpl : public SysInfo
{
public:
    SysInfoWindowsImpl();
    void init() override;
    double cpuLoadAverage() override;
    double memoryUsed() override;
};
```

最初にすべきことは、親クラスであるSysInfoをincludeディレクティブを追加することです。これで基底クラスで定義されている仮想関数をオーバーライドできるようになりました。

## Qt tip

派生クラス名(classの後ろのキーワード)にカーソルを置き、Alt + Enterキー(Windows/Linux)またはCommand + Enterキー(Mac)を押すと、ベースクラスの仮想関数が自動的に挿入されます。

キーワードoverrideはC++11から使用できます。これにより、関数が基底クラスでvirtualとして宣言されていることが保証されます。overrideとしてマークされた関数シグネチャが、親クラスのvirtual関数と一致しない場合は、コンパイル時にエラーが表示されます。

Windowsで現在使用されているメモリを取得するのは簡単です。まずはSysInfoWindowsImpl.cppファイルのこの機能から始めます。

```C++
#include "sysinfowindowsimpl.h"

#include <windows.h>

SysInfoWindowsImpl::SysInfoWindowsImpl() :
    SysInfo()
{

}

double SysInfoWindowsImpl::memoryUsed()
{
    MEMORYSTATUSEX memoryStatus;
    memoryStatus.dwLength = sizeof(MEMORYSTATUSEX);
    GlobalMemoryStatusEx(&memoryStatus);
    qulonglong memoryPhysicalUsed = memoryStatus.ullTotalPhys - memoryStatus.ullAvailPhys;
    return (double)memoryPhysicalUsed / (double)memoryStatus.ullTotalPhys * 100.0;
}
```

Windows APIを使えるようにwindows.hファイルをインクルードするのを忘れずに！この関数は合計と使用可能な物理メモリを取得しています。単純な減算で使用されているメモリの量がわかります。基底クラスSysInfoで宣言されたように、この実装ではdouble型で値を返します。例えば、Windows OSで23%のメモリを使用している場合は23.0という値を返します。

総メモリ使用量を取得することは良い開始点ですが、ここで終わるわけには生きません。私たちのクラスはCPU負荷も取得しなければなりません。WindowsのAPIは時々厄介なことがあります。コードをより読みやすくするために、2つのプライベートヘルパ関数を作成します。SysInfoWindowsImpl.hファイルを以下のように更新します。

```C++
#include <QtGlobal>
#include <QVector>

#include "sysinfo.h"

typedef struct _FILETIME FILETIME;

class SysInfoWindowsImpl : public SysInfo
{
public:
    SysInfoWindowsImpl();

    // SysInfo interface
public:
    void init() override;
    double cpuLoadAverage() override;
    double memoryUsed() override;

private:
    QVector<qulonglong> cpuRawData();
    qulonglong convertFileTime(const FILETIME& filetime) const;

private:
    QVector<qulonglong> mCpuLoadLastValues;
};
```

上記の変更点を分析してみましょう。

* cpuRawData()は、Windows API呼び出しを実行して、システムのタイミング情報を取得し、一般的な形式で値を返す関数です。システムがアイドル、カーネル、ユーザーモードで消費した時間の3つを返します。
* convertFileTIme()関数は、2番目のヘルパです。これはWindowsのFILETIME構造体構文をqulonglong型に変換します。qulonglong型はQtのunsigned long long intです。この型はQtによって、すべてのプラットフォームで64ビットであることが保証されています。また、型定義されたquint64を使用することもできます。
* mCpuLoadLastValuesは、任意の瞬間のシステムタイミング(アイドル、カーネル、ユーザ)を格納する変数です。
* qulonglong型を使用するには\<QtGlobal\>タグを、QVectorクラスを使用するには\<QVector\>タグを忘れずにインクルードしてください。
* typedef struct _FILETIME FILETIMEは、FILETIME構文の前方宣言のようなものです。参照のみを使用しているため、SysInfoWindowsInpl.hファイルの中に\<windows.h\>タグを含めず、CPPファイルの中に入れておくことができます。

あとはファイルSysInfoWindowsImpl.cppに切り替えて、これらの関数を実装することで、WindowsのCPU負荷平均機能を終了させることができます。

```C++
#include "sysinfowindowsimpl.h"

#include <windows.h>

SysInfoWindowsImpl::SysInfoWindowsImpl() :
    SysInfo(),
    mCpuLoadLastValues()
{

}

void SysInfoWindowsImpl::init()
{
    mCpuLoadLastValues = cpuRawData();
}
```

init()関数が呼ばれたときに、cpuRawData()関数の戻り地をクラス変数mCouLoadLastValuesに格納しています。これは、cpuLoadAverage()関数の処理に役立ちます。

なぜコンストラクタの初期化リストでこのタスクを実行しないのか不思議に思うかもしれません。それは、初期化リストから関数を呼び出すと、オブジェクトがまだ完全に構築されていないからです！状況によっては、まだ構築されていないメンバ変数に関数がアクセスしようとする可能性があるので、安全ではないかもしれません。しかし、このch02-sysinfoプロジェクトでは、cpuRawData関数はメンバ変数を一切使用していないので、どうしてもやりたい場合は一応安全です。cpuRawData()関数をSysInfoWindowsImpl.cppファイルに追加します。

```C++
QVector<qulonglong> SysInfoWindowsImpl::cpuRawData()
{
    FILETIME idleTIme;
    FILETIME kernelTime;
    FILETIME userTime;

    GetSystemTimes(&idleTIme, &kernelTime, &userTime);

    QVector<qulonglong> rawData;
    rawData.append(convertFileTime(idleTIme));
    rawData.append(convertFileTime(kernelTime));
    rawData.append(convertFileTime(userTime));
    return rawData;
}
```

これが、GetSystemTimes関数へのWindows API呼び出しです！この関数は、システムがアイドルモード、カーネルモード、ユーザモードで費やした時間を教えてくれます。QVectorクラスに入力する前に、次のコードで説明されているヘルパconvertFileTimeを使用して各値を変換します。

```C++
qulonglong SysInfoWindowsImpl::convertFileTime(const FILETIME &filetime) const
{
    ULARGE_INTEGER largeInteger;
    largeInteger.LowPart = filetime.dwLowDateTime;
    largeInteger.HighPart = filetime.dwHighDateTime;
    return largeInteger.QuadPart;
}
```

WindowsのFILETIME構造体は、64ビットの情報を2つの32ビット部分(ローとハイ)に格納します。関数convertFileTimeは、WindowsのULARGE_INTEGERを使用して、qulonglong型として返す前に、64ビット値を正しく構築します。最後になりましたが、以下cpuLoadAverage()の実装です。

```C++
double SysInfoWindowsImpl::cpuLoadAverage()
{
    QVector<qulonglong> firstSample = mCpuLoadLastValues;
    QVector<qulonglong> secondSample = cpuRawData();
    mCpuLoadLastValues = secondSample;

    qulonglong currentIdle = secondSample[0] - firstSample[0];
    qulonglong currentKernel = secondSample[1] - firstSample[1];
    qulonglong currentUser = secondSample[2] - firstSample[2];
    qulonglong currentSystem = currentKernel + currentUser;

    double percent = (currentSystem - currentIdle) * 100.0 /
            currentSystem;
    return qBound(0.0, percent, 100.0);
}
```

ここで注意すべき点が3つあります。

* サンプルは絶対時間であるため、2つの異なるサンプルを差し引くと、現在のCPU負荷を取得するために処理できる瞬間的な値が得られます。
* 最初のサンプルは、メンバー変数mCpuLoadLastValuesからのもので、init()関数によって初めて設定されます。2つ目は、cpuLoadAverage()関数が呼び出されたときに取得されます。サンプルを初期化した後、mCpuLoadLastValues変数は、次の呼び出しで使用される新しいサンプルを格納できます。
* WindowsAPIから取得したカーネル値にはアイドル値も含まれるため、percentの式は少し厄介です。

## Tip

Windows APIの詳細については、https://msdn.microsoft.com/library にあるMSDNのドキュメントをご覧ください。

Windowsの実装を完了する最後の手順は、ファイルch02-sysinfo.proを編集して、次のようにすることです。

```QMake
QT       += core gui

greaterThan(QT_MAJOR_VERSION, 4): QT += widgets

CONFIG += C++14

SOURCES += \
    main.cpp \
    mainwindow.cpp \
    sysinfo.cpp

HEADERS += \
    mainwindow.h \
    sysinfo.h

windows {
SOURCES += sysinfowindowsimpl.cpp
HEADERS += sysinfowindowsimpl.h
}

FORMS += \
    mainwindow.ui
```

todoアプリケーションで行ったように、ch02-sysinfoプロジェクトでもC++14を使用します。ここでの新しい点は、共通のSOURCESとHEADERS変数からSysInfoWindowsImpl.cppとSysInfoWindowsImpl.hファイルを削除したことです。それらは、Windowsプラットフォームのスコープに移動しました。他のプラットフォーム用にビルドする場合、それらのファイルはqmakeによって処理されません。そのため、他のプラットフォームでのコンパイルに影響を与えることなく、windows.hなどの特定のヘッダをソースファイルSysInfoWindows.cppに安全に含めることができます。

***
**[戻る](../index.html)**
