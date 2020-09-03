# SysInfoをシングルトンに変換する

ここでは、SysInfoをシングルトンに変換します。C++は、シングルトンデザインパターンを実装する多くの方法を提供します。ここではそのうちの1つについて説明します。SysInfo.hファイルを開き、次の変更を行います。

```C++
class SysInfo
{
public:
    static SysInfo& instance();
    virtual ~SysInfo();

    virtual void init() = 0;
    virtual double cpuLoadAverage() = 0;
    virtual double memoryUsed() = 0;

protected:
    explicit SysInfo();

private:
    SysInfo(const SysInfo& rhs);
    SysInfo& operator=(const SysInfo &rhs);
};
```

シングルトンは、クラスのインスタンスが1つだけ存在し、このインスタンスが単一のアクセスポイントから簡単にアクセスできることを保証する必要があります。

そこで最初にすべきことは、コンストラクタの可視性をprotectedに変更することです。こうすることで、このクラスと子クラスだけがコンストラクタを呼び出すことができるようになります。

オブジェクトのインスタンスは1つしか存在してはいけないので、コピーコンストラクタと代入演算子を許可するのはナンセンスです。この問題を解決する一つの方法は、これらをプライベートにすることです。

## C++ Tip

C++11以降では、void myFunction() = deleteという構文で関数を削除済みとして定義できます。削除された関数を使用すると、コンパイルエラーが表示されます。これは、コピーコンストラクタと代入演算子をシングルトンで使用しないようにする別の方法です。

最後の変更点は、SysInfoクラスの参照を返す静的関数インスタンスを持つ「一意のアクセスポイント」です。

これでSysInfo.cppファイルにシングルトンの変更をコミットすることができました。

```C++
#include <QtGlobal>

#ifdef Q_OS_WIN
    #include "sysinfowindowsimpl.h"
#elif defined(Q_OS_MAC)
    #include "sysinfomacimpl.h"
#elif defined(Q_OS_LINUX)
    #include "sysinfolinuximpl.h"
#endif

SysInfo &SysInfo::instance()
{
    #ifdef Q_OS_WIN
        static SysInfoWindowsImpl singleton;
    #elif Q_OS_MAC
        static SysInfoMacImpl singleton;
    #elif Q_OS_LINUX
        static SysInfoLinuxImpl singleton;
    #endif

    return singleton;
}

SysInfo::SysInfo()
{

}

SysInfo::~SysInfo()
{

}
```

ここでは、もう一つのQtのクロスOSのトリックを見ることが出来ます。QtはQ_OS_WIN、Q_OS_LINUX、Q_OS_MACというマクロを提供しています。QtのOSマクロは、対応するOS上でのみ定義されています。これらのマクロと条件付きプリプロセッサ司令#ifdefを組み合わせることで、すべてのOS上で常に正しいSysInfo実装をインクルードしてインスタンス化することができます。

instance()関数でシングルトン変数を静的変数として宣言することは、C++でシングルトンを作成する方法です。シングルトンでのメモリ管理を気にする必要がないので、このバージョンが好まれる傾向があります。最初のインスタンス化だけでなく、破棄もコンパイラが処理してくれます。さらに、C++11以降、この方法はスレッドセーフです。

***
**[戻る](../index.html)**
