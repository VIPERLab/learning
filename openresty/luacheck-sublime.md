**luacheck是sublime插件社区提供了一款lua语法检查的插件.
由于lua不需要编译, 直接上传到目标机器即可运行, luacheck可以避免上传后因语法错误的重复上传工作.**

##安装sublime
建议安装sublime text 3, [下载地址](https://www.sublimetext.com/3)

##安装package control
package control是sublime的插件管理器, [下载地址和安装方法](https://packagecontrol.io/installation)

公司环境下使用package control可能需要设置http proxy, 设置方法见[KM这篇文章](http://km.oa.com/articles/show/141625)

##安装 sublimelinter & sublimelinter-luacheck
安装好package control后, 可以使用以下步骤安装插件:
1.ctrl + shift +p调出command palette
![图片描述](/tfl/pictures/201607/tapd_10132751_1468654362_5.png)

2.选择 Install Package, 输入包名
![图片描述](/tfl/pictures/201607/tapd_10132751_1468654635_63.png)

使用上述的包安装方法安装 **sublimelinter** 和  **sublimelinter-luacheck** 插件

##安装lua windows & luacheck
1.安装附件中的LuaForWindows
2.安装附件中的luacheck
3.运行luacheck安装目录的install.lua
![图片描述](/tfl/pictures/201607/tapd_10132751_1468656052_8.png)


##配置sublimelinter
*"Preferences" -&gt; "Package Settings" -&gt; "SublimeLinter" -&gt; "Settings User"*
改写user.linter.luacheck.ignore 和 user.linter.paths.windows
```
"user":
 {
	"linters":
	 {
		"luacheck":
		 {
			"ignore":
			 [
				"channel",
				"ngx",
				"self"
			],
		}
	 },
	"paths":
	 {
		"linux": [],
		"osx": [],
		"windows": [
			"C:\\Program Files\\Sublime Text 3\\bin"
		]
	},
}
```

##语法检查效果展示
- 代码行&状态栏的错误展示**
![代码行&状态栏的错误展示](/tfl/pictures/201607/tapd_10132751_1468656210_28.png)

- 保存时列出所有错误**
![保存时列出所有错误](/tfl/pictures/201607/tapd_10132751_1468656222_13.png)



**SublimeLinter-luacheck v2.0版本忽略一些错误的配置**
>Instead of using very limited editor-specific configuration, write a luacheck configuration file. In your case, just put std = "ngx_lua" into a file named .luacheckrc in the root directory of your project.


.luacheckrc
```
std = "ngx_lua"
self = false
ignore = {"61."}
```
ignore中加入"61."，可以实现忽略所有whitespace错误
