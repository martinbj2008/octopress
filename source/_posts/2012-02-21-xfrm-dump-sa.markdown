---
layout: post
title: "xfrm dump sa"
date: 2013-02-10 00:00
comments: true
categories: [kernel, xfrm]
tags: [kernel, xfrm, dump sa]
---

##应用层通过pfkey，dump sa的步骤：

1. 创建pfkeyl类型的一个socket， 
2. 通过sendmsg发送一个dump sa 的请求 
3. 然后通过while循环调用recvmsg

##kernel里对应的处理：

###`pfkey_sendmsg`
内核遍历存放SA的hlist。将SA的信息转化到一个或多个skb, 将这些skb添加到相应pfkey socket的sk_receive_queue链表里。
每次添加Skb时候会检查是否超出socket允许接受的最大receive字节数。
如果SA的数量比较多,则无法一次将所有的SA信息转化到skb里。

因此内核引入xfrmwalk，xfrmwalk始终处于hlist里最后一个被dump完成的sa的后面。
当sk_receive_queue满了时， xfrmwalk被保留在hlist里。
等待socket的已接受的数据被消耗掉一部分后，
内核重新激活dump操作，根据xfrmwalk继续dump剩余的SA，
直到遍历完所有的SA后，删除xfrmwalk。

```c
> err = pfkey_process(sk, skb, hdr);
> > err = pfkey_funcs[hdr->sadb_msg_type](sk, skb, hdr, ext_hdrs);
> > > pfkey_dump
> > > > pfkey_do_dump
> > > > > pfk->dump.msg_pid = hdr->sadb_msg_pid;
> > > > > pfk->dump.dump = pfkey_dump_sa;
> > > > > pfk->dump.done = pfkey_dump_sa_done;
> > > > > xfrm_state_walk_init(&pfk->dump.u.state, proto);
> > > return pfkey_do_dump(pfk);
```
```c
 279 static int pfkey_do_dump(struct pfkey_sock *pfk)
 280 {
 281         struct sadb_msg *hdr;
 282         int rc;
 283        
 284         rc = pfk->dump.dump(pfk);
 285         if (rc == -ENOBUFS)
 286                 return 0;
 287 
 288         if (pfk->dump.skb) {
 289                 if (!pfkey_can_dump(&pfk->sk))  
 290                         return 0;
 291  
 292                 hdr = (struct sadb_msg *) pfk->dump.skb->data;
 293                 hdr->sadb_msg_seq = 0;
 294                 hdr->sadb_msg_errno = rc;
 295                 pfkey_broadcast(pfk->dump.skb, GFP_ATOMIC, BROADCAST_ONE,
 296                                 &pfk->sk, sock_net(&pfk->sk));
 297                 pfk->dump.skb = NULL;
 298         }
 299 
 300         pfkey_terminate_dump(pfk);
 301         return rc;
 302 }
```
相当于pfkey_dump_sa
```c
> 针对SA 每个sa调用
> > dump_sa
> > > pfkey_broadcast_one
> > > > skb_queue_tail(&sk->sk_receive_queue, *skb2);
xfrm_state_walk(net, &pfk->dump.u.state, dump_sa, (void *) pfk);
```

```
1554 int xfrm_state_walk(struct net *net, struct xfrm_state_walk *walk,
1555                     int (*func)(struct xfrm_state *, int, void*),
1556                     void *data)
1557 {
1558         struct xfrm_state *state;
1559         struct xfrm_state_walk *x;
1560         int err = 0;             
1561 
1562         if (walk->seq != 0 && list_empty(&walk->all))
1563                 return 0;
1564 
1565         spin_lock_bh(&xfrm_state_lock);
1566         if (list_empty(&walk->all))
1567                 x = list_first_entry(&net->xfrm.state_all, struct xfrm_state_walk, all);
1568         else
1569                 x = list_entry(&walk->all, struct xfrm_state_walk, all);
1570         list_for_each_entry_from(x, &net->xfrm.state_all, all) {
1571                 if (x->state == XFRM_STATE_DEAD)
1572                         continue;
1573                 state = container_of(x, struct xfrm_state, km);
1574                 if (!xfrm_id_proto_match(state->id.proto, walk->proto))
1575                         continue;
1576                 err = func(state, walk->seq, data);
1577                 if (err) {
1578                         list_move_tail(&walk->all, &x->all);
1579                         goto out;
1580                 }
1581                 walk->seq++;
1582         }
1583         if (walk->seq == 0) {
1584                 err = -ENOENT;
1585                 goto out;
1586         }
1587         list_del_init(&walk->all);
1588 out:
1589         spin_unlock_bh(&xfrm_state_lock);
1590         return err;
1591 }
1592 EXPORT_SYMBOL(xfrm_state_walk);
```

2. recvmsg
跟普通socket的recvmsg没有太大的区别。

调用skb_recv_datagram从socket的sk_receive_queue队列里摘一个skb下来， 并将skb的数据copy到用户空间。

检查SA是否已经全部dump了。 如果还有没被dump的sa，并且socket的有比较宽裕的空间，则再次激活SA dump。 
宽裕的剩余空间的检查，保证了dump的效率， 避免激活一次SA dump只能dump很少的SA。

```c
3609 static int pfkey_recvmsg(struct kiocb *kiocb,
3610                          struct socket *sock, struct msghdr *msg, size_t len,
3611                          int flags)
3612 {
3613         struct sock *sk = sock->sk;
3614         struct pfkey_sock *pfk = pfkey_sk(sk);
3615         struct sk_buff *skb;
3616         int copied, err;
3617 
3618         err = -EINVAL;
3619         if (flags & ~(MSG_PEEK|MSG_DONTWAIT|MSG_TRUNC|MSG_CMSG_COMPAT))
3620                 goto out;
3621 
3622         msg->msg_namelen = 0;
3623         skb = skb_recv_datagram(sk, flags, flags & MSG_DONTWAIT, &err);
3624         if (skb == NULL)
3625                 goto out;
3626 
3627         copied = skb->len;
3628         if (copied > len) {
3629                 msg->msg_flags |= MSG_TRUNC;
3630                 copied = len;
3631         }
3632 
3633         skb_reset_transport_header(skb);
3634         err = skb_copy_datagram_iovec(skb, 0, msg->msg_iov, copied);
3635         if (err)
3636                 goto out_free;
3637 
3638         sock_recv_ts_and_drops(msg, sk, skb);
3639 
3640         err = (flags & MSG_TRUNC) ? skb->len : copied;
3641 
3642         if (pfk->dump.dump != NULL &&
3643             3 * atomic_read(&sk->sk_rmem_alloc) <= sk->sk_rcvbuf)
3644                 pfkey_do_dump(pfk);
3645 
3646 out_free:
3647         skb_free_datagram(sk, skb);
3648 out:
3649         return err;
3650 }
```
