安装Python，如果采用Python官方下载安装的方式，则会安装在/Library/Frameworks/Python.framework

然后需要在~/.bash_profile中配置了环境变量，添加了路径/Library/Frameworks/Python.framework/Versions/3.7/bin，然后就直接在终端使用命令

Python的更新，没有增量更新，例如要把3.7更新到3.8，就要删除原本的再下载新的


如果使用brew install python3 安装，可以使用 brew list python3 查看安装目录，which python3 查看命令位置。homebrew 会生成软连接 /opt/homebrew/bin/python3，以3.13版本为例，实际链接到 /opt/homebrew/Cellar/python@3.13/3.13.1/Frameworks/Python.framework/Versions/3.13/bin/python3.13

> 在MacOS Monterey 12.3 版本之前，系统会自带python2，之后则自带了python3，位于/Library/Developer/CommandLineTools/Library/Frameworks/Python3.framework