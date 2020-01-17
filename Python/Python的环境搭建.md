自己电脑安装Python，采用了Python官方下载安装的方式，没有使用brew。安装在/Library/Frameworks/Python.framework

自己在~/.bash_profile中配置了环境变量，添加了路径/Library/Frameworks/Python.framework/Versions/3.7/bin，然后就直接在终端使用命令

Python的更新，没有增量更新，例如要把3.7更新到3.8，就要删除原本的再下载新的