# 在什么情况下不建议使用 Squash Merge

## Squash Merge

Squash 的意思是压缩，挤压。Squash Merge 即在 Merge 的时候把多个 Commit 信息进行合并。

通常在开源产品上进行 PR 合并的时候，会选择 Squash and Merge. 

## 什么情况下有问题

在内部产品迭代的过程中，在以下情况下不建议选择 Squash and Merge


1. 存在两条产品分支，A 产品分支（A/release_xxx） 以及 B 产品分支（B/release_xxx），需要频繁地将 A 分支的内容合并到 B 分支上，如果使用 Squash and Merge 合并，会导致如果存在冲突，则需要频繁解决。