---
layout: post
title: "ipv4 route fib tree reflate"
date: 2013-09-29 17:16
comments: true
categories: [route]
ctags: [kernel, ipv4, route, trie]
---


###summary
`inflate` duplicate a new node whose child array is double of orignal node.
and put original node's child into the new node.

`struct tnode *oldnode` is the orignal node.
`struct tnode *tn` is the new malloc node.

that means, `tn`'s `bits` will be doulbe of original value.

Just like the comment in the source, child nodes are divided into 3 kind nodes:

1. child node is a leaf or an internal node with skipped bits
2. an internal node with two children
3. an internal node with more than two children

```c
if (a leaf node) then
    belong case1.
else //assert: it must be a internal node).
     if (it has skipped bits) then
        belong case1
     else //assert: pos= parent's(pos+bits)
        if (two child nodes)
            belong case2
        else (more than two child nodes)
            belong case3
            (4,8,16... child nodes)
```




### `inflate`
```c
 723 static struct tnode *inflate(struct trie *t, struct tnode *tn)
 724 {
 725         struct tnode *oldtnode = tn;
 726         int olen = tnode_child_length(tn);
 727         int i;
 728 
 729         pr_debug("In inflate\n");
 730         fib_trie_debug("before inflate");
 731         {
 732                 __be32 prf = htonl(mask_pfx(tn->key, tn->pos));
 733                 debug_print("inflate: %pI4/(%d,%d)", &prf, tn->pos, tn->bits);
 734         }
 735         tn = tnode_new(oldtnode->key, oldtnode->pos, oldtnode->bits + 1);
 736 
 737         if (!tn)
 738                 return ERR_PTR(-ENOMEM);
 739 
 740         /*
 741          * Preallocate and store tnodes before the actual work so we
 742          * don't get into an inconsistent state if memory allocation
 743          * fails. In case of failure we return the oldnode and  inflate
 744          * of tnode is ignored.
 745          */
 746 
 747         for (i = 0; i < olen; i++) {
 748                 struct tnode *inode;
 749 
 750                 inode = (struct tnode *) tnode_get_child(oldtnode, i);
 751                 if (inode &&
 752                     IS_TNODE(inode) &&
 753                     inode->pos == oldtnode->pos + oldtnode->bits &&
 754                     inode->bits > 1) {
 755                         struct tnode *left, *right;
 756                         t_key m = ~0U << (KEYLENGTH - 1) >> inode->pos;
 757 
 758                         left = tnode_new(inode->key&(~m), inode->pos + 1,
 759                                          inode->bits - 1);
 760                         if (!left)
 761                                 goto nomem;
 762 
 763                         right = tnode_new(inode->key|m, inode->pos + 1,
 764                                           inode->bits - 1);
 765 
 766                         if (!right) {
 767                                 tnode_free(left);
 768                                 goto nomem;
 769                         }
 770 
 771                         put_child(tn, 2*i, (struct rt_trie_node *) left);
 772                         put_child(tn, 2*i+1, (struct rt_trie_node *) right);
 773                 }
 774         }
 775 
 776         for (i = 0; i < olen; i++) {
 777                 struct tnode *inode;
 778                 struct rt_trie_node *node = tnode_get_child(oldtnode, i);
 779                 struct tnode *left, *right;
 780                 int size, j;
 781 
 782                 /* An empty child */
 783                 if (node == NULL)
 784                         continue;
 785 
 786                 /* A leaf or an internal node with skipped bits */
 787 
 788                 if (IS_LEAF(node) || ((struct tnode *) node)->pos >
 789                    tn->pos + tn->bits - 1) {
 790                         if (tkey_extract_bits(node->key,
 791                                               oldtnode->pos + oldtnode->bits,
 792                                               1) == 0)
 793                                 put_child(tn, 2*i, node);
 794                         else
 795                                 put_child(tn, 2*i+1, node);
 796                         continue;
 797                 }
 798 
 799                 /* An internal node with two children */
 800                 inode = (struct tnode *) node;
 801 
 802                 if (inode->bits == 1) {
 803                         put_child(tn, 2*i, rtnl_dereference(inode->child[0]));
 804                         put_child(tn, 2*i+1, rtnl_dereference(inode->child[1]));
 805 
 806                         tnode_free_safe(inode);
 807                         continue;
 808                 }
 809 
 810                 /* An internal node with more than two children */
 811 
 812                 /* We will replace this node 'inode' with two new
 813                  * ones, 'left' and 'right', each with half of the
 814                  * original children. The two new nodes will have
 815                  * a position one bit further down the key and this
 816                  * means that the "significant" part of their keys
 817                  * (see the discussion near the top of this file)
 818                  * will differ by one bit, which will be "0" in
 819                  * left's key and "1" in right's key. Since we are
 820                  * moving the key position by one step, the bit that
 821                  * we are moving away from - the bit at position
 822                  * (inode->pos) - is the one that will differ between
 823                  * left and right. So... we synthesize that bit in the
 824                  * two  new keys.
 825                  * The mask 'm' below will be a single "one" bit at
 826                  * the position (inode->pos)
 827                  */
 828 
 829                 /* Use the old key, but set the new significant
 830                  *   bit to zero.
 831                  */
 832 
 833                 left = (struct tnode *) tnode_get_child(tn, 2*i);
 834                 put_child(tn, 2*i, NULL);
 835 
 836                 BUG_ON(!left);
 837 
 838                 right = (struct tnode *) tnode_get_child(tn, 2*i+1);
 839                 put_child(tn, 2*i+1, NULL);
 840 
 841                 BUG_ON(!right);
 842 
 843                 size = tnode_child_length(left);
 844                 for (j = 0; j < size; j++) {
 845                         put_child(left, j, rtnl_dereference(inode->child[j]));
 846                         put_child(right, j, rtnl_dereference(inode->child[j + size]));
 847                 }
 848                 put_child(tn, 2*i, resize(t, left)); <== NOTE: HERE resize is called !!!
 849                 put_child(tn, 2*i+1, resize(t, right));
 850 
 851                 tnode_free_safe(inode);
 852         }
 853         tnode_free_safe(oldtnode);
 854         return tn;
 855 nomem:
 856         tnode_clean_free(tn);
 857         return ERR_PTR(-ENOMEM);
 858 }
```
