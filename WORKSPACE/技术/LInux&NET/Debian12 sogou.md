~~~shell
apt purge fcitx* ibus
apt autoremove
apt install fcitx
sudo apt install libqt5qml5 libqt5quick5 libqt5quickwidgets5 qml-module-qtquick2
sudo apt install libgsettings-qt1
##下载搜狗deb包，进行dpkg -i 安装
## 在fcitx配置中配置搜狗
~~~

~~~bash
# !/bin/bash
##https://www.bilibili.com/read/cv30070217/
rm /opt/sogoupinyin/files/lib/qt5/lib/libQt5Core.so.5
ln -s /usr/lib/x86_64-linux-gnu/libQt5Core.so.5 /opt/sogoupinyin/files/lib/qt5/lib/libQt5Core.so.5

rm /opt/sogoupinyin/files/lib/qt5/lib/libQt5Gui.so.5
ln -s /usr/lib/x86_64-linux-gnu/libQt5Gui.so.5 /opt/sogoupinyin/files/lib/qt5/lib/libQt5Gui.so.5

rm /opt/sogoupinyin/files/lib/qt5/lib/libQt5Widgets.so.5
ln -s /usr/lib/x86_64-linux-gnu/libQt5Widgets.so.5 /opt/sogoupinyin/files/lib/qt5/lib/libQt5Widgets.so.5

rm /opt/sogoupinyin/files/lib/qt5/lib/libQt5DBus.so.5
ln -s /usr/lib/x86_64-linux-gnu/libQt5DBus.so.5 /opt/sogoupinyin/files/lib/qt5/lib/libQt5DBus.so.5

rm /opt/sogoupinyin/files/lib/qt5/lib/libQt5Qml.so.5
ln -s /usr/lib/x86_64-linux-gnu/libQt5Qml.so.5 /opt/sogoupinyin/files/lib/qt5/lib/libQt5Qml.so.5

rm /opt/sogoupinyin/files/lib/qt5/lib/libQt5Network.so.5
ln -s /usr/lib/x86_64-linux-gnu/libQt5Network.so.5 /opt/sogoupinyin/files/lib/qt5/lib/libQt5Network.so.5

rm /opt/sogoupinyin/files/lib/qt5/lib/libQt5Quick.so.5
ln -s /usr/lib/x86_64-linux-gnu/libQt5Quick.so.5 /opt/sogoupinyin/files/lib/qt5/lib/libQt5Quick.so.5

rm /opt/sogoupinyin/files/lib/qt5/lib/libQt5QuickWidgets.so.5
ln -s /usr/lib/x86_64-linux-gnu/libQt5QuickWidgets.so.5 /opt/sogoupinyin/files/lib/qt5/lib/libQt5QuickWidgets.so.5

rm /opt/sogoupinyin/files/lib/qt5/lib/libQt5Svg.so.5
ln -s /usr/lib/x86_64-linux-gnu/libQt5Svg.so.5 /opt/sogoupinyin/files/lib/qt5/lib/libQt5Svg.so.5

rm /opt/sogoupinyin/files/lib/qt5/lib/libQt5XcbQpa.so.5
ln -s /usr/lib/x86_64-linux-gnu/libQt5XcbQpa.so.5 /opt/sogoupinyin/files/lib/qt5/lib/libQt5XcbQpa.so.5

rm /opt/sogoupinyin/files/lib/qt5/lib/libFcitxQt5DBusAddons.so.1
ln -s /usr/lib/x86_64-linux-gnu/libFcitxQt5DBusAddons.so.1 /opt/sogoupinyin/files/lib/qt5/lib/libFcitxQt5DBusAddons.so.1

rm /opt/sogoupinyin/files/lib/qt5/lib/libgsettings-qt.so.1
ln -s /usr/lib/x86_64-linux-gnu/libgsettings-qt.so.1 /opt/sogoupinyin/files/lib/qt5/lib/libgsettings-qt.so.1

rm /opt/sogoupinyin/files/lib/qt5/lib/libpcre.so.3
ln -s /usr/lib/x86_64-linux-gnu/libpcre.so.3 /opt/sogoupinyin/files/lib/qt5/lib/libpcre.so.3

# plugins

rm /opt/sogoupinyin/files/lib/qt5/plugins/iconengines/libqsvgicon.so
ln -s /usr/lib/x86_64-linux-gnu/qt5/plugins/iconengines/libqsvgicon.so /opt/sogoupinyin/files/lib/qt5/plugins/iconengines/libqsvgicon.so

rm /opt/sogoupinyin/files/lib/qt5/plugins/imageformats/libqsvg.so
ln -s /usr/lib/x86_64-linux-gnu/qt5/plugins/imageformats/libqsvg.so /opt/sogoupinyin/files/lib/qt5/plugins/imageformats/libqsvg.so

rm /opt/sogoupinyin/files/lib/qt5/plugins/platforminputcontexts/libfcitxplatforminputcontextplugin.so
ln -s /usr/lib/x86_64-linux-gnu/qt5/plugins/platforminputcontexts/libfcitxplatforminputcontextplugin.so /opt/sogoupinyin/files/lib/qt5/plugins/platforminputcontexts/libfcitxplatforminputcontextplugin.so

rm /opt/sogoupinyin/files/lib/qt5/plugins/platforms/libqlinuxfb.so
ln -s /usr/lib/x86_64-linux-gnu/qt5/plugins/platforms/libqlinuxfb.so /opt/sogoupinyin/files/lib/qt5/plugins/platforms/libqlinuxfb.so

rm /opt/sogoupinyin/files/lib/qt5/plugins/platforms/libqminimal.so
ln -s /usr/lib/x86_64-linux-gnu/qt5/plugins/platforms/libqminimal.so /opt/sogoupinyin/files/lib/qt5/plugins/platforms/libqminimal.so

rm /opt/sogoupinyin/files/lib/qt5/plugins/platforms/libqoffscreen.so
ln -s /usr/lib/x86_64-linux-gnu/qt5/plugins/platforms/libqoffscreen.so /opt/sogoupinyin/files/lib/qt5/plugins/platforms/libqoffscreen.so

rm /opt/sogoupinyin/files/lib/qt5/plugins/platforms/libqxcb.so
ln -s /usr/lib/x86_64-linux-gnu/qt5/plugins/platforms/libqxcb.so /opt/sogoupinyin/files/lib/qt5/plugins/platforms/libqxcb.so

rm /opt/sogoupinyin/files/lib/qt5/plugins/xcbglintegrations/libqxcb-glx-integration.so
ln -s /usr/lib/x86_64-linux-gnu/qt5/plugins/xcbglintegrations/libqxcb-glx-integration.so /opt/sogoupinyin/files/lib/qt5/plugins/xcbglintegrations/libqxcb-glx-integration.so
~~~
