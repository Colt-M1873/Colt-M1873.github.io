---
title: "Merkle Tree 和 Hash List用于数据完整性验证"
date: 2023-07-26T20:23:47+08:00
draft: false
ShowToc: true
TocOpen: false
---

在点对点网络中作数据传输的时候，会同时从多个机器上下载数据，而且很多机器可以认为是不稳定或者不可信的。为了校验数据的完整性，更好的办法是把大的文件分割成小的数据块（例如，把分割成2K为单位的数据块）。这样的好处是，如果小块数据在传输过程中损坏了，那么只要重新下载这一快数据就行了，不用重新下载整个文件。

## Hash List

在下载到真正数据之前，我们会先下载一个Hash列表。
对于其每个小数据块的验证，就通过对这个数据块进行hash运算，并和Hash列表中的值相比对，哪一块出现了错误就重新下载那一块。
而对于Hash List列表本身的验证，是通过对Hash List本身生成一个Hash值，称作根哈希值（Top Hash或者Root Hash），并确保从可信信源中获取这个根Hash值，这样就能通过一个可信信源的验证(仅验证根哈希)，来从多个不可信的信源中下载各部分文件。



## Merkle Tree

Hash List虽然很大程度上解决了P2P0据传输的问题，但仍有一个遗留问题是如果文件总量超级大的话，Hash List本身也很大，而Hash List方法本身就是要先下载好整个的Hash List，才能去验证每个小文件块的正确性。
而Merkel Tree由于是树形结构，Merkel Root附近的哈希值体积非常小，可以自上而下地选择下载树的一个分支并立即进行验证，不需要先把所有文件块的Hash List都下载下来。

## 总结
原理及图示参考[Merkle Tree和Hash List用于数据完整性验证](https://www.cnblogs.com/fengzhiwu/p/5524324.html)
Merken Tree的好处是按分支进行验证，并且只要获得Merkel Tree的树根后，就可以从其他的不可信的信源获取Merkle Tree，并且可以按分支获取，不必像Hash List一样全下载完再验证完整性。Merkel Tree在P2P形式的网络（例如P2P下载和加密货币）中很重要，提供了一种简洁高效低成本的防篡改手段。


## 参考

[博客园：Merkle Tree学习](https://www.cnblogs.com/fengzhiwu/p/5524324.html)

