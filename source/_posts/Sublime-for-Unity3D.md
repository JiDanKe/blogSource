title: "Unity3D使用Sublime作为编辑器"
date: 2015-04-13 10:42:35
categories: Unity3D
tags: Sublime
---
终于无法忍受MonoDeveloper的龟速，决心换一个工具作为Unity3D的代码编辑器。看身边的朋友开始用Sublime，清凉快速、自动补全、内置构建、配色丰富，总之一个字，酷！

于是请教余老板给配置了个同样的环境，这里记录下过程，分享给大家。

#环境要求：
已安装Unity3D
已安装[Sublime Text 3](http://www.sublimetext.com/3)

#步骤一：给Sublime安装PackageControl
具体方法见PackageControl的[官方网站](https://packagecontrol.io)，如果已安装请跳过。

#步骤二：为Sublime安装OmniSharp插件
OmniSharp**只支持Sublime Text 3**，请在Package Control中进行安装。
在Sublime里按Cmd+Shift+P，输入“Install Package”，回车。在PackageControl的列表中搜索“OmniSharp”，进行安装。

安装完后，菜单选择 Preferences > Settings - More > Syntax Specific - User

![OmniSharp设置.png](/images/4e00efe8f4c0be8ba100cf95a02bb048952cc12c.png)

填入以下内容并保存

```
{
    "auto_complete": true,
    "auto_complete_selector": "source - comment",
    "auto_complete_triggers": [ {"selector": "source.cs", "characters": ".<"} ],
}
```

#步骤三：为Sublime安装Unity3D Build System插件
经测试，通过Package Control安装的这个插件在Sublime Text 3里面有问题，具体表现为Sublime的Packages目录里相关文件没有被下载，囧。
不过我们可以手动安装，选择菜单：Sublime Text > Preferences > Browse Packages...
在这个目录里面运行
git clone [https://github.com/fredpointzero/UnityBuild.git]() "Unity3D Build System"

TIPS:后面必须加的”Unity3D Build System”参数，否则不能正常运行。

#步骤四：设置Unity3D的默认编辑器为Sublime

![设置Unity3D的默认脚本编辑器.png](/images/9b3959bc0da7d1af66af6e97e2df46b137022bb9.png)

#步骤五：创建项目的Sublime工程文件
1. 在Sublime中，选择菜单 File > Open…
在文件选择对话框中，打开Unity3D项目的文件夹，这时Sublime左侧会有树形目录。

2. 选择菜单 Project > Save Project As...
将sublime-project文件保存到Unity3D相应项目的根目录，并以项目名称命名。
例如：Unity3D项目中.Net工程文件为TestProject.sln，则保存为TestProject.sublime-project。
>NOTE:这里有两个sln文件，一个是xxx.sln，另一个是xxx-csharp.sln，选用前者。

3. 修改sublime-project内容
为了更好适配Unity3D开发，我们需要设置一下sublime工程文件。

```
{
    "folders":
    [
        {
            "file_exclude_patterns":
            [
                "*.dll",
                "*.meta",
                "*.asset",
                "*.unity",
                "*.ttf",
                "*.pdf",
                "*.unitypackage",
                "*.mdb",
                "*.prefab",
                "*.ogg",
                "*.controller",
                "*.anim",
                "*.jar",
                ".*"
            ],
            "folder_exclude_patterns":
            [
                "Fonts",
                "Materials",
                "NGUIAtlas",
                "Prefabs",
                "StreamingAssets",
                "Images",
                "Scenes",
                ".*"
            ],
            "path": "Assets",
            "follow_symlinks": true
        }
    ],
    "solution_file": "./TestProject.sln"
}
```

将其中的TestProject.sln改为你的工程文件名。以后双击这个sublime工程文件就可以用Sublime打开工程。

#步骤六：配置快捷键
代码跳转等功能会比较常用，我们把快捷键配置成和MonoDeveloper中一样。
选择菜单：Sublime Text > Package Settings > OmniSharp > KeyBindings User，替换成以下内容并保存(感谢余老板提供：）)。

```
[
    {"command": "omni_sharp_go_to_definition", "keys": ["super+d"]},
    {"command": "omni_sharp_rename", "keys": ["alt+r"]},
    {"command": "omni_sharp_find_usages", "keys": ["shift+super+r"]},
    {"command": "omni_sharp_format_document", "keys": ["super+k"]},
    {"command": "omni_sharp_code_actions", "keys": ["alt+enter"]},
    {"command": "omni_sharp_new_file", "keys": ["super+alt+ctrl+c"], "args":{"paths": []}},
    {"command": "omni_sharp_new_file", "keys": ["super+alt+ctrl+i"], "args":{"tmpltype":"interface","paths": []}},
    {"command": "omni_sharp_hide_panel", "keys": ["alt+x"]},
    {"command": "omni_sharp_show_panel", "keys": ["alt+z"]},
    {"command": "omni_sharp_type_lookup", "keys": ["alt+`"]},
    {"command": "omni_sharp_run_unit_tests", "keys":["ctrl+shift+r"]},
    {"command": "omni_sharp_build_project", "keys":["super+alt+b"]},
    {"command": "omni_sharp_reload_solution", "keys": ["super+option+r"]},
    {"command": "omni_sharp_fix_usings", "keys": ["option+super+u"]}
]
```

经过以上步骤，Sublime for Unity3D的环境就搭建完成，请自行试一下自动补全、编译等功能是否OK。这样除了断点调试，可以把MonoDeveloper晾一边了:P

>补充：第一次建立工程时，需要用MonoDeveloper来生成sln文件。



