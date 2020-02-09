# 特性说明

本插件使用 Vim 8 / NeoVim 的异步机制，让你在后台运行 shell 命令，并将结果实时显示到 Vim 的 Quickfix 窗口中：

- 使用简单，输入 `:AsyncRun {command}` 即可在后台执行你的命令（和传统的 `!` 命令类似）。
- 命令会在后台运行，不会阻碍 Vim 操作，不需要等待整个命令结束你就继续操作 Vim。
- 进程的输出会实时的显示在下方的 quickfix 窗口里，编译信息会自动同 `errorformat` 匹配。
- 你可以立即浏览错误输出，或者在任务执行的同时继续编辑你的文件。
- 当任务结束时，播放一个铃声提醒你，避免你眼睛盯着代码忘记编译已经结束。
- 丰富的参数和配置项目，可以自由指定运行方式，命令初始目录，autocmd 触发等。
- 除了异步任务+quickfix 外，还提供多种运行方式，比如一键在内置终端里运行命令。
- 快速和轻量级，无其他依赖，仅仅单个 `asyncrun.vim` 源文件。
- 同时为 Vim/NeoVim/GVim/MacVim 提供一致的用户体验。

具体运行效果，可以看下面的 GIF 截屏。

# 新闻

- 2020/01/21 使用 `-mode=term` 在内置终端里运行你的命令，见 [这里](https://github.com/skywind3000/asyncrun.vim/wiki/Specify-how-to-run-your-command).
- 2018/04/17 支持 range 了，可以 Vim 中选中一段文本，然后 `:%AsyncRun cat`。
- 2017/06/26 新增参数 `-cwd=<root>` 可以指定在项目的根目录运行命令，见 [project-root](#项目根目录).

# 安装

拷贝 `asyncrun.vim` 到你的 `~/.vim/plugin` 目录，或者用 vim-plug/Vundle 之类的包管理工具从 `skywind3000/asyncrun.vim` 位置安装。

# 例子

![](doc/screenshot.gif)

异步运行 gcc/grep 的演示，别忘记在运行前使用 `:copen` 命令打开 vim 的 quickfix 窗口，否则你看不到具体输出，还可以设置 `g:asyncrun_open=6` 来自动打开。

# 内容目录

<!-- TOC -->

- [快速入门](#快速入门)
- [使用手册](#使用手册)
    - [AsyncRun - 运行 shell 命令](#asyncrun---运行-shell-命令)
    - [AsyncStop - 停止正在运行的任务](#asyncstop---停止正在运行的任务)
    - [函数接口](#函数接口)
    - [全局设置](#全局设置)
    - [全局变量](#全局变量)
    - [Autocmd](#autocmd)
    - [项目根目录](#项目根目录)
    - [运行模式](#运行模式)
    - [内置终端](#内置终端)
    - [Quickfix window](#quickfix-window)
    - [Range 支持](#range-支持)
    - [运行需求](#运行需求)
    - [同 fugitive 协作](#同-fugitive-协作)
- [语言参考](#语言参考)
- [更多话题](#更多话题)
- [插件协作](#插件协作)
- [Credits](#credits)

<!-- /TOC -->

## 快速入门

**异步运行 gcc 编译当前的文件**

	:AsyncRun gcc % -o %<
	:AsyncRun g++ -O3 "%" -o "%<" -lpthread 

上面的命令会在后台运行 gcc 命令，并把编译输出实时显示到 quickfix 窗口中，标记 '`%`' 代表当前正在编辑的文件名，而 '`%<`' 代表去掉扩展名的文件名。

**异步运行 make**

    :AsyncRun make
	:AsyncRun make -f makefile

记得在执行 `AsyncRun` 命令前，提前使用 `copen` 命令打开 quickfix 窗口，不然你看不到任何内容。

**Grep 关键字**

    :AsyncRun! grep -n -R word . 
    :AsyncRun! grep -n -R <cword> . 
    
当 `AsyncRun` 命令后面追加一个叹号时，quickfix 将不会自动滚动，保持在第一行。`<cword>` 代表光标下面的单词。

**编译 go 项目**

    :AsyncRun go build "%:p:h"

标记 '`%:p:h`' 表示当前文件的所在目录。 

**查看 man page**

    :AsyncRun! man -S 3:2:1 <cword>

**异步 git push**

    :AsyncRun git push origin master

**初始化 `<F7>` 来编译文件**

    :noremap <F7> :AsyncRun gcc "$(VIM_FILEPATH)" -o "$(VIM_FILEDIR)/$(VIM_FILENAME)" <cr> 

文件可能会包含空格，所以正式使用推荐用 `$(...)` 的宏形，后面有表格说明可用宏。

**运行 Python 脚本**

    :AsyncRun -raw python %

使用 `-raw` 参数可以在 quickfix 中显示原始输出（不进行 errorformat 匹配），记得用 `let $PYTHONNUNBUFFERED=1` 来禁止 python 的行缓存，这样可以实时查看结果。很多程序在后台运行时都会将输出全部缓存住直到调用 flush 或者程序结束，python 可以设置该变量来禁用缓存，让你实时看到输出，而无需每次手工调用 `sys.stdout.flush()`。

关于缓存的更多说明见 [这里](https://github.com/skywind3000/asyncrun.vim/wiki/FAQ#cant-see-the-realtime-output-when-running-a-python-script).

## 使用手册

本插件有且只提供了两条命令：`:AsyncRun` 以及 `:AsyncStop` 来控制你的任务。

### AsyncRun - 运行 shell 命令

```VimL
:AsyncRun[!] [options] {cmd} ...
```

在后台运行 shell 命令，并把结果实时输出到 quickfix 窗口，当命令后跟随一个 `!` 时，quickfix 将不会自动滚动。参数用空格分隔，如果某项参数包含空格，那么需要双引号引起来（unix 下面还可以使用反斜杠加空格）。

参数可以接受各种以 '`%`', '`#`' or '`<`' 开头的文件名宏：

    %:p     - 当前 buffer 的文件名全路径
    %:t     - 当前 buffer 的文件名（没有前面的路径）
    %:p:h   - 当前 buffer 的文件所在路径
    %:e     - 当前 buffer 的扩展名
    %:t:r   - 当前 buffer 的主文件名（没有前面路径和后面扩展名）
    %       - 相对于当前路径的文件名
    %:h:.   - 相对于当前路径的文件路径
    <cwd>   - 当前路径
    <cword> - 光标下的单词
    <cfile> - 光标下的文件名
    <root>  - 当前 buffer 的项目根目录

在运行前会批量初始化一些环境变量（方便你在 shell 脚本中使用）：

    $VIM_FILEPATH  - 当前 buffer 的文件名全路径
    $VIM_FILENAME  - 当前 buffer 的文件名（没有前面的路径）
    $VIM_FILEDIR   - 当前 buffer 的文件所在路径
    $VIM_FILEEXT   - 当前 buffer 的扩展名
    $VIM_FILENOEXT - 当前 buffer 的主文件名（没有前面路径和后面扩展名）
    $VIM_PATHNOEXT - 带路径的主文件名（$VIM_FILEPATH 去掉扩展名）
    $VIM_CWD       - 当前 Vim 目录
    $VIM_RELDIR    - 相对于当前路径的文件名
    $VIM_RELNAME   - 相对于当前路径的文件路径
    $VIM_ROOT      - 当前 buffer 的项目根目录
    $VIM_CWORD     - 光标下的单词
    $VIM_CFILE     - 光标下的文件名
    $VIM_GUI       - 是否在 GUI 下面运行？
    $VIM_VERSION   - Vim 版本号
    $VIM_COLUMNS   - 当前屏幕宽度
    $VIM_LINES     - 当前屏幕高度
    $VIM_SVRNAME   - v:servername 的值

上面这些环境变量，可以使用 `$(...)` 的形式（比如 `$(VIM_FILENAME)`之类）用在 AsyncRun 的参数里面，这样的用法比 `%` 前缀的 vim 宏要安全，因为百分号的 vim 宏是由 vim 展开的，当使用 `-cwd=?` 时百分号宏会失效，所以调用 `AsyncRun` 命令时，推荐使用 `$(...)` 形式的参数。

宏 `$(VIM_ROOT)` 或者 `<root>` 指代当前文件的[项目根目录](https://github.com/skywind3000/asyncrun.vim/wiki/Project-Root)。

调用 `AsyncRun` 时，在具体的命令前面可以有一些 `-` 开头的参数：

| 参数 | 默认值 | 含义 |
|-|-|-|
| `-mode=?` | "async" | 用 `-mode=?` 的形式指定运行模式可选模式有： `"async"` (默认模式，后台运行输出到 quickfix), `"terminal"` (在内建终端运行), `"bang"` (使用 `!` 命令运行) and `"os"` (新的外部终端窗口)，具体查看 [运行模式](#运行模式)。 |
| `-cwd=?` | `未设置` | 命令初始目录（没有设置就用 vim 当前目录），比如  `-cwd=<root>` 就能在 [项目根目录](#项目根目录) 运行命令，或者 `-cwd=$(VIM_FILEDIR)` 就能在当前文件所在目录运行命令。 |
| `-save=?` | 0 | 运行命令前是否保存文件，`-save=1` 保存当前文件，`-save=2` 保存所有修改过的文件 |
| `-program=?` | `未设置` | 设置成 `make` 可以用 `&makeprg`，设置成 `grep` 可以使用 `&grepprt`，而设置成 `wsl` 则可以在 WSL 中运行命令 （需要 Windows 10）|
| `-post=?` | `未设置` | 命令结束后自动运行的 vimscript，如果包含空格则要用反斜杠加空格代替。 |
| `-auto=?` | `未设置` | 出发 autocmd `QuickFixCmdPre`/`QuickFixCmdPost` 后面的名称。 |
| `-raw` | `未设置` | 如果提供了，就输出原始内容，忽略 `&errorformat` 过滤。 |
| `-strip` | `未设置` | 过滤收尾消息 (头部命令名称以及尾部 "[Finished in ...]" 信息)。|
| `-pos=?` | "bottom" | 当用 `-mode=term` 在内置终端运行命令时， `-pos` 用于指定内置终端窗口位置， 可以设置成 `"tab"`，`"curwin"`，`"top"`，`"bottom"`，`"left"` 以及 `"right"`。|
| `-rows=num` | 0 | 内置终端窗口的高度。|
| `-cols=num` | 0 | 内置终端窗口的宽度。|
| `-errorformat=?` | `未设置` | 用于 quickfix 中匹配错误输出的格式字符串，如果未提供，则使用当前 `&errorformat` 的值。 |

所有的这些配置参数都必须放在具体 shell 命令 **前面**，因为没有任何 shell 命令使用 `-` 开头，因此很容易区分哪里是命令的开始。如果你确实有一条 shell 命令是减号开头的，那么为了明显区别参数和命令，可以在命令前面放一个 `@` 符号，那么 AsyncRun 在解析参数时碰到 `@` 就知道参数结束了，后面都是命令。


### AsyncStop - 停止正在运行的任务

```VimL
:AsyncStop[!]
```

没有叹号时，使用 `TERM` 信号尝试终止后台任务，有叹号时会使用 `KILL` 信号来终止。

### 函数接口

本插件提供了函数形式的接口，方便你在 vimscript 中调用：

```VimL
:call asyncrun#run(bang, opts, command)
```

参数说明:

- `bang`：空字符串或者内容是一个叹号的字符串 `"!"` 和 `AsyncRun!` 的叹号作用一样。
- `opts`：参数字典，包含：`mode`, `cwd`, `raw` 以及 `errorformat` 等。
- `command`：具体要运行的命令。

### 全局设置

- g:asyncrun_exit - 命令结束时自动运行的 vimscript。
- g:asyncrun_bell - 命令结束后是否响铃？
- g:asyncrun_mode - 全局的默认[运行模式](#运行模式).
- g:asyncrun_encs - 如果系统编码和 Vim 内部编码 `&encoding`，不一致，那么在这里设置一下，具体见 [编码设置](https://github.com/skywind3000/asyncrun.vim/wiki/Quickfix-encoding-problem-when-using-Chinese-or-Japanese)。
- g:asyncrun_trim - 设置成非零的话剔除空白行。
- g:asyncrun_auto - 用于触发 QuickFixCmdPre/QuickFixCmdPost 的 autocmd 名称，见 [FAQ](https://github.com/skywind3000/asyncrun.vim/wiki/FAQ#can-asyncrunvim-trigger-an-autocommand-quickfixcmdpost-to-get-some-plugin-like-errormaker-processing-the-content-in-quickfix-)。
- g:asyncrun_open - 大于零的话会在运行时自动打开高度为具体值的 quickfix 窗口。
- g:asyncrun_save - 全局设置，运行前是否保存文件，1是保存当前文件，2是保存所有修改过的文件。
- g:asyncrun_timer - 每 100ms 处理多少条消息，默认为 25。
- g:asyncrun_wrapper - 命令前缀，默认为空，比如可以设置成 `nice`。
- g:asyncrun_stdin - 设置成非零的话，允许 stdin，比如 cmake 在 windows 下要求 stdin 为打开状态。

更多配置内容，见 **[这里](https://github.com/skywind3000/asyncrun.vim/wiki/Options)**.


### 全局变量

- g:asyncrun_code - 命令返回码。
- g:asyncrun_status - 命令状态：'running', 'success' or 'failure' 可以用于设置到 statusline。

### Autocmd

```VimL
autocmd User AsyncRunPre   - 运行前出发
autocmd User AsyncRunStart - 命令成功开始了触发
autocmd User AsyncRunStop  - 命令结束时触发
```

注意，`AsyncRunPre` 一般都会被触发，但是 `AsyncRunStart` 和 `AsyncRunStop` 只会在任务成功开始的情况下才会被触发。

比如当前任务未结束，AsyncRun 会失败。在这种情况下，`AsyncRunPre` 会被触发，但是 `AsyncRunStart` 和 `AsyncRunStop` 无法触发。

### 项目根目录

Vim 缺乏项目管理，然而日常编辑的一个个文件，大部分都会从属于某个项目。如果你缺乏项目相关的信息，你就很难针对项目做点什么事情。参考 CtrlP 插件的设计，AsyncRun 使用 `root markers` 的机制来识别项目的根路径，当前文件所在的项目目录是该文件的最近一级包含 `root marker` 的父目录（默认为`.git`, `.svn`, `.root` 以及 `.project`），如果递归到根目录还没找到标识文件，那么会用当前文件所在目录代替。

在命令或者 `-cwd=?` 参数中使用 `<root>` 或者 `$(VIM_ROOT)` 来表示当前文件的 **项目根目录**，比如：

```VimL
:AsyncRun make
:AsyncRun -cwd=<root> make
```

第一个 `make` 命令会在 vim 的当前路径（可以用 `:pwd` 命令查看）下面执行，而第二个 `make` 命令会在当前的项目根目录下面执行。当你要对整个项目做点什么的时候（比如 make/grep），这个特性会非常有用。

更多信息参考：[Project Root](https://github.com/skywind3000/asyncrun.vim/wiki/Project-Root).

### 运行模式

AsyncRun 可以用 `-mode=?` 参数指定运行模式，不指定的话，将会用默认模式，在后台运行，并将输出实时显示到 quickfix 窗口，然而你还可以用 `-mode=?` 设置成下面几种运行模式：

| 模式 | 说明 |
|--|--|
| async | 默认模式，在后台运行命令，并将输出实时显示到 quickfix 窗口。 |
| bang | 用 Vim 传统的 `:!` 命令执行。 |
| term | 使用一个可复用的内部终端执行命令。 |
| os | 目前只支持 Windows，打开一个新的 cmd.exe 窗口执行命令。|

更多信息，见 [这里](https://github.com/skywind3000/asyncrun.vim/wiki/Specify-how-to-run-your-command).

### 内置终端

如果 `AsyncRun` 命令后加了一个 `-mode=term` 参数，那么将会在 Vim 内置终端中运行命令，在该模式下，还可以提供一个 `-pos=?` 来指定终端窗口的位置：

- `-pos=tab`: 在新的 tab 中打开终端。
- `-pos=curwin`: 在当前窗口打开终端。
- `-pos=top`: 在上方打开终端。
- `-pos=bottom`: 在下方打开终端。
- `-pos=left`: 在左边打开终端。
- `-pos=right`: 在右边打开终端。

例子:

```VimL
:AsyncRun -mode=term -pos=tab python "$(VIM_FILEPATH)"
:AsyncRun -mode=term -pos=bottom -rows=10 python "$(VIM_FILEPATH)"
:AsyncRun -mode=term -pos=right -cols=80 python "$(VIM_FILEPATH)"
:AsyncRun -mode=term -pos=curwin python "$(VIM_FILEPATH)"
:AsyncRun -mode=term -pos=curwin -hidden python "$(VIM_FILEPATH)"
```

当你用 split 窗口打开终端时 (`-pos` 为 `top`, `bottom`, `left` 和 `right` 的其中之一)，AsyncRun 会先检查是否有之前已经运行结束的终端窗口，有的话会复用，没有的话，才会新建一个 split。

### Quickfix window

默认运行模式下，AsyncRun 会在 quickfix 窗口中显示任务的输出，所以如果你事先没有用 `:copen {height}` 打开 quickfix 窗口，你将会看不到任何内容。方便起见，引入了一个 `g:asyncrun_open` 的全局配置，如果设置成非零：

    :let g:asyncrun_open = 8

那么在运行命令前，会自动按照 8 行的高度打开 quickfix 窗口。

### Range 支持

AsyncRun 可以指定一个当前 buffer 的文本范围，用作命令的 stdin 输入，比如：

```VimL
:%AsyncRun cat
```

那么当前文件的整个内容将会作为命令 `cat` 的标准输入传递过去，这个命令运行后，quickfix 窗口内将会显示当前文件的所有内容。


```VimL
:10,20AsyncRun python
```

使用当前 buffer 的第 10 行到 20 行的内容作为命令 python 的标准输入，可以用来执行一小段 python 代码，并在 quickfix 展现结果。

```VimL
:'<,'>AsyncRun -raw perl
```

选中区域的文本 (行模式) 作为标准输入。


### 运行需求

Vim 7.4.1829 是最低的运行版本，如果低于此版本，运行模式将会从 `async` 衰退回 `sync`。NeoVim 0.1.4 是最低的 nvim 版本。

推荐使用 vim 8.0 及以后的版本。

### 同 fugitive 协作

asyncrun.vim 可以同 `vim-fugitive` 协作，为 fugitive 提供异步支持，具体见 [here](https://github.com/skywind3000/asyncrun.vim/wiki/Cooperate-with-famous-plugins#fugitive).

![](https://raw.githubusercontent.com/skywind3000/asyncrun.vim/master/doc/cooperate_with_fugitive.gif)


## 语言参考

- [Better way for C/C++ developing with AsyncRun](https://github.com/skywind3000/asyncrun.vim/wiki/Better-way-for-C-and-Cpp-development-in-Vim-8)

## 更多话题

- [Additional examples (background ctags updating, pdf conversion, ...)](https://github.com/skywind3000/asyncrun.vim/wiki/Additional-Examples)
- [Notify user job finished by playing a sound](https://github.com/skywind3000/asyncrun.vim/wiki/Playing-Sound)
- [View progress in status line or vim airline](https://github.com/skywind3000/asyncrun.vim/wiki/Display-Progress-in-Status-Line-or-Airline)
- [Best practice with quickfix window](https://github.com/skywind3000/asyncrun.vim/wiki/Quickfix-Best-Practice)
- [Scroll the quickfix window only if the cursor is on the last line](https://github.com/skywind3000/asyncrun.vim/wiki/Scroll-the-quickfix-window-only-if-cursor-is-on-the-last-line)
- [Replace old ':make' command with asyncrun](https://github.com/skywind3000/asyncrun.vim/wiki/Replace-old-make-command-with-AsyncRun)
- [Quickfix encoding problem when using Chinese or Japanese](https://github.com/skywind3000/asyncrun.vim/wiki/Quickfix-encoding-problem-when-using-Chinese-or-Japanese)
- [Example for updating and adding cscope files](https://github.com/skywind3000/asyncrun.vim/wiki/Example-for-updating-and-adding-cscope)
- [The project root directory of the current file](https://github.com/skywind3000/asyncrun.vim/wiki/Project-Root)
- [Specify how to run your command](https://github.com/skywind3000/asyncrun.vim/wiki/Specify-how-to-run-your-command)

Don't forget to read the [Frequently Asked Questions](https://github.com/skywind3000/asyncrun.vim/wiki/FAQ).

## 插件协作

| Name | Description |
|------|-------------|
| [vim-fugitive](https://github.com/skywind3000/asyncrun.vim/wiki/Cooperate-with-famous-plugins#fugitive)  | perfect cooperation, asyncrun gets Gfetch/Gpush running in background |
| [errormarker](https://github.com/skywind3000/asyncrun.vim/wiki/Cooperate-with-famous-plugins) | perfect cooperation, errormarker will display the signs on the error or warning lines |
| [airline](https://github.com/skywind3000/asyncrun.vim/wiki/Cooperate-with-famous-plugins#vim-airline) | very well, airline will display status of background jobs |
| [sprint](https://github.com/pedsm/sprint) | nice plugin who uses asyncrun to provide an IDE's run button to runs your code |
| [netrw](https://github.com/skywind3000/asyncrun.vim/wiki/Get-netrw-using-asyncrun-to-save-remote-files) | netrw can save remote files on background now. Experimental, take your own risk | 


See: [Cooperate with famous plugins](https://github.com/skywind3000/asyncrun.vim/wiki/Cooperate-with-famous-plugins)

## Credits

Trying best to provide the most simply and convenience experience in the asynchronous-jobs. 

Author: skywind3000
Please vote it if you like it: 
http://www.vim.org/scripts/script.php?script_id=5431
