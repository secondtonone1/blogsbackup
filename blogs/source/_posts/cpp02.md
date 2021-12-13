---
title: vscode搭建windows C++开发环境
date: 2021-12-10 15:33:02
categories: C++
tags: C++
---

## 简介
本文介绍如何在windows环境下，通过vscode搭建C++的开发环境
需要准备如下文件
1  vscode 软件
2  安装vscode开发插件
3  MinGW
<!--more-->
## 安装vscode
[下载地址](https://code.visualstudio.com/)
选择Download for windows 就可以了
## 安装vscode插件
安装好vscode后打开，选择左侧应用扩展或者按住Ctrl + shift + x 唤出扩展应用界面，输入C++，选择C++插件安装
![C++扩展](https://cdn.llfc.club/1639121954%281%29.jpg)
## 安装MinGW
下载地址：[https://sourceforge.net/projects/mingw-w64/files/](https://sourceforge.net/projects/mingw-w64/files/)
如果下载速度慢，可以去网盘下载
链接: [https://pan.baidu.com/s/1rAJNlB-iqC950lf-6tImGg](https://pan.baidu.com/s/1rAJNlB-iqC950lf-6tImGg) 
提取码: h1zm 

下载的文件：进入网站后不要点击 "Download Lasted Version"，往下滑，找到最新版的 "x86_64-posix-seh"。
![https://cdn.llfc.club/1639122968.jpg](https://cdn.llfc.club/1639122968.jpg)
安装MinGW：下载后是一个7z的压缩包，解压后移动到你想安装的位置即可。我的安装位置是：D:\cppsoft\mingw64
## 配置环境变量
把安装目录D:\cppsoft\mingw64 配置在用户的环境变量path里即可
选择用户环境变量path
![https://cdn.llfc.club/1639123237.jpg](https://cdn.llfc.club/1639123237.jpg)
点击后添加D:\cppsoft\mingw64
![https://cdn.llfc.club/1639123293.jpg](https://cdn.llfc.club/1639123293.jpg)
点确定保存后开启cmd输入g++,如提示no input files 则说明Mingw64 安装成功,如果提示'g++' 不是内部或外部命令，也不是可运行的程序或批处理文件则说明安装失败
![https://cdn.llfc.club/1639123513.jpg](https://cdn.llfc.club/1639123513.jpg)
## 配置vscode
我的项目结构是这样的，inc文件夹下为.h文件 src文件夹下为.cpp文件
![https://cdn.llfc.club/1639124154%281%29.jpg](https://cdn.llfc.club/1639124154%281%29.jpg)
选中main.cpp文件，单击左侧三角形，会进入调试界面添加配置环境，选择C ++ windows 就会自动生成launch.json文件
![https://cdn.llfc.club/1639119249%281%29.jpg](https://cdn.llfc.club/1639119249%281%29.jpg)
我们修改launch.json文件 如下
``` json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "g++.exe build and debug active file",
            "type": "cppdbg",
            "request": "launch",
            "program": "${fileDirname}\\${fileBasenameNoExtension}.exe",
            "args": [],
            "stopAtEntry": false,
            "cwd": "${workspaceFolder}",
            "environment": [],
            "externalConsole": true, //修改此项，让其弹出终端
            "MIMode": "gdb",
            "miDebuggerPath": "D:\\cppsoft\\mingw64\\bin\\gdb.exe",
            "setupCommands": [
                {
                    "description": "Enable pretty-printing for gdb",
                    "text": "-enable-pretty-printing",
                    "ignoreFailures": true
                }
            ],
            "preLaunchTask": "task g++" //修改此项
        }
    ]
}
```
我将 miDebuggerPath 配置为 "D:\\cppsoft\\mingw64\\bin\\gdb.exe"
你们配置为自己的mingw路径就好
我将 preLaunchTask 设置为 "task g++" 这个名字可以随便取也可以用默认的
此时再次点击三角形或者F5运行会提示没有task文件，vscode会自动生成task.json文件
编辑task.json文件，配置如下
``` json
{
	"version": "2.0.0",
	"tasks": [
		{
			"type": "shell",
			"label": "task g++", //修改此项
			"command": "D:\\cppsoft\\mingw64\\bin\\g++.exe",
			"args": [
				"-g",
				// "${file}",
				"${cwd}//src//*.cpp",
				"-o",
				"${fileDirname}\\${fileBasenameNoExtension}.exe"
			],
			"options": {
				"cwd": "D:\\cppsoft\\mingw64\\bin"
			},
			"problemMatcher": [
				"$gcc"
			],
			"group": "build"
		}
	]
}
```
我在args中添加了src文件目录，这样就可以编译多个cpp文件
同时设置label为"task g++",这个和lauch.json中preLaunchTask 对应
options 设置cwd为自己的mingw路径
再次运行，可以看到弹出对话框显示程序执行结果
![https://cdn.llfc.club/1639125121%281%29.jpg](https://cdn.llfc.club/1639125121%281%29.jpg)
如果还是有配置不明白的地方，可以看[ https://gitee.com/secondtonone1/cpplearn]( https://gitee.com/secondtonone1/cpplearn)




