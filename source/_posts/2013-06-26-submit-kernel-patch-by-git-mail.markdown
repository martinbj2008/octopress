---
layout: post
title: "How to submit kernel patch by git-mail"
date: 2013-06-26 00:00
comments: true
categories: [git]
tags: [git]
---

```
git send-email --annotate  --subject-prefix="PATCH v2 net-next" \
  --in-reply-to=The_previous_email_message_id_to_be_replied \
  --smtp-debug --to MAINTENACE_person_email --cc CCLIST HEAD^^
```

1. `—annotate`: 临时增加一些注释到邮件里。添加方法跟git commit --amend 类似。 只是这些注释要添加在---的后面。例如，
2. Cc: .....
3. Signed-off-by: ....

```
37 ---
<=== 使用annotate，在这里添加临时注释。
38  arch/x86/kernel/cpu/mtrr/cleanup.c | 8 ++++----
39  1 file changed, 4 insertions(+), 4 deletions(-)
这样的注释不会被保留到最终的git 仓库里。所以我把它们成为临时注释。
```

4. `—subject-prefix`: 邮件标题前面中括号里的内容。 要是邮件不小心发错了，需要重发时。要带有”RESEND”字样。 “PATCH vX YYYY”, X标识版本号。第一个版本时可以省略x及前面的v. YYYY 标识具体的那个模块。对网络模块来说，重要的问题发送net， 不重要的发送到net-next。

5. `—in-reply-to` 主要用在邮件讨论上。 使邮件保持在一个thread(?)里。

6. `HEAD^^`: means two patch to be sent. 当发送一系列的patch时有用。

关于维护人。 有两种方式可以得到， kerel源文件根目录下的MAINTAINERS 或者./scripts/get_maintainer.pl

```
$ cat ~/.gitconfig
[user]
  name = XXXX
  email = martinbj2008@gmail.XXXXX
[sendemail]
  smtpencryption = tls
  smtpserver = smtp.gmail.com
  smtpuser = martinbj2008@gmail.XXXX
  smtpserverport = 587
  confirm = always
  smtppass=XXXXXXXXXXXX
  from=martinbj2008@XXXX
junwei@junwei:~/tmp/linux$
```
