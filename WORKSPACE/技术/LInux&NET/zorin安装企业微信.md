~~~bash
# 基于zorin os18来做
sudo dpkg --add-architecture i386
sudo mkdir -pm755 /etc/apt/keyrings
# 这一步如果有存在的key文件，建议删除
wget -O - https://dl.winehq.org/wine-builds/winehq.key | sudo gpg --dearmor -o /etc/apt/keyrings/winehq-archive.key -
sudo wget -NP /etc/apt/sources.list.d/ https://dl.winehq.org/wine-builds/ubuntu/dists/noble/winehq-noble.sources
# 这里最好给apt配置翻墙工具
sudo apt update
sudo apt install --install-recommends winehq-staging
sudo apt install winetricks
winetricks -q corefonts fakechinese gdiplus riched20 windowscodecs
### 这里最好执行一下winecfg，调整成win10的模式
接下来直接安装企业微信
wine --verison # 我这里是10.18
wine Wecom_xxxx.exe之后就开始安装
~~~
	尝试启动没啥问题，但是图标无法正常显示
```desktop
[Desktop Entry]
Comment[zh_CN]=
Comment=
Exec=env WINEPREFIX=/home/ksw/.wine ENABLE_VULKAN_RENDERDOC_CAPTURE=0 DXVK_LOG_LEVEL=none GTK_IM_MODULE=fcitx QT_IM_MODULE=fcitx XMODIFIERS=@im=fcitx wine 'C:\\ProgramData\\Microsoft\\Windows\\Start Menu\\Programs\\企业微信\\企业微信.lnk'
GenericName[zh_CN]=
GenericName=
Icon=xxxxxxxxxxxxxxxxxxxxxxxxxxx
MimeType=
Name[zh_CN]=企业微信
Name=企业微信
Path=/home/ksw/.wine/dosdevices/c:/Program Files (x86)/WXWork
StartupNotify=true
StartupWMClass=wxwork.exe
Terminal=false
TerminalOptions=
Type=Application
X-KDE-SubstituteUID=false
X-KDE-Username= 
```
	写在最后一笔：自己去网上下载了Wxwork的icon并保存之后，把xxxxxxxxxxxxxxxxxxx那个配置进行了更换，终于可以在任务栏上看到图标了。但是由于我安装之前没有执行winecfg进行配置成win10，我当前的兼容模式还是win7，不确定会不会有新的问题。不敢动了。