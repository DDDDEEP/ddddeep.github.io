---
layout: post
title:      "整理 Mac 下使用的软件与设置"
subtitle:   ""
date:       2020-05-23 00:00:00
author:     "DEEP"
header-img: "img/home-bg.jpg"
catalog: true
tags:
    -   日常
---

最近 1 年换了 3 台 Mac（准备又要弄第 4 台），为了以后配置环境更顺利，这里整理一下个人常用的软件与配置。

个人十分赞同一个思路：「不要迎合软件的功能，而是利用软件解决需求」。

需求越清晰，就越容易找到精准、不冗杂的软件方案。

一般日常工作有 **双手打字**、**（左手键盘）右手鼠标** 两种姿势，右手切换的次数越少，工作体验会越好；

键盘操作的优势是可以快速盲打，在克服操作上的思维习惯后（即第一时间联想到用键盘操作），大多数情况是更快的。

鼠标操作的优势是单手操作舒适，适合长时间进行的操作（如网页滚动）。

## Alfred

### Features

其实 Alfred 的搜索就是用 Spotlight 的索引，搜索功能并非其独特优点。

个人认为 Alfred 的真正优点在于，其可以尽可能在打字姿势下实现各种需求，

个人最常用的 Features 有：

- Default Results：给常用的搜索网站配 fallback results，快速切到浏览器。
- Web Bookmarks：浏览器弄一个书签文件夹，保存常用的网站页面，并修改名字为简单的英文缩写，实现快速打开页面。
- Clipboard History：很实用的剪贴板历史功能。
- Snippets：看自身需求。
- Calculator：输入 = 功能更多。
- Dictionary：
- System：一般只有 sleep 用得上（

文件相关功能个人觉得只有 Reveal File 用得上，其它操作通过 QSpace 进行更符合使用习惯。

配置同步方面个人仍采用土方法（

### Workflows

最强大的功能，可以作为各种操作的入口，使日常操作思维更简单。

作为脚本语言的启动途径，自定义比较简单。

- [**CodeVar**](https://github.com/xudaolong/CodeVar)：根据中文生成变量名。
- [**Colors**](https://github.com/TylerEich/Alfred-Extras)：颜色相关。
- [**Currency Convert**](https://github.com/jin5354/alfred3-workflow-CurrencyConvert)：汇率。
- **Dash**：与文档工具 Dash 联动。
- [**Datetime Format Converter**](https://github.com/ACBingo/alfred-datetime-format-converter)：时间相关。
- [**Emoji**](https://github.com/carlosgaldino/alfred-emoji-workflow)：Emoji。
- [**Encode/Decode**](https://github.com/willfarrell/alfred-encode-decode-workflow)：编码/解码。
- [**Hash**](https://github.com/willfarrell/alfred-hash-workflow)：哈希。
- **IP Address**：IP。
- [**Kill Process**](https://github.com/nathangreenstein/alfred-process-killer)：杀进程。
- [**NSC**](https://github.com/obstschale/NSC)：进制相关。
- [**TimeZones**](https://github.com/jaroslawhartman/TimeZones-Alfred)：时区。

## 软件

个人一般配置全局激活软件的快捷键为 ⌥ 或 ⌥ + ⌘，以尽可能避免与各种快捷键冲突。

### App Store

- **QSpace**：很好用的文件管理软件，除了某些操作权限问题外，可完全替代访达，根据使用习惯调下设置和工具栏，操作卡的时候重启软件。
- **Xnip**：精简的截图与取色工具。（但个人发现有时色值不对，留个心眼）
- **Manico**：应用启动和切换工具，轻量精简，可通过左手解决鼠标切应用麻烦的问题。
- **EasyFind**：搜索文件工具，个人实际使用场景少，不是十分熟悉。
- **Moom**： 窗口布局软件，自定义设置丰富。
- **OneDrive**：跟随 Windows 使用惯性。
- **WPS Office**：Word for Mac 个人体验不好，因其功能布局与 Windows 有一定差别。
- **Xcode**
- **QQ**
- **微信**
- **网易云音乐**
- **QQ 音乐**
- ~~**Magnet**~~：窗口布局软件，轻量简单，但实际不常用。

### DMG

- **Scroll Reverser**：鼠标滚动反转工具。

- **eZip**：归档工具。
- **Charles**：网络抓包。
- **IINA**：视频软件。
- **Dash**：API 文档。
- **Fork**：Git 工具。
- **Lepton**：Github Gist 工具。
- **Mounty**：NTFS 读写。
- **Postman**：API 请求工具。
- **Typora**：Markdown 编辑器，但比较卡。
- **HandBrake**：视频压缩。
- **Chrome**：
- **VSCode**：
- **Iterm2**：
- **PDFpen / Adobe Reader / PDFelement**
- **Bartender 3 / Dozer** 菜单栏图标管理工具。
- ~~**Clipy**~~： 如果不用 Alfred，这个是比较好的一个剪贴板历史软件。

### brew cask

```bash
brew cask install scroll-reverser visual-studio-code ezip iina fork postman typora iterm2
```

## Chrome

Chrome 可以根据账户同步扩展程序。

- **Adblock Plus**：广告屏蔽。

- **crxMouse**：鼠标手势，单手网上冲浪利器，使用后右键菜单需双击。

    个人设置：↑新建标签，↓关闭标签，↔︎切换左/右标签，→↕︎滚动到顶部/底部，←↑重新打开标签，↑↓/↓↑刷新。
- **EditThisCookie**：Cookie 编辑器 ~~(eh 门票)~~。
- **Imagus**：图片预览。
- **Infinity**：标签页配置。
- **Octotree**：Github 文件目录。
- **Set Character Encoding**：页面编码。
- **Tampermonkey**：脚本。
- **Vimium**：
- **划词翻译**：
- **保护眼睛**：

## 命令行

### 初始

安装 brew：

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install.sh)"
```

获取 cask：
```bash
xcode-select --install
brew tap caskroom/cask
```

代理：

```bash
# ~/.bash_profile
alias proxy='export all_proxy=socks5://127.0.0.1:1080'
alias unproxy='unset all_proxy'

# .gitconfig
[http]
	proxy = socks5://127.0.0.1:1080
[https]
	proxy = socks5://127.0.0.1:1080

# 测试：
curl cip.cc
```

Python 3 & Node && Mosh
```bash
brew install python
brew install git
brew install -g node
brew install mosh
brew install ruby
npm install -g typescript
```

Git 多账号，编辑 ~/.ssh/config：
```bash
#github
       Host self  #别名
       HostName github.com
       IdentityFile ~/.ssh/id_rsa_github
       User selfname

#gitlab
       Host work
       HostName gitlab.com
       IdentityFile ~/.ssh/id_rsa
       User workname
```

```bash
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.ci commit
git config --global alias.st status
```

实际命令例如：`git clone git@self:NAME/REPO`。

也可也修改现有仓库的 `.git/config`:
```bash
[user]
    email = self@email.com
    name = selfname
[remote "origin"]
	url = git@self:NAME/REPO
```

### 终端

oh-my-zsh：

```bash
sh -c "$(curl -fsSL https://raw.github.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

powerlevel10k:

```bash
git clone --depth=1 https://github.com/romkatv/powerlevel10k.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/themes/powerlevel10k

设置 ZSH_THEME="powerlevel10k/powerlevel10k" in ~/.zshrc

source ~/.zshrc，进行系列配置

# 配置过程会询问安装 Meslo Nerd 字体
p10k configure
```

iterm 2 主题 Dracula：
```bash
cd /Users/{$usr}/Downloads
git clone https://github.com/dracula/iterm.git

Profiles -> Colors 导入
```

iterm 2 集成 Shell Integration：
```bash
curl -L https://iterm2.com/shell_integration/zsh -o ~/.iterm2_shell_integration.zsh

# ~/.iterm2_shell_integration.zsh 下面行间增加一个空行
ITERM_SHELL_INTEGRATION_INSTALLED=Yes

ITERM2_SHOULD_DECORATE_PROMPT="1"

# ~/.zshrc 行尾加入
source ~/.iterm2_shell_integration.zsh
```

安装 zsh 插件 zsh-autosuggestions、zsh-syntax-highlighting，

安装 fasd、fzf、ripgrep：

```bash
brew install fzf fasd ripgrep

git clone https://github.com/zsh-users/zsh-autosuggestions ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions

git clone https://github.com/zsh-users/zsh-syntax-highlighting.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting

# ~/.zshrc 中间修改：
export FZF_BASE=/usr/local/Cellar/fzf/0.21.1/shell
plugins=(
    git
    zsh-syntax-highlighting
    zsh-autosuggestions
    fasd
    fzf
    ripgrep
)
eval "$(fasd --init auto)"

...

# Example aliases
# alias zshconfig="mate ~/.zshrc"
# alias ohmyzsh="mate ~/.oh-my-zsh"

# 因为其它不少工作的文章都使用 .bash_profile，为了方便还是额外弄一个
source ~/.bash_profile
```

alias：

```bash
# ~/.bash_profile
# 代理
alias proxy='export all_proxy=socks5://127.0.0.1:1081'
alias unproxy='unset all_proxy'

# fasd
unalias a
unalias s
unalias d
unalias f
unalias sd
unalias sf
unalias z
unalias zz
alias j='fasd_cd -d'
alias v='fasd -f -e vim'
alias c='fasd -f -e code'

# VSCode 权限
alias forcode='sudo chown -R admin '

export OBJC_DISABLE_INITIALIZE_FORK_SAFETY=YES
export PATH="/usr/local/opt/ruby/bin:$PATH"
```

最后执行：

```bash
source ~/.zshrc
```

同时需要修改光标按照单词快速移动的设置

### 最终 Feature
- TAB 补全命令，双击 TAB 目录浏览。
- 鼠标双击匹配单词、三击匹配整行、四击智能匹配，且自动复制。
- ⌘ + F 可搜索，关键词后按 TAB 可智能匹配。
- ⌘ + ⇧ + H 剪贴板历史，⌘ + ⇧ + B 历史终端快照，⌘ + ⇧ + ; 命令历史。
- 按住 ⌘ 可智能点击、拖动文本。
- 可直接输入目录，不用 cd。
- ⌘ + 方向切换 Tab，⌘ + ⌥ + 方向切换 Pane。
- fasd：j 跳转近期目录，v vim 近期文件。

## IDE

根据 80/20 法则，过度优化写代码的效率并不能大幅度增强个人能力，有时间还是多学些东西，少折腾软件配置，配置到够用即可。

### 快捷键

关于快捷键前缀，个人使用 (⌥) Manico 切换应用窗口、Moom 调整布局， (⌥ + ⌘) 激活全局应用 。

而 (⇧ + ⌘) 一般会被应用占用，因此一般配置于应用自带功能的快捷键；同时个人一般用 (⇧ + ⌃) 作为应用扩展功能的快捷键。

个人不太会用 Vim，下面列出的是个人认为必要的编辑功能列表，不同 IDE 上手后应将下面的功能快捷键改为自己习惯、顺手的。

编辑与选择：

| 功能                                                         | 个人偏好快捷键                       |
| ------------------------------------------------------------ | ------------------------------------ |
| 光标移至行首/行尾，按单词，第一行/最后一行，跳转括号         | ⌘ + ←/→，⌥ + ←/→，⌘ + ↑/↓，⌘ + M     |
| 选择行，复制行，向下复制行，向上/下插入行                    | ⌘ + L，⌘ + C，⇧ + ⌘ + D，(⇧) + ⌘ + ⏎ |
| 剪切行，删除行，合并行，上下移动行                           | ⇧ + ⌘ + X，⌘ + X，⌘ + J，⌥ + ⌘ + ↑/↓ |
| 向上/下新增一个光标，通过鼠标增加光标（没找到修改方法），光标撤销 | ⇧ + ⌃ + ↑/↓，⌥，⌘ + U                |
| 下一个查找匹配添加到选择，跳过该匹配项并选择下一个匹配项，选择所有匹配 | ⌘ + D，⌘+K ⌘+D，⌘ + ⏎                |

复合功能：

| 功能                                          | 个人偏好快捷键                |
| --------------------------------------------- | ----------------------------- |
| 裁剪尾随空格                                  | ⌘+K ⌘+X                       |
| 行注释、块注释                                | ⌘ + /，⌥ + ⌘ + /              |
| 大小写                                        | ⌘+K ⌘+U，⌘+K ⌘+L              |
| 新建标签页，切换左/右标签，重新打开关闭的标签 | ⌘ + T，⌥ + ⌘ + ←/→，⇧ + ⌘ + T |
| 上/下一个编辑位置                             | ⌃ + ⌘ + ←/→                   |
| 智能查找，智能跳转，文档等等                  | ⇧ + ⌘+ F/O，⌘，⌥              |
| 头文件/源文件切换                             | ⌃ + ⌘ + ↑/↓                   |

### VSCode

首先在 command 里安装 code 命令行。

插件：

- Settings Sync：同步设置。
- Better Align：对齐等号。
- Bookmarks：书签。
- Bracket Pair Colorizer：括号颜色。
- Chinese：
- Debugger for Chrome：
- ESLint：
- expand-region：选择单词/括号内容。
- GitLens：Git 信息增强。
- Increment Selection：多光标数字递增。
- Markdown All in One：编辑 Markdown。
- Markdown Padding：Markdown 中英文空格。
- Python：

### Xcode

因为不知如何配置冲突键位，个人 Xcode 部分复合功能快捷键前缀为 (⇧ + ⌃ + ⌘)。

写下该文字时 Xcode 版本为 11.5。

Xcode Extension：

- TextPlus：补全行编辑功能。
- XCFormat：代码格式化。
- Soothe：等号对齐和行排序。

个人快捷键配置，在 `/Users/admin/Library/Developer/Xcode/UserData/KeyBindings` 新建文件配置文件 `My.idekeybindings`，内容见 [Gist 地址](https://gist.github.com/DDDDEEP/d4ea14fa1f89619a7e2ee25f847f43c7)。

[调试教程](https://objccn.io/issue-19-2/)
