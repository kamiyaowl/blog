---
layout: post
title: "vscode+lldbでstdinを扱える環境の構築"
date: 2019-03-01
description: 'vscodeにおけるc++開発環境構築'
main-class: 'jekyll'
image: ''
color: '#B31917'
tags:
- vscode
- c++
categories:
introduction: 
---

## Problem: stdinを扱うデバッグができない

C/C++の拡張機能(type:cppdbg)にはconsoleが指定できず、externalConsoleを使うほかなかった。integrate terminalがあるのに使えないのはあんまりではないか。

あと欲張りな注文だがファイルからの標準入力もやらせたい。デバッグのたびに打つのは流石に人権が怪しい。

目標としては以下の2つを実現
* 現在のディレクトリあるファイルをclangでコンパイル→実行
* 実行する際に、同じディレクトリにあるinput.txtを標準入力にぶっこむ


## Answer: CodeLLDBがよさそう

いろいろ試行錯誤したところCodeLLDBを使って上げると良さそうだった。手順を示す

### clang, lldb環境の構築

macの場合、xcode関係のツールを入れると一緒に入るみたい。ほかは適宜。

### CodeLLDB

vscodeの拡張検索でCodeLLDBを検索→インストール。

### .vscode/tasks.jsonの編集

ビルド用のタスクを定義します。ここではa.outに出力させます。

```
{
    "version": "2.0.0",
    "tasks": [
        {
            "label": "build before debug",
            "type": "shell",
            "command": "clang++",
            "args": ["-std=c++17", "-g", "${fileDirname}/*.cpp", "-o", "${fileDirname}/a.out"],
            "group": {
                "kind": "build",
                "isDefault": true
            }
        }
    ]
}
```

### .vscode/launch.jsonの編集

まずは実行できるコンフィグから。type:lldbであれば、先のCodeLLDBが使える。こちらではterminalの指定ができる。

```
{
    "name": "(lldb) Launch",
    "type": "lldb",
    "request": "launch",
    "program": "${fileDirname}/a.out",
    "args": [],
    "cwd": "${fileDirname}",
    "terminal": "integrated",
    "preLaunchTask": "build before debug"
}
```

次は標準入力を扱う方。[vscode-lldb/MANUAL.md](https://github.com/vadimcn/vscode-lldb/blob/master/MANUAL.md##stdio)に詳細が記載されているが、stdioの指定することでstdin,stdout,stderrをファイルから流したり吐き出したりできるらしい。かなりありがたい。

```
{
    "name": "(lldb) Launch with stdin(./input.txt)",
    "type": "lldb",
    "request": "launch",
    "program": "${fileDirname}/a.out",
    "args": [],
    "cwd": "${fileDirname}",
    "terminal": "integrated",
    "stdio": ["${fileDirname}/input.txt", null, null],
    "preLaunchTask": "build before debug"
}
```


## 総括

便利になった。きっとRustのデバッグも全く同じ手順でできそう。
ところでMacBook Airを買ったが最後に使ったのがSnow Leopardで全然なれない。おしゃれ力はめっちゃ上がった気がするが。