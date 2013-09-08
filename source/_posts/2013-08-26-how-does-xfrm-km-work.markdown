---
layout: post
title: "How does xfrm km work"
date: 2013-08-26 15:19
comments: true
categories: [xfrm]
tagss: [xfrm, km] 
---

```c
 579 struct xfrm_mgr {
 580         struct list_head        list;
 581         char                    *id;
 582         int                     (*notify)(struct xfrm_state *x, const struct km_event *c);
 583         int                     (*acquire)(struct xfrm_state *x, struct xfrm_tmpl *, struct xfrm_policy *xp);
 584         struct xfrm_policy      *(*compile_policy)(struct sock *sk, int opt, u8 *data, int len, int *dir);
 585         int                     (*new_mapping)(struct xfrm_state *x, xfrm_address_t *ipaddr, __be16 sport);
 586         int                     (*notify_policy)(struct xfrm_policy *x, int dir, const struct km_event *c);
 587         int                     (*report)(struct net *net, u8 proto, struct xfrm_selector *sel, xfrm_address_t *addr);
 588         int                     (*migrate)(const struct xfrm_selector *sel,
 589                                            u8 dir, u8 type,
 590                                            const struct xfrm_migrate *m,
 591                                            int num_bundles,
 592                                            const struct xfrm_kmaddress *k);
 593 };
```

```c
1630 static LIST_HEAD(xfrm_km_list);
```

```c
1807 int xfrm_register_km(struct xfrm_mgr *km)
1808 {               
1809         spin_lock_bh(&xfrm_km_lock);
1810         list_add_tail_rcu(&km->list, &xfrm_km_list);
1811         spin_unlock_bh(&xfrm_km_lock);
1812         return 0;
1813 }               
```

```c
1816 int xfrm_unregister_km(struct xfrm_mgr *km)
1817 {               
1818         spin_lock_bh(&xfrm_km_lock);
1819         list_del_rcu(&km->list);
1820         spin_unlock_bh(&xfrm_km_lock);
1821         synchronize_rcu();
1822         return 0;
1823 }
```

```c
1643 void km_state_notify(struct xfrm_state *x, const struct km_event *c)
1644 { 
1645         struct xfrm_mgr *km;       
1646         rcu_read_lock();           
1647         list_for_each_entry_rcu(km, &xfrm_km_list, list)
1648                 if (km->notify)    
1649                         km->notify(x, c);                        
1650         rcu_read_unlock();         
1651 } 
```

```c
1656 void km_state_expired(struct xfrm_state *x, int hard, u32 portid)
1657 { 
1658         struct net *net = xs_net(x);
1659         struct km_event c;
1660   
1661         c.data.hard = hard;        
1662         c.portid = portid;         
1663         c.event = XFRM_MSG_EXPIRE; 
1664         km_state_notify(x, &c);    
1665   
1666         if (hard)
1667                 wake_up(&net->xfrm.km_waitq);            
1668 } 
1669   
1670 EXPORT_SYMBOL(km_state_expired);
```

```c
1632 void km_policy_notify(struct xfrm_policy *xp, int dir, const struct km_event *c)
1633 {
1634         struct xfrm_mgr *km;
1635 
1636         rcu_read_lock();
1637         list_for_each_entry_rcu(km, &xfrm_km_list, list)
1638                 if (km->notify_policy)
1639                         km->notify_policy(xp, dir, c);
1640         rcu_read_unlock();
1641 }
```

```c
1632 void km_policy_notify(struct xfrm_policy *xp, int dir, const struct km_event *c)
1633 {
1634         struct xfrm_mgr *km;
1635 
1636         rcu_read_lock();
1637         list_for_each_entry_rcu(km, &xfrm_km_list, list)
1638                 if (km->notify_policy)
1639                         km->notify_policy(xp, dir, c);
1640         rcu_read_unlock();
1641 }
```
```c
1708 void km_policy_expired(struct xfrm_policy *pol, int dir, int hard, u32 portid)
1709 {
1710         struct net *net = xp_net(pol);
1711         struct km_event c;
1712 
1713         c.data.hard = hard;
1714         c.portid = portid;
1715         c.event = XFRM_MSG_POLEXPIRE;
1716         km_policy_notify(pol, dir, &c);
1717 
1718         if (hard)
1719                 wake_up(&net->xfrm.km_waitq);
1720 }
1721 EXPORT_SYMBOL(km_policy_expired);
```

```c
3031 static int __init xfrm_user_init(void)
3032 { 
...
3040         rv = xfrm_register_km(&netlink_mgr);     
...
3044 } 
```
```c
2989 static struct xfrm_mgr netlink_mgr = {
2990         .id             = "netlink",
2991         .notify         = xfrm_send_state_notify,
2992         .acquire        = xfrm_send_acquire,
2993         .compile_policy = xfrm_compile_policy,
2994         .notify_policy  = xfrm_send_policy_notify,
2995         .report         = xfrm_send_report,
2996         .migrate        = xfrm_send_migrate,
2997         .new_mapping    = xfrm_send_mapping,
2998 };
```



```c
2577 static int xfrm_send_state_notify(struct xfrm_state *x, const struct km_event *c)
2578 {
2579 
2580         switch (c->event) {
2581         case XFRM_MSG_EXPIRE:
2582                 return xfrm_exp_state_notify(x, c);
2583         case XFRM_MSG_NEWAE:
2584                 return xfrm_aevent_state_notify(x, c);
2585         case XFRM_MSG_DELSA:
2586         case XFRM_MSG_UPDSA:
2587         case XFRM_MSG_NEWSA:
2588                 return xfrm_notify_sa(x, c);
2589         case XFRM_MSG_FLUSHSA:
2590                 return xfrm_notify_sa_flush(c);
2591         default:
2592                 printk(KERN_NOTICE "xfrm_user: Unknown SA event %d\n",
2593                        c->event);
2594                 break;
2595         }
2596 
2597         return 0;
2598 
2599 }
```

```c
2428 static int xfrm_exp_state_notify(struct xfrm_state *x, const struct km_event *c)
2429 {
2430         struct net *net = xs_net(x);
2431         struct sk_buff *skb;
2432         
2433         skb = nlmsg_new(xfrm_expire_msgsize(), GFP_ATOMIC);
2434         if (skb == NULL)
2435                 return -ENOMEM;
2436         
2437         if (build_expire(skb, x, c) < 0) {
2438                 kfree_skb(skb);
2439                 return -EMSGSIZE;
2440         }
2441                 
2442         return nlmsg_multicast(net->xfrm.nlsk, skb, 0, XFRMNLGRP_EXPIRE, GFP_ATOMIC);
2443 }  
```

### todo

1. what is net->xfrm.km_waitq and where it is used.
