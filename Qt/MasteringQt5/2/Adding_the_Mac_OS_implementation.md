# Mac OS実装を追加する

SysInfoクラスのMac実装を見てみましょう。まず、SysInfoクラスを継承するSysInfoMacImplという名前の新しいC++クラスを作成します。SysInfo仮想関数をオーバーライドすると次のようなSysInfoMacImpl.hファイルが作成されます。

```C++
#include "sysinfo.h"

#include <QtGlobal>
#include <QVector>

class SysInfoMacImpl : public SysInfo
{
public:
    SysInfoMacImpl();

    // SysInfo interface
public:
    void init() override;
    double cpuLoadAverage() override;
    double memoryUsed() override;
};
```

最初に行う実装は、SysInfoMacImpl.cppファイル内のmemoryUsed()関数です。

```C++
#include <mach/vm_statistics.h>
#include <mach/mach_types.h>
#include <mach/mach_init.h>
#include <mach/mach_host.h>
#include <mach/vm_map.h>

SysInfoMacImpl::SysInfoMacImpl() :
    SysInfo()
{

}

double SysInfoMacImpl::memoryUsed()
{
    vm_size_t pageSize;
    vm_statistics64_data_t vmStats;

    mach_port_t machPort = mach_host_self();

    mach_msg_type_number_t count = sizeof(vmStats)
            / sizeof(natural_t);
    host_page_size(machPort, &pageSize);

    host_statistics64(machPort,
                      HOST_VM_INFO,
                      (host_info64_t)&vmStats,
                      &count);

    qulonglong freeMemory = (int64_t)vmStats.free_count
            * (int64_t)pageSize;

    qulonglong totalMemoryUsed = ((int64_t)vmStats.active_count +
                                  (int64_t)vmStats.inactive_count +
                                  (int64_t)vmStats.wire_count)
            * (int64_t)pageSize;

    qulonglong totalMemory = freeMemory + totalMemoryUsed;

    double percent = (double)totalMemoryUsed
            / (double)totalMemory * 100.0;
    return qBound(0.0, percent, 100.0);
}
```

まず、Mac OSカーネルのさまざまなヘッダを含めます。次に、mach_host_self()関数を呼び出してmachPortを初期化します。machPortは、システムへの情報を要求できるようにする、カーネルへの一種の特別な接続のようなものです。次に、host_statistics64()で仮想メモリ統計を取得できるように、他の変数の準備に進みます。

vmStatsクラスに必要な情報が入力されると、関連するデータであるfreeMemoryとtotalMemoryUsedが抽出されます。

Mac OSには、メモリを管理する独特の方法があることに注意してください。これは、大量のメモリをキャッシュに保持し、必要なときにフラッシュできるようにします。これは、私たちの統計が誤解される可能性があることを意味します。メモリが使用されたように見えますが、単に「念のため」に保管されているだけなのです。

割合の計算は簡単ですが、今後のグラフでおかしな値が出ないようにするために、クランプされた最小/最大値を返します。

次はcpuLoadAverage()の実装です、パターンは同じで、定期的な感覚でサンプルを取得し、この感覚での増加を計算します。したがって、次のサンプルとの差を計算できるように、中間値を保存する必要があります。

```C++
#include "sysinfo.h"

#include <QtGlobal>
#include <QVector>

...

private:
    QVector<qulonglong> cpuRawData();

private:
    QVector<qulonglong> mCpuLoadLastValues;
};

void SysInfoMacImpl::init()
{
    mCpuLoadLastValues = cpuRawData();
}

QVector<qulonglong> SysInfoMacImpl::cpuRawData()
{
    host_cpu_load_info_data_t cpuInfo;
    mach_msg_type_number_t cpuCount = HOST_CPU_LOAD_INFO_COUNT;
    QVector<qulonglong> rawData;
    qulonglong totalUser = 0, totalUserNice = 0, totalSystem = 0,
            totalIdle = 0;
    host_statistics(mach_host_self(),
                    HOST_CPU_LOAD_INFO,
                    (host_info_t)&cpuInfo,
                    &cpuCount);

    for(unsigned int i = 0; i < cpuCount; i++) {
        unsigned int maxTicks = CPU_STATE_MAX * i;
        totalUser += cpuInfo.cpu_ticks[maxTicks + CPU_STATE_USER];
        totalUserNice += cpuInfo.cpu_ticks[maxTicks
                + CPU_STATE_SYSTEM];
        totalSystem += cpuInfo.cpu_ticks[maxTicks
                + CPU_STATE_NICE];
        totalIdle += cpuInfo.cpu_ticks[maxTicks + CPU_STATE_IDLE];
    }

    rawData.append(totalUser);
    rawData.append(totalUserNice);
    rawData.append(totalSystem);
    rawData.append(totalIdle);
    return rawData;
}
```

見たとおり、使用されるパターンはLinuxの実装と厳密に同等です。SysInfoLinuxImpl.cppファイルからcpuLoadAverage()関数の本体をコピーして貼り付けることも出来ます。それらは全く同じことを行います。

ここでは、cpuRawData()関数の実装は異なります。host_statistics()を使用してcpuInfoとcpuCountをロードし、各CPUをループして、totalUser、totalUserNice、totalSystem、totalIdleをを設定しています。最後に、この全てのデータをrawDataオブジェクトに追加してから返します。

最後の部分は、Mac OSでのみSysInfoMacImplクラスをコンパイルすることです。次の内容になるように.proファイルを変更します。

```QMake
...

linux {
    SOURCES += sysinfolinuximpl.cpp
    HEADERS += sysinfolinuximpl.h
}

macx {
    SOURCES += sysinfomacimpl.cpp
    HEADERS += sysinfomacimpl.h
}

FORMS += \
    mainwindow.ui
```

***
**[戻る](../index.html)**
