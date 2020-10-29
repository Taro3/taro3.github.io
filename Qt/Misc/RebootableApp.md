# 再起動可能なアプリケーション

Qtで簡単な再起動可能なアプリケーションを作成します。

## ソース

```c++
#ifndef MAINWINDOW_H
#define MAINWINDOW_H

#include <QMainWindow>

QT_BEGIN_NAMESPACE
namespace Ui { class MainWindow; }
QT_END_NAMESPACE

/**
 * @brief The MainWindow class
 */
class MainWindow : public QMainWindow
{
    Q_OBJECT

public:
    static int const EXIT_CODE_REBOOT;  //!< リブート終了コード

    MainWindow(QWidget *parent = nullptr);
    ~MainWindow();

public slots:
    void slotReboot();

private:
    Ui::MainWindow *ui;
};
#endif // MAINWINDOW_H
```

リブート用の定数を定義して、リブートコマンドのイベントハンドラも宣言します。

```c++
#include <QDebug>

#include "mainwindow.h"
#include "ui_mainwindow.h"

/**
 * @brief       MainWindow::MainWindow
 *              コンストラクタ
 * @param[in]   parent  親ウィジェットポインタ
 */
MainWindow::MainWindow(QWidget *parent)
    : QMainWindow(parent)
    , ui(new Ui::MainWindow)
{
    ui->setupUi(this);
}

/**
 * @brief   MainWindow::~MainWindow
 *          デストラクタ
 */
MainWindow::~MainWindow() {
    delete ui;
}

/**
 * @brief   MainWindow::slotReboot
 *          Rebootイベントハンドラ
 */
void MainWindow::slotReboot() {
    qDebug() << "Performing application reboot...";
    qApp->exit(EXIT_CODE_REBOOT);
}
```

リブートコマンドのイベントハンドラでリブート用定数でQApplicationを終了します。

```c++
#include "mainwindow.h"

#include <QApplication>

int const MainWindow::EXIT_CODE_REBOOT = -123456789;

/**
 * @brief qMain
 *        プログラムメイン
 * @param argc
 * @param argv
 * @return
 */
int main(int argc, char *argv[])
{
    int currentExitCode = 0;

    do {
        QApplication a(argc, argv);
        MainWindow w;
        w.show();
        currentExitCode = a.exec();
    } while (currentExitCode == MainWindow::EXIT_CODE_REBOOT);

    return currentExitCode;
}
```

プログラムメインでリブートコマンドで終了した場合は再度QApplicationを実行し続けます。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<ui version="4.0">
 <class>MainWindow</class>
 <widget class="QMainWindow" name="MainWindow">
  <property name="geometry">
   <rect>
    <x>0</x>
    <y>0</y>
    <width>108</width>
    <height>58</height>
   </rect>
  </property>
  <property name="windowTitle">
   <string>MainWindow</string>
  </property>
  <widget class="QWidget" name="centralwidget"/>
  <widget class="QMenuBar" name="menubar">
   <property name="geometry">
    <rect>
     <x>0</x>
     <y>0</y>
     <width>108</width>
     <height>20</height>
    </rect>
   </property>
   <widget class="QMenu" name="menu_File">
    <property name="title">
     <string>&amp;File</string>
    </property>
    <addaction name="actionReboot"/>
   </widget>
   <addaction name="menu_File"/>
  </widget>
  <widget class="QStatusBar" name="statusbar"/>
  <action name="actionReboot">
   <property name="text">
    <string>&amp;Restart</string>
   </property>
   <property name="toolTip">
    <string>Resarts the application</string>
   </property>
  </action>
 </widget>
 <resources/>
 <connections>
  <connection>
   <sender>actionReboot</sender>
   <signal>triggered()</signal>
   <receiver>MainWindow</receiver>
   <slot>slotReboot()</slot>
   <hints>
    <hint type="sourcelabel">
     <x>-1</x>
     <y>-1</y>
    </hint>
    <hint type="destinationlabel">
     <x>399</x>
     <y>299</y>
    </hint>
   </hints>
  </connection>
 </connections>
 <slots>
  <slot>slotReboot()</slot>
 </slots>
</ui>
```

GUI定義です。
メインメニューにRebootのアクションを割り当てています。

***

**[戻る](../Qt.html)**
