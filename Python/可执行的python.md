一个 .py 文件，或者说一个模块，无论是作为模块被 import 导入，还是直接执行，其中的没有代码缩进的语句（也就是除了函数、类等定义）都会被执行。

而如果想区分是被直接执行，还是作为模块被导入，可以用 `__name__` 区分：

* 当一个模块（其实就是存有 Python 代码的 .py 文件，例如：fun.py）被 import 语句导入的时候，这个模块的 `__name__ `就是模块名（例如：'fun'）。

* 而当一个模块被命令行运行的时候，这个模块的 `__name__` 就被 Python 解释器设定为 `__main__`

```
print("普通语句被执行")
def routine_1():
    print('Routine 1 done.')

def routine_2():
    print('Routine 2 done.')

def main():
    routine_1()
    routine_2()
    print('This is the end of the program.')
    
if __name__ == '__main__':
    main()
```
如此将某些语句放到main中， 且判断`__name__`等于'__main__'，则可以仅在直接执行时才执行