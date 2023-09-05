---
title: 复古EMail配置
date: 2023-04-10
tags: [email,config]
categories: [dev]
author: Phillips Horselover
---
## Intro

Thunderbird 曾经是我的主力邮件客户端, 它支持非常多的功能, 包括 RSS ,
多种即时聊天( Matrix 也包括在其中), 还能与网络日历同步.
但是大部分功能我都用不上: 我只是需要一个简单的邮件客户端,
让我可以流畅地处理工作就行了. 这时Thunderbird丰富的功能反而成了累赘.

最后选择了 [Mutt](https://mutt.org/) 作为我的新邮件客户端(事实上, 在本篇
Post 写作的三个月前我就在使用 `Mutt` 了). 这是一个最早发行于 199X
年的基于文本的邮件客户端, 截止本篇 Post 写成, 已经更新到了 2.2.10 版本.
支持的特性包括:

- 颜色支持
- Threading
- MIME Support
- IMAP / POP3 支持
- 高可定制

这里所写的特性并不是 `Mutt` 的全部. 因为它的高可定制,
我们可以把它变成我们想要它成为的任何模样, 正如其开发者所说:

*"All mail clients suck. This one just sucks less"*

## 关于电子邮件的简单介绍

### 当我们使用电子邮件时, 到底发生了什么?

假设我们的邮件地址是 `a@a.com` , 配置的 SMTP 服务器是 `smtp.a.com` ,
接收者的邮件地址是 `b@b.com` .

- 使用SMTP协议, 将邮件推送到我们的 SMTP 服务器上.
- `smtp.a.com` 的 MTA 接收到邮件后
  - 找到收件人的邮件服务器地址 `b.com`
  - 向 DNS 服务器查询 `b.com` 的 MX 记录, 得到 `mail.b.com`
  - 向 DNS 服务器查询 `mail.b.com` 的 IP 地址, 得到 `1.2.3.4`
  - 向该 IP 地址使用 SMTP 协议发送邮件
- 接收者的邮件服务器收到邮件, 确认收件人在当前服务器上后, 使用 MDA
  投放邮件(可能是对应用户的文件夹, 也有可能是数据库)

这个时候, 邮件仍然在服务器上, 接收者可以登录邮件服务器查看邮件,
比如ssh登录查看, 或者登录邮件服务商提供的网页接口. 当然,
我们还有另外一个选择: 将邮件下载下来. 有两个协议可以实现这个目的, IMAP
和 POP3 . 下面是两个协议间的简单对比:

| POP3                                                         | IMAP                               |
|--------------------------------------------------------------|------------------------------------|
| 同时只能有一个客户端访问                                     | 可以有多个客户端访问               |
| 单向传输                                                     | 双向传输                           |
| 快                                                           | 相对 POP3 更慢                     |
| 删除模式和保留模式. 区别在于收取邮件后是否删除服务器上的备份 | 收取后还能保留副本                 |
| 一次需要收取所有邮件                                         | 在下载前可以查看邮件头决定是否下载 |

### 邮件客户端的架构

我们的邮件客户端的架构如下:

- MUA: mutt , 用于查看邮件. 虽然 mutt 拥有 IMAP 和 POP3 支持,
  但是使用起来不算方便, 我们将要用其他的软件来实现这些功能.
- MRA: fetchmail , 用于从服务器收取邮件, 实现 IMAP 和 POP3 的功能.
  fetchmail 也可以看作一个特殊的 MTA : 它将邮件收取后, 也会将邮件输出给
  MDA 进行投送. 由大名鼎鼎的 Raymond 开发.
- MDA: procmail , 投送邮件. 根据邮件的内容投送到对应的地方.
- smtp: msmtp , 发送邮件.

处理邮件的流程就是由 fetchmail 从邮件服务器收取邮件, 将邮件通过 stdin
输入给 procmail 投递到对应的文件夹. 使用 mutt 访问邮件文件夹.
需要发送邮件时, 通过 mutt 调用编辑器 (比如 neovim / vim / nano)
进行邮件内容编辑, 再调用 msmtp 发送到 SMTP 服务器.
在发送邮件和阅读邮件时, 也可以使用 gpg 来对邮件进行签名加密 / 验证解密.

## fetchmail & procmail

fetchmail 负责收取邮件, 相关的配置在 `$HOME/.fetchmailrc` ,
我们可以在配置文件中指定我们需要收取的邮件地址.

    poll pop.AAA.com protocol POP3 user "me@AAA.com" password "123"
    poll pop.BBB.com protocol POP3 user "me" there with password "123" is falko here fetchall
    poll pop.CCC.com protocol POP3 user "me" there with password "123" is till here keep
    poll pop.DDD.com
      protocol POP3
      user "me"
      password "123"

    # 全局选项
    mimedecode
    mda "/usr/local/bin/procmail"

poll \<ADDRESS\>  
指定一个需要收取的账户

protocol POP3 \| IMAP  
指定使用的协议

user \<USERNAME\>  
指定用户名, 或者邮件地址

password \<PASSWORD\>  
账户密码

\<OPTION\> ..  
包括 fetchall, nofetchall, keep, nokeep 等等

mimedecode  
自动解码 mime

mda \<PATH\>  
指定 mda 程序的路径, 这里我们使用 procmail

``` bash
# 收取未读邮件
$ fetchmail

# 收取所有邮件
$ fetchmail -a

# 收取未读邮件, 但是不删除服务器上已经收取的邮件
$ fetchmail -k

# 设置超时时间(s)
$ fetchmail -t 60
```

每一次执行上述命令都会收取所有的邮件并交由 mda 处理,
并不会在后台自动收件, 所以我们需要手动执行命令 / Cron 定期任务.

procmail 的配置在 `$HOME/.procmailrc` ,
其中定义了收取到的邮件的处理规则, 使用正则表达式定义.

    MAILDIR=$HOME/Mail   #邮件存储地址
    DEFAULT=$MAILDIR/inbox   #默认：收件箱
    VERBOSE=off
    LOGFILE=/tmp/procmaillog

    # 某个垃圾邮件规则
    :0
    * ^From: spam@aaa\.com
    /dev/null    #垃圾文件的存储位置

    # 其它所有都存到收件箱中
    :0:
    inbox/

完成配置后, 我们使用 fetchmail
收件后就能在收件箱路径下找到我们的新邮件了.

## msmtp

msmtp 的配置就很简单了,
基本上就是将平常我们使用邮件客户端时需要填写的配置内容以文本的形式填写到配置文件中.
msmtp 的配置文件路径为 `$HOME/.msmtprc` .

    acount default
    auth login
    host smtp.XXX.com
    port 587
    from ME@XXX.com
    user ME
    password passwd
    tls on
    tls_certcheck off

    logfile /tmp/msmtp.logrc

如果需要多账户, 我们只需要定义多个 account \<ACCOUNT\> , 在调用 msmtp
时添加参数 ~ -a \<ACCOUNT\> ~ 即可.

## mutt

    # 通用设定
    set use_from=yes
    set envelope_from=yes
    set move=yes    #移动已读邮件
    set include #回复的时候调用原文
    set charset="utf-8"

    # 发送者账号
    set realname="Phillips Horselover"
    set from="phillips@navihx.top"

    # 分类邮箱
    set mbox_type = Maildir #Mail box type
    set folder = "$HOME/Mail"
    set spoolfile = "$HOME/Mail/inbox" #INBOX
    set mbox="$HOME/Mail/seen"  #Seen box
    set record="$HOME/Mail/sent"  #Sent box
    set postponed="$HOME/Mail/draft"  #Draft box

    set editor="vim -nw"
    set sendmail="/usr/local/bin/msmtp"

以上的配置对于只需要一个账户的使用者已经足够了.
但是如果你需要使用很多个邮件账户, 每个账户中都有不同的配置, 可以使用
mutt 提供的 folder hooker 功能: 当 mutt 进入某个邮件文件夹时,
重新读入一段配置来覆盖原本的配置.

    folder-hook 'INBOX' 'source ~/.config/mutt/accounts/default'
    folder-hook 'gmail' 'source ~/.config/mutt/accounts/gmail'
    folder-hook 'personal' 'source ~/.config/mutt/accounts/personal'
    folder-hook 'phillips' 'source ~/.config/mutt/accounts/phillips'

## 其他

### 使用 GPG 加密 / 签名邮件

在 muttrc 中添加 `source $HOME/.mutt/gpg.rc` , 在对应文件中写入 gpg
的设置. 示例配置在 `/usr/share/doc/mutt/smaples/gpg.rc` .

### 保护你的密码

我们在上面的配置中全部写了明文的密码, 这是非常不安全的,
即使我们可以将配置文件的权限设置为 `700` 或者更低. 我们依然可以使用 gpg
来对我们的密码进行加密.

msmtp 可以使用 passwordeval \<COMMAND\> 代替 password \<PASSWORD\> ,
它执行一条指令, 将指令的输出作为密码. 我们可以这样写
`passwordeval  "gpg --quiet --for-your-eyes-only --no-tty --decrypt $HOME/mail/.msmtp-credentials.gpg"`
.

mutt 可以将我们密码的设置放到一个单独文件中, 对其加密, 然后在 mutt
配置中先对其解密在执行. 比如我们这样设置了密码:

    # .my-passwordrc
    set my_pass = "password"

我们对其加密得到一个新文件 .my-passwordrc.gpg . 在 mutt 配置文件中新增:

    source "gpg -dq $HOME/.my-passwordrc.gpg |"
