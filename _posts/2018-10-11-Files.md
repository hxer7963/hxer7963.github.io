---
layout: keynote
title: Practical Vim Chapter6 Manage Multiple Files
subtitle: Practical Vim notes
date: 2017-08-27 23:46:03
tags: Vim
catalog: true
---

### 技巧37 用缓冲区列表管理打开的文件

#### 文件与缓冲区的区别

Vim允许读取、编辑文件，并保存修改。通常说的“编辑一个文件”，从操作系统的角度来看其实只是编辑文件在**内存中的映像**，也就是Vim术语中的“缓冲区”。

文件存储于磁盘，缓冲区存在于内存。当Vim打开一个文件，该文件的内容被读入一个具有相同名字的缓冲区。绝大多数Vim命令都用来操作缓冲区，但`:w`、`:update`及`:saveas`命令用来操作文件。

#### 结识缓冲区列表

```sh
$ vim *.md	# 打开当前路径下所有的md文件，vim启动一个窗口，窗口内的缓冲区对应第一个文件
:ls		# vim 内部ls命令，列出所有vim缓冲区的文件列表
1 %a   "Chapter5-CommandMode.md"      line 5
2     "CodeforcesRound429Div1.md"    line 0
:bnext	# 切换到列表中下一个缓冲区
```

`%`表示指明当前窗口显示的是哪个缓冲区文件，`#`代表**轮转**文件，`<C-^>`在当前文件和轮转文件中切换。

注：最初没调用`:bnext`命令时，并没有`#`轮转文件；`:ls`列表的开头数字是在缓冲区创建时（即在Shell中用vim编辑文件时）自动分配的。可以使用`:buffer N`命令直接跳转到编号为`N`的缓冲区（参见：`:h :b`）。`:bufdo`命令允许在`:ls`列出的所有缓冲区中执行`Ex`命令（参见：`:h :bufdo`）。

#### 使用缓冲区列表

`:bprev`和`:bnext`在列表中反向/正向移动一项。`:bfirst`和`:blast`跳到列表开头和结尾。

[unimpaired.vim](https://github.com/tpope/vim-unimpaired)插件

##### 常用的Ex命令的别名

`]q: cnext`，`]q: cprevious`；`]a: next`, `[b: bprevious`;

即unimpaired.vim插件定义了下列按键映射：

```Sh
nnoremap <silent> [b :bprevious<CR>
nnoremap <silent> ]b :bnext<CR>
nnoremap <silent> [B :bfirst<CR>
nnoremap <silent> ]B :blast<CR>
```

注：`nnoremap`:`normal no recursion map`，即在normal mode 下的非递归映射；

`<silent>`指执行映射键时，不在**命令行**回显。如果不加`<silent>`，则按下`[b`后将会在命令行模式上显示`:bprevious`；

##### 行操作映射

`[<Space>` 和`]<Space>`在光标前、后新建一行。光标跟着当前行移动，且不切换到插入模式；`[e`和`]e` 在普通模式中，改变当前行与前一行、后一行的顺序。

#### 删除缓冲区

使用`:bdelete N1 N2 N3`命令删除一个缓冲区。如`:N,M db`表示删除标号在[N, M]中的缓冲区。由于缓冲区编号无法手动修改，所以一般不会手动去删除缓冲区，而是将工作区分割成多个窗口、标签页，或是使用参数列表。

### 技巧38 用参数列表将缓冲区分组

```sh
$ vim *.md
:args
[a.md] b.md
```

`:args`显示**参数列表**，参数列表是`vi`的一个功能，而缓冲区列表`:ls`是Vim引入的增强功能，虽然参数列表仍然*简陋*，但Vim对其增强了参数列表的功能。`:args`列表的内容并不一定是启动Vim时传入的参数，可以使用`:args {arglist}`填充参数列表。

#### 用Glob模式指定文件

`*`用于匹配**当前目录**0个或多个字符，`**`可**递归**到子目录，详见：`:h wildcard`和`:h starstar-wildcard`；

### 技巧39 管理隐藏缓冲区

```sh
$ vim *.txt
:ls
  1 %a   "A.txt"  line 1
  2      "B.txt"  line 0
```

`Go`在缓冲区结尾新增一行, ]a 或 :bnext，出现：`E37: No write since last change (add ! to override)`，提示需要使用`!`强制跳转。

```sh
:bnext
:ls
 1 #h + "A.txt"  line 76
 2 %a   "B.txt"  line 1
```

注：`a`表示`active`，即当前窗口所对应的缓冲区文件；`h`表示`hidden`，即隐藏缓冲区；`+`表示缓冲区内容被修改但未被写入到磁盘。

这时使用`:q`退出当前Vim时，Vim会自动将第一个有改动的隐藏缓冲区载入到当前窗口，以便决定是保存修改(`:w[rite]`)还是摒弃此次修改(`:e[dit]`)。如果想退出Vim并**摒弃**所有修改，可用`:qa[ll]`，如果想将所有修改的缓冲区写入磁盘，可用`:wa[ll]`.

### 技巧40 将工作区切分成窗口

#### 创建分割窗口

* `<C-w>s`: 水平分割当前窗口；
* `<C-w>v`: 垂直分割当前窗口；

新生成的窗口会显示与原窗口相同的缓冲区。这在修改**长**文件时比较有效。

* `sp[lit] {file}`: 水平切分当前窗口，并在新窗口中载入`file`；
* `vsp[lit] {file}`: 垂直切分当前窗口，并在新窗口中载入`file`;

#### 窗口之间切换

* `<C-w>w`：窗口间循环切换；`<C-w><C-w>`也可实现切换活动窗口；
* `<C-w>h/j/k/l`: 切换到左/右/上/下边窗口；

详见：`:h window-move-cursor`，在Gvim中可以使用鼠标点击来激活一个窗口(`:h mouse`)。

#### 关闭窗口

| Ex命令       | 普通模式命令   | 作用                   |
| ---------- | -------- | -------------------- |
| `:clo[se]` | `<C-w>c` | 关闭**活动**窗口           |
| `:on[ly]`  | `<C-w>o` | 只保留活动窗口，关闭其他**所有**窗口 |

VIM提供了用于改变窗口大小的按键映射项，详见：`:h window-resize`；

* `<C-w>=`：使所有窗口等高、等宽；
* `<C-w>_`: 最大化活动窗口的**高度**；
* `<C-w>|`：最大化活动窗口的**宽度**；
* `N<C-w>_`: 把活动窗口的高度设为N行；
* `N<C-w>|`: 把活动窗口的宽度设为N行；

