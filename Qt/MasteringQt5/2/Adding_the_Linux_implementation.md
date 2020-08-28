# Linuxの実装を追加する

Linuxのch02-sysinfoプロジェクトを実装してみましょう。Windowsの実装をすでに完了している場合は簡単です。SysInfo実装クラスの作成方法、キーボードショートカット、SysInfoインターフェースの詳細など、一部の情報のヒントはこの部分では繰り返しません。

SysInfoクラスから継承するSysInfoLinuxImplという新しいC++クラスを作成し、基底クラスから仮想関数を挿入します。

```C++
#include "sysinfo.h"

class SysInfoLinuxImpl : public SysInfo
{
public:
    SysInfoLinuxImpl();
    
    // SysInfo interface
public:
    void init();
    double cpuLoadAverage();
    double memoryUsed();
};
```

まず、SysInfoLinuxImpl.cppにmemoryYsed()関数を実装します。

```C++
double SysInfoLinuxImpl::memoryUsed()
{
    struct sysinfo memInfo;
    sysinfo(&memInfo);
    
    qulonglong totalMemoryUsed = memInfo.totalram - memInfo.freeram;
    totalMemoryUsed += memInfo.totalswap - memInfo.freeswap;
    totalMemoryUsed += memInfo.mem_unit;
    
    double percecnt = (double) totalMemoryUsed /
            (double)totalMemoryUsed * 100.0;
    return qBound(0.0, percecnt, 100.0);
}
```

この関数はLinux固有のAPIを使用します。必要なインクルードを追加した後、システム全体の統計に関する情報を返すLinuxのsysinfo()関数を使用できます。合計メモリと合計使用メモリで、簡単にパーセント値を返すことが出来ます。スワップメモリが考慮されていることに注意してください。

CPUロード機能は、メモリ機能よりも少し複雑です。実際には、CPUが様々な種類の作業の実行に費やした合計時間をLinuxから取得します。しかし、これは私たちが望んでいることとは全く違います。瞬間的なCPU負荷を返す必要があります。これを得る一般的な方法は、2つのサンプル値を短時間で取得し、その差を使用して瞬間的なCPU負荷を取得することです。

```C++
#include <QtGlobal>
#include <QVector>

#include "sysinfo.h"

class SysInfoLinuxImpl : public SysInfo
{
public:
    SysInfoLinuxImpl();

    // SysInfo interface
public:
    void init();
    double cpuLoadAverage();
    double memoryUsed();
    
private:
    QVector<qulonglong> cpuRawData();
    
private:
    QVector<qulonglong> mCpuLoadLastValues;
};
```

この実装では、ヘルパ関数とメンバ変数を1つだけ追加します。

* cpuRawData()は、Linux API呼び出しを実行してシステムのタイミング情報を取得し、qulonglong型のQVectorクラスの値を返す関数です。ユーザモードの通常プロセス、ユーザモードのniceプロセス、カーネルモードのプロセス、アイドルの4つにCPUが費やした時間を含む値を取得して返します。
* mCpuLoadLastValuesは、特定の瞬間のシステムタイミングのサンプルを格納する変数です。

SysInfoLinuxImpl.cppファイルに移動して更新します。

```C++
#include "sysinfolinuximpl.h"

#include <sys/types.h>
#include <sys/sysinfo.h>

#include <QFile>

SysInfoLinuxImpl::SysInfoLinuxImpl() :
    SysInfo(),
    mCpuLoadLastValues()
{
}

void SysInfoLinuxImpl::init()
{
    mCpuLoadLastValues = cpuRawData();
}
```

前述のように、cpuLoadAverage関数は、瞬時のCPU負荷平均を計算できるように2つのサンプルを必要とします。init()関数を呼び出すと、mCpuLoadLastValuesを初めて設定できます。

```C++
QVector<qulonglong> SysInfoLinuxImpl::cpuRawData()
{
    QFile file("/proc/stat");
    file.open(QIODevice::ReadOnly);
    
    QByteArray line = file.readLine();
    file.close();
    qulonglong totalUser = 0, totalUserNice = 0, totalSystem = 0, totalIdle = 0;
    
    std::sscanf(line.data(), "cpu %llu %llu %llu %llu",
                &totalUser, &totalUserNice, &totalSystem, &totalIdle);
    
    QVector<qulonglong> rawData;
    rawData.append(totalUser);
    rawData.append(totalUserNice);
    rawData.append(totalSystem);
    rawData.append(totalIdle);
    
    return rawData;
}
```

LinuxシステムでCPUの未加工情報を取得するために、/proc/statファイルで利用可能な情報を解析することを選択しました。必要なのは最初の行でだけなので、単一のreadLine()で十分です。Qtはいくつかの便利な機能を提供しますが、C標準ライブラリ関数のほうが簡単な場合があります。それはこの場合に当てはまります。std::sscanfを使用して、文字列から変数を抽出しています。次に、cpuLoadAverage()本体を見てみましょう。

```C++
double SysInfoLinuxImpl::cpuLoadAverage()
{
    QVector<qulonglong> firstSample = mCpuLoadLastValues;
    QVector<qulonglong> secondSample = cpuRawData();
    mCpuLoadLastValues = secondSample;
    
    double overall = (secondSample[0] - firstSample[0])
            + (secondSample[1] - firstSample[1])
            + (secondSample[2] - firstSample[2]);
    
    double total = overall + (secondSample[3] - firstSample[3]);
    double percent = (overall / total) * 100.0;
    return qBound(0.0, percent, 100.0);
}
```

ここでマジックが起こります。この最後の関数では、すべてのパズルのピースを組み合わせます。
この関数は、CPUのローデータの2つのサンプルを使用します。最初のサンプルは、メンバー変数mCpuLoadLastValuesから取得され、init()関数によって初めて設定されます。2番目のサンプルは、cpuLoadAverage()関数によって要求されます。その後、mCpuLoadLastValues変数は、次のcpuLoadAverage()関数呼び出しの最初のサンプルとして使用される新しいサンプルを格納します。

パーセント式は理解しやすいはずです。

* ovarallは、user + nice + kernelに等しい
* totalは、overall + idleに等しい

## Tip

/proc/statの詳細については、https://www.kernel.org/doc/Documentation/filesystems/proc.txt にあるLinuxカーネルのドキュメントを参照してください。

他の実装と同様に、最後に行うことは、ch02-sysinfo.proファイルを次のように編集することです。

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

linux {
    SOURCES += sysinfolinuximpl.cpp
    HEADERS += sysinfolinuximpl.h
}

FORMS += \
    mainwindow.ui
```

このLinuxスコープ条件がch02-sysinfo.proファイルにあるため、Linux固有のファイルは、他のプラットフォームではqmakeコマンドによって処理されません。

***
**[戻る](../index.html)**
