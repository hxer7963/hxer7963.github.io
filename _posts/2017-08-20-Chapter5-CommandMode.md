---
layout: post
title: Command Line Mode
subtitle: Pratical Vim notes
date: 2017-08-20 14:05
tags: Vim
catalog: true
---

### Vim及其家族

`ed`是最初的UNIX文本编辑器，那时图形显示器非常稀缺，源代码都是打印在纸带上，在电传终端机上编辑，在终端机上输入命令被送到大型机上进行处理，每条命令输出会被**打印**出来。由于终端到大型机的带宽很低，以至于快速打字员输入命令的速度比命令传输的速度还要快。这时需要`ed`提供更简洁的语法，以减少传输命令的字节数；如`p`被用作打印当前行，`%p`被用来打印整个文件(`%`为整个文本范围)。

ed编辑器的进化：`ed`->`em`(editor for mortals) -> en -> ex；

ex把终端屏幕设置成**交互窗口**，并在窗口内显示文本内容（窗口就是常说的**缓冲区**）。这样，修改时就能**实时**看到效果；此屏幕编辑模式由`:visual`命令激活，简写为`:vi`，这就是`vi`名字的由来；也是`Vim`命令行模式中执行的命令又称为`Ex`命令的由来；

Vim为`vi improved`，使用`:h vi-differences`查看两者的区别；

### 技巧27 认识Vim的命令行模式

在普通模式下，使用`:`、`/`(调出查找提示符)或`<C-r>=`(访问表达式寄存器)进入命令行模式，使用`<ESC>`或`<C-[>`回到普通模式；

#### 多标签

* tabnew: 创建新标签页；
* tabc: 关闭当前标签页；
* tabp: 跳转到前一个标签页；
* tabn: 跳转到后一个标签页；
* tabs: 查看所有的标签页；

参考：[vim多标签和多窗口](http://blog.csdn.net/fuxingdaima/article/details/8658342)

Ex读写文件命令：`:edit`和`:write`。分割窗口`:split`,操作参数列表：`:prev/:next`及缓冲区列表`:bprev/:bnext`。参见 `:h ex-cmd-index`

#### 操作缓冲区文本的Ex命令

* `:[range]delete [x]`：删除指定范围内的行[到寄存器x中]
* `:[range]yank [x]`: 复制指定范围的行[到寄存器x中]；
* `:[line]put [x]`：在指定行后粘贴寄存器x中的内容；
* `:[range]copy {address}`: 把指定范围内的行拷贝到{address}指定的行之下；
* `:[range]move {address}`: 把指定范围内的行**移动**到{address}指定行之下；
* `:[range]join`：连接指定范围内的行；
* `:[range]normal {command}`：对指定范围内的每一行执行**普通模式**命令；
* `：[range]substitute/{pattern}/{string}/[flags]`：把指定范围内出现的{pattern}替换为{string}
* `:[range]global/{pattern}/[cmd]`:对指定范围内匹配{pattern}的所有行执行Ex命令{cmd}

### 技巧28 在一行或多个连续行上执行命令

如果Ex命令只包含数字，Vim会解析为行号，其中如果输入的是**负数**(-1)，则移动到当前行的上一行；如果输入的数字大于缓存区中的行数，则移动到末尾行；这与`:$`效果一样；

如果Ex命令在数字后加上命令，如`:3p`、`:3d`则会先跳到指定行再执行命令；

#### 用地址指定范围

`:{start}, {end}p`，`{start}`和`{end}`都是地址，可用*行号*、*查找模式*、*位置标记*作为地址；

#### 用高亮选区指定范围

`.`代表当前行，`$`代表文件末尾行，则`:.,$p`表示打印当前行到文件末尾行之间的文本；`%`表示整个文件`:1,$`。

在**可视模式**选中高亮选区时进入命令行模式，这时会在命令行预先填充`:'<,'>`，其中`'<`和`'>`分别为高亮选区**首行**的位置标记、最后一行的位置标记；位置标记详见->技巧54；

#### 用模式指定范围

```html
<html>
  <head><title>Practical Vim</title></head>
  <body><h1>Practical Vim</h1></body>
</html>
```

`/<html>/,/<\/html>/p`将打印整个`<html>`标签(含)的内容；其中`{start}`为标签`/<html>`，查找模式的**值**(`{address}`)可以进行加减；如`/<html>/+1`；

### 技巧29 使用`:t`和`:m`命令复制和移动行

`:copy`命令的简写形式为`:t`、`:co`，`t`的含义可理解为`to`；

`:6t.`将第6行复制到当前行下方；如果使用的是普通模式，则必须先`6G`跳到指定行，`yy`复制第6行，`<C-o>`快速跳回原先位置，再使用粘贴命令`p`；

`:t6`把当前行复制到第6行下，`:t.`相当于`yyp`；`:'<, '>t0` 把高亮选中的行复制到文件**开头**（文件首行为第一行）；

### 技巧30 在指定范围上执行普通模式

`:'<,'>normal .`对高亮选区中的每一行执行普通操作下的`.`命令；

`:%normal A;`将在整个文件的末尾添加上`;`并返回到普通模式。

在执行指定的普通模式命令前，Vim会先将光标焦点移到到该行**起始处**，即不需要关心光标的位置；

### 技巧31 重复上次的Ex命令

'.'命令不会重复命令行中做出的修改，重复上一次Ex命令：`@:`；`@`表示调用寄存器，`:`指定为命令行寄存器；`:bnext`和`:bprevious`命令操作有点问题...

### 技巧32 自动补全Ex命令

`<C-d>`命令会让Vim显示可用补全列表，参见`:h c_CTRL-D`；使用tab键**正向**遍历补全列表，`<S-Tab>`反向遍历列表（注：Shift按键在浏览器中遍历页面时作用也是一样）

Tab补全行为详见：`:h :command-complete`；

#### 在多个补全项间选择

base shell: `set wildmode=longest, list`

zsh: `set wildmenu`,`set wildmode=full`；

当`wildmenu`选项被启用时，Vim会提供一个补全导航列表。

### 技巧33 把当前单词插入命令行

`<C-r><C-w>`映射项会复制当前光标下的单词并把它插入命令行中；

```javascript
var tally;
for(tally = 1; tally <= 10; tally++) {
  
}
```

将tally全部替换为counter，可以在命令行中输入: `%s/tally/counter/g`；这样需要输入替换前和替换后的单词。注：如果不添加模式`g`则只替换每行的第一个tally；

使用`<C-r><C-w>`的方法：现将光标定位到tally中，使用`*`命令查找tally出现的**每一个**地方（相当于在命令行模式下输入`\tally`）,并将光标定位到下一个tally出现的首字符上；这时使用`cwcounter<ESC>`对单词进行修改后返回普通模式。这时光标在counter的`r`字符上，使用`:%s//<C-r><C-w>/g`将剩下的tally全部替换为`counter`；

注：上面的`//`是将**查找**模式留空，这意味着Vim**重用**上一次查找模式；这是Vim的substitute命令将查找和替换操作彻底分开的**哲学**，但是把查找域留空，会在**substitute命令历史**中留下一项不完整的记录，由于模式通常保存在Vim的**查找历史记录**中，subtitute命令则保存在Ex命令的历史记录中（参见`:h cmdline-history`），可以想到，Vim在Ex命令的查找域留空时，直接将查找历史记录的最后一项作为查找域，而没有将查找域*填充*后保存道Ex命令历史记录中再执行；这将导致再想**重用**之前的subtitute命令时用由于查找域的不同而出错；

如果以后仍需要用到当前的查找域，即需要以完整模式调用Ex历史记录，只需在命令行中使用`<C-r>/`把上次查找域中的内容粘贴进来即可；—> 技巧91

### 技巧34 回溯历史命令

Vim不仅保留Ex命令，还会为查找命令单独保存一份历史记录；命令历史不仅是为当前编辑会话记录的，这些历史即使在退出Vim再重启之后仍然存在，可以使用`set history=1000`来设置历史记录数目；参见：`:h viminfo`；

#### 结识命令行窗口

在普通模式下输入`q:`调出命令行窗口；参见：`:h cmdwin`；

命令行窗口就想一个**常规**的Vim缓存区，只是编辑的对象是命令历史中的每条命令！哲学！

使用命令行历史记录来编辑当前需要的命令，需要注意的是这里的编译并不会改变之前的历史，只是用来生成当前需要执行的命令，执行由历史记录修改而来的命令执行后将成为历史记录的一部分~

使用`:q`退出命令行窗口；修改好历史命令之后，`<CR>`执行光标所在处的命令并退出命令行窗口；这里不能使用`ZZ`来退出命令行窗口，`:h ZZ`->写回并退出当前文件，难道是因为Vim不让用户“修改”（其实在命令行窗口中编辑历史命令并不是修改历史命令，而是为了得到当前需要的命令）

注：如果使用`<CR>`以执行历史命令的方式退出命令行窗口，历史命令执行的对象为打开命令行窗口之前的**活动窗口**；在分割窗口中需要注意当前活动窗口是哪一个；

当在`Ex`命令行模式编辑命令时，如果这时发现需要使用到历史记录，可以使用`<C-f>`切换到命令行窗口，这时已构建的命令仍然会保存在命令行中；

注：`q/`打开查找命令历史的命令行窗口；

### 技巧35 运行Shell命令

Vim命令行模式，在命令前加`!`前缀表示调用**外部**程序；即`:ls`表示显示缓冲区列表的内容，`:!ls`表示调用Shell命令中的`ls`命令；

`%`会被当前文件名替代；（详见`:h :_%`）

```shell
$ vim	# 进入Vim
:shell	# 在Vim内启动交互式Shell
$ pwd
/Users/xxx/
$ exit	# 退出此Shell，并返回Vim
```

在bash Shell 或 zsh shell中均支持**作业控制**，可以使用`<C-z>`将一个作业挂起到后台，使用`fg`命令唤醒一个被挂起的作业，把它移到前台。

```shell
$ vim	# press <C-z>
[1]  + 31801 suspended  mvim -v
$ jobs
[1]  + suspended  mvim -v
$ fg	# 进到Vim, ZZ退出Vim
[1]  + 31862 continued  mvim -v
$
```

#### 把缓冲区的内容作为标准输入或输出

`:!{cmd}`会将Vim缓冲区的内容作为`{cmd}`命令的标准输入、输出，同时会回显`{cmd}`的输出；

`:r !{cmd}`将`{cmd}`的输出读入到当前缓冲区中；

`:w !{cmd}`将当前缓冲区的内容作为指定`{cmd}`的标准输入；

```shell
$ ls -al /etc/ | grep hosts
-rw-r--r--   1 root  wheel     213 Feb 21  2017 hosts	# root用户可读写，其他用户只能读
$ vim /etc/hosts
# 使用 < C-g> 查看文件状态 -> readonly
# 修改文件后，无法保存-> 当前进程没权限啊!
:w !sudo tee % > /dev/null
```

其中，`:w !sudo tee %`表示以超级用户的权限运行tee程序，` > /dev/null`表示`tee`命令的回显(标准输入，即当前缓冲区的内容)写入到“黑洞”文件；

```sh
$vim
:w !sh	# 将当前缓冲区的内容以标准输入的形式传给外部sh命令，并在shell中执行当前换从去的每行内容
:w! sh	# 将缓冲区的内容写入到名为sh的文件，这里的叹号会让Vim覆盖已存在的sh文件
```

#### 使用外部命令过滤缓冲区内容

```shell
# cmdline_mode/email.csv
first name,last name, email
john,smith,john@example.com
drew,neil,drew@vimcasts.org
jane,doe,jane@example.com
```

使用`:2,$!sort -t',' -k2`命令对`[range]`进行排序；其中`-t','`表示数据以`,`分割，`-k2`表示以每行第二个数据为关键字；

Vim可以使用`!{motion}`操作符切换到命令行模式，同时将指定的`{motion}`范围预置于命令行中；如，在第二行使用`!G`将会在命令行预置`:2,$!`；

### 技巧36 批处理运行Ex命令

```html
<!-- cmdline_mode/vimcasts/spisodes-1.html -->
<ol>
	<li>
		<a href="/episodes/show-invisibles/">
			Show invisibles
		</a>
	</li>

	<li>
		<a href="/episodes/tabs-and-spaces/">
			Tabs abd Spaces
		</a>
	</li>
</ol>
```

需要将内容转成纯文本格式：

```txt
Show invisibles:: http://vimcasts.org
Tabs abd Spaces: http://vimcasts.org 
```

#### 逐条执行Ex命令

```sh
:g/href/j	# 含有href的行执行j命令->当前行和下一行合并
:v/href/d	# 删除不含href的行
8 fewer lines
:%norm A: http:vimcasts.org # 在剩下的每行后面添加: http:vimcasts.org
:%s:/\v^[^\>]+\>\s//g	# 删除<a href="">标签
```

可以将上面的命令保存在文件batch.vim中，在Vim中使用`:source batch.vim`执行；

