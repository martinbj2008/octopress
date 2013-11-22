---
layout: post
title: "the inheriting of linux sock type"
date: 2013-11-22 17:50
comments: true
categories: 
---

###
```
133 struct tcp_sock {
134         /* inet_connection_sock has to be the first member of tcp_sock */
135         struct inet_connection_sock     inet_conn;
...
```

```
 87 struct inet_connection_sock {
 88         /* inet_sock has to be the first member! */
 89         struct inet_sock          icsk_inet;
...
```

```
37 struct inet_sock {
138         /* sk and pinet6 has to be the first two members of inet_sock */
139         struct sock             sk;
...
```


```
 288 struct sock {
 289         /*
 290          * Now struct inet_timewait_sock also uses sock_common, so please just
 291          * don't add nothing before this first member (__sk_common) --acme
 292          */
 293         struct sock_common      __sk_common;
...
```
