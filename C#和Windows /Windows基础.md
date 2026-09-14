Windows原生应用开发提供了多种技术框架：
* win32 API是最基础的C风格编程接口，是内核api的封装，提供了UI、网络、文件、进程、线程等功能。gdi和directx也属于win32 api。
* WinRT API是微软提供的更新的C++风格编程接口，内部可能会直接调用内核api，也可能调用win32 api
* .net framework则是更上层的C#框架，内部仍然会依赖win32等api。
* WPF、WinForm、UWP、WinUI、Maui：这些都属于依赖.net的UI框架。

## Win32 创建窗口App示例
```
#include <windows.h>
#include <stdio.h>

// 窗口处理函数，处理窗口消息
// 各种输入事件、绘制都是通过消息，比如绘制是WM_PAINT消息，在按钮区域按下鼠标左键是WM_LBUTTONDOWN消息
// 当用户点击按钮时，系统会发送一系列消息，包括鼠标事件（如 WM_LBUTTONDOWN、WM_LBUTTONUP）、
// 焦点事件（如 WM_SETFOCUS）、绘制事件（如 WM_PAINT），以及父窗口的 WM_COMMAND 消息
LRESULT CALLBACK WindowProc(HWND hwnd, UINT uMsg, WPARAM wParam, LPARAM lParam) {
    switch (uMsg) {
        case WM_COMMAND: {
            // 检查是否是按钮点击事件
            if (LOWORD(wParam) == 1001) { // 按钮的 ID
                printf("click\n");
            }
            break;
        }
        case WM_DESTROY: {
            // 窗口关闭时退出程序
            PostQuitMessage(0);
            return 0;
        }
    }
    // 默认消息处理
    return DefWindowProc(hwnd, uMsg, wParam, lParam);
}

// 窗口程序入口点
// hInstance: 应用程序实例句柄
// hPrevInstance: 前一个实例句柄（在现代 Windows 系统中，此参数始终为 NULL）
// lpCmdLine: 命令行参数
// nCmdShow: 窗口显示方式（如最大化、最小化等）
int WINAPI WinMain(HINSTANCE hInstance, HINSTANCE hPrevInstance, LPSTR lpCmdLine, int nCmdShow) {

    // 1. 注册窗口类
    // 定义了窗口的基本属性和行为
    const wchar_t CLASS_NAME[] = L"SimpleWindowClass";
    
    WNDCLASS wc = { 0 };
    wc.lpfnWndProc = WindowProc;          // 窗口处理函数
    wc.hInstance = hInstance;             // 应用程序实例句柄
    wc.lpszClassName = CLASS_NAME;        // 窗口类名
    wc.hCursor = LoadCursor(NULL, IDC_ARROW); // 默认光标
    wc.hbrBackground = (HBRUSH)(COLOR_WINDOW + 1); // 背景色

    // 创建主窗口使用了 CLASS_NAME，所以这里注册的窗口类会影响主窗口。创建按钮使用的是 L"BUTTON"，所以和这个窗口类无关。可以注册多个窗口类用于不同的window/控件，比如专门给按钮注册一个窗口类，提供专门的窗口处理函数。
    RegisterClass(&wc);

    // 2. 创建主窗口
    // HWND 是 Windows 中用于标识窗口的句柄类型。它的全称是 Handle to a Window。
    HWND hwnd = CreateWindow(
        CLASS_NAME,                      // 窗口类名
        L"Simple Window",                // 窗口标题
        WS_OVERLAPPEDWINDOW,             // 窗口样式
        CW_USEDEFAULT, CW_USEDEFAULT,    // 窗口位置（默认）
        400, 300,                        // 窗口大小（宽，高）
        NULL,                            // 父窗口（无）
        NULL,                            // 菜单（无）
        hInstance,                       // 应用程序实例
        NULL                             // 附加数据（无）
    );

    if (hwnd == NULL) {
        return 1; // 创建失败
    }

    // 3. 创建按钮（win32中窗口是一个广义的概念，所以控件也是通过此API创建）
    // 系统内置的控件类名：BUTTON、EDIT等，可以使用它们传给CreateWindow
    CreateWindow(
        L"BUTTON",                       // 控件类名
        L"Click Me",                     // 按钮文本
        WS_TABSTOP | WS_VISIBLE | WS_CHILD | BS_DEFPUSHBUTTON, // 按钮样式
        150, 100,                        // 按钮位置（x, y）
        100, 50,                         // 按钮大小（宽，高）
        hwnd,                            // 父窗口
        (HMENU)1001,                     // 按钮 ID
        hInstance,                       // 应用程序实例
        NULL                             // 附加数据
    );

    // 4. 显示窗口
    ShowWindow(hwnd, nCmdShow);
    UpdateWindow(hwnd);

    // 5. 消息循环
    MSG msg = { 0 };
    // 启动消息循环，获取消息
    while (GetMessage(&msg, NULL, 0, 0)) {
        TranslateMessage(&msg); // 将键盘按键消息（如 WM_KEYDOWN）转换为字符消息（如 WM_CHAR）
        DispatchMessage(&msg); // 转发到窗口处理函数中处理
    }

    return (int)msg.wParam;
}
```