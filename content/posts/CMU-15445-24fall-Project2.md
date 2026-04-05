---
title: "CMU 15445 24fall Project2"
date: 2025-03-21T15:34:02+08:00
lastMod: 2025-03-21T15:34:02+08:00
draft: false # 是否为草稿
author: ["tkk"]

categories:  ["CMU15445 24fall", C++]

tags:  [database, C++, Project]

keywords:  ["CMU15445 24fall", bustub, Project2]

description:  "CMU 15445 24fall Project2 实现及优化" # 文章描述, 与搜索优化相关
summary:  "CMU 15445 24fall Project2 实现及优化" # 文章简单描述, 会展示在主页
weight: # 输入1可以顶置文章，用来给文章展示排序，不填就默认按时间排序
slug: ""
comments: false
autoNumbering: true # 目录自动编号
hideMeta: false # 是否隐藏文章的元信息，如发布日期、作者等
mermaid: true
cover:
    image: ""
    caption: ""
    alt: ""
    relative: false
---

## 前言

断断续续写了好几周，终于是把B+树部分写完了，没想到最耽误时间的还是p1部分，p1部分遗留的错误导致我在写p2时Contention测试一直出问题，我还以为是p2哪里写错了呢，检查了很久，最终还是排查到p1了。这个故事告诉我们，不要过早优化，当然，我这也是因为原本23fall的代码在24fall有问题才想起来去修改的，情有可原（嘿嘿）。

相比于可扩展哈希，B+树层数更多，并且支持顺序读操作，相对来说效率和功能性更强，怪不得怎么没见过用可扩展哈希的数据库。在实现上面，相比于可扩展哈希，B+树的实现也不复杂，甚至感觉比可扩展哈希更简单，也可能是写的多了吧，总而言之，具体实现主要是增、删、查这三个操作，又以删最为复杂。24fall有意思的一点是有Contention测试检查你是否对B+树的并发做优化，也即是通过加大锁检查乐观锁有没有实现。

下面是最终结果，挑了个有意思的。

![24fall-Project2-LeaderBoard](/images/cmu-15445-24fall-p2/24fall-Project2-LeaderBoard.png)

## InternalPae && LeafPage

这部分很简单，就是简单实现下内部的几个函数，看函数名就知道需要实现啥功能了。我稍微修改了下InternalPage的page_id_array_的类型，因为这里应该是存子节点的page_id，当然，不改也可以存page_id

## GetValue

这部分功能比较简单，就是一个DFS搜索加上一些判断而已。

乐观锁的话就是先获取子节点，再释放父节点，保证查询操作不会被插队。

## Insert

Insert操作有多种情况，需要分别判断，我的Insert部分实现流程如下：

1. 判断有无根节点，无根节点就创建叶子节点作为根节点并插入数据然后`return`，有根节点则往下继续执行
2. 向下搜索叶节点，同时保存中间的InternalPage和搜索路径（即InternalPage搜索过程中Value的下标Index），将其放入write_set_和index_set_中
3. 在叶子节点中搜索，如果已经存在，直接`return`，如果不存在，则在搜索到的位置执行Insert操作
4. 接下来便是一波循环操作，利用write_set实现一波反向搜索（自下而上）：
   1. 如果PageSize大于MaxSize，表示需要进行分裂操作，将Page进行分裂，将分裂后的两个Page的中间节点插入父节点。
   2. 循环执行1中操作，直到PageSize <= MaxSize时执行`return`，或者write_set_为空
5. 执行到这里说明write_set_已经为空，这时候就需要根节点进行分裂了，并将新的root_page_id写入header_page

乐观锁实现：

1. 主要是修改上述`2`中的操作，在进行搜索时，如果当前节点PageSize < MaxSize，表明这个Page不会发生分裂操作。在这种情况下，可以保证之后不会对它的父节点执行操作，直接清空write_set和index_set，同时将ctx中的header_page设置为空（和其他Page类型不一样，没有放入write_set中）
2. 在执行插入操作时，如果是插入最左边，更新最左边的数据，一般会发生在插入最左节点时，这个操作是利用InternalPage中第一个key，在后续插入和删除时就不需要对write_set之外的InternalPage中节点进行修改（因为这棵树是一颗有序树）

## Iterator

Iterator操作比较简单吧，Begin()就是直接查最左节点，Begin(Key)根据key查找对应节点page_id和offset(index)，然后使用page_id和offset初始化Iterator即可。每次调用Iterator直接更新Page中的offset, offset超出page范围就根据next_page_id更新page_id即可，直到page_id变为INVALID_PAGE_ID表示Iterator结束。这里从测试来看，不需要实现并发操作，所以比较简单。

## Remove

Remove操作其实和Insert差不多，主要区别在于Remove时有一个借节点的操作，也是比较麻烦的一个点，下面是我的实现：

1. 判断有无根节点，没有根节点直接`return`，有则继续执行
2. 和Insert中一样，向下搜索叶节点，同时保存中间的InternalPage和搜索路径（即InternalPage搜索过程中Value的下标Index），将其放入write_set_和index_set_中
3. 这里也会Insert差不多，在叶子节点中搜索，如果不存在，直接`return`，如果存在，则在搜索到的位置执行Remove操作。如果当前叶子是根节点，判断删除后是否为空，为空则重置header_page中root_page_id，否则直接`return`
4. 这里便是代码比较多的地方了，也是一波循环操作：
   1. 如果PageSize大于MinSize，表明不满足B+树规则。首先向左边节点借value，如果两个节点中value数量总和不大于MaxSize，就将两个Page进行合并，否则执行借value操作；如果是第一个节点，则向右边借节点，如果两个的节点中value数加起来不大于MaxSize，进行合并；如果无左右节点，直接进入下一轮循环，执行删除父节点操作。
   2. 这里也是一样，如果PageSize >= MinSize了，则不需要进行合并了，直接`return`
5. 执行到这里，说明删除到根节点了，如果PageSize == 1，直接将根节点中第一个子节点替换根节点，修改header_page

乐观锁实现：

1. 和Insert中基本一样，修改`2`中操作，在进行搜索时，如果当前节点的PageSize > MinSize，表明这个节点不会发生合并操作。在这种情况下，可以保证之后不会对它的父节点执行操作，直接清空write_set_和index_set_，同时将ctx中的header_page设置为空。
2. 在`5`中判断是否是根节点，不判断应该也没啥问题

## 其他优化操作

1. Search使用二分查找
2. 先执行Insert操作，再判断是否违反B+树规则，这主要是为了写起来简单，将原本的InternalPageSize和LeafPageSize上限都减一了，不然出现PageSize > MaxSize时可能会爆
3. 然后就是Remove中借操作，提前移位留足空间，保证每个元素只执行一次移位操作

## 写在最后

到这里，CMU-15445算是告一段落了，之后应该是不会再动了，还是学习到不少东西，tinykv再见。
