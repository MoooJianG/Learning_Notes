Git入门我选择观看这个教程，以下是我的笔记，更多的是对这个教程的翻译，少部分的理解，仅作为个人的学习笔记，如有不足还望指正。
[Version Control](https://missing.csail.mit.edu/2020/version-control/)

# 概述
版本控制系统（Version control systems(VCSs))是追踪源代码（其他文件或者文件集）更改的工具。这些工具有助于维护变更历史，有助于团队协作。

VCSs追踪文件夹及其内容的变化，将这些变化记录为一系列的快照[^1]，
其中每个快照包含了顶层目录中文件和文件夹的完整状态。

VCSs还维护元数据，比如谁创建了什么快照，与每个快照相关的信息等等。

VCSs可以使用户查看项目过去的快照（版本），记录为什么做了某些更改，且能解决并行开发的问题

Git的底层设计是很优雅的。这是一个自底向上学习Git教程。从Git的数据模型开始到命令行页面（command-line interface)，一旦我们了解了Git的数据模型，我们就能更好的理解这些命令是如何操作底层数据的。

[^1]:快照：这个术语常常用来指代系统或数据的某个特定时刻的状态的副本或拷贝

# Git的数据模型 Git's data model
Git支持许多版本控制系统的优秀特性，比如维护历史记录，支持分支，支持协作等。
## 快照 Snapshots
Git将文件或者文件夹建模成一系列的快照。在Git的定义中，一个文件就是一个”blob“，它只是一串字节。一个目录就是一个”树“（一个目录可以包含其他目录）。快照就是正在追踪的顶层树。一个树大概如下所示。
```
<root> (tree)
|
+- foo (tree)
|  |
|  + bar.text (blob, contents = "hello, world")
|
+- baz.text (blob, contents = "Learning")
```
这个顶层树包含两个元素树” foo “（blob ” bar.text “ ）和 blob ” baz.text “

## 建模历史-相关快照 Modeling history: relating snapshots
在Git中，历史记录是快照的有向无环图。一个快照来自一组父节点（1 or more）
Git将snapshot称为"commit"提交，可视化如下图
```
o <-- o <-- o <-- o
            ^
             \
              --- o <-- o
```
o代表着一个提交，箭头表示来自的关系，即箭头指向是父节点的意思。
从第三个节点开始，历史记录分出了两条并行的记录线。这两条分支可以合并成一个新的快照。
```
o <-- o <-- o <-- o <---- o
            ^            /
             \          v
              --- o <-- o
```
Git中的提交是不可改变的。但是不是说不能纠正错误，再纠正错误的时候会创建出一个新的提交，并指向这个错误的提交。
## 数据模型的伪码 Data model, as pseudocode
读伪代码有助于了解Git的数据模型结构：
``` 
// a file is a bunch of bytes
type blob = array<byte>

// a directory contains named files and directories 
type tree = map<string, tree | blob>

// a commit has parents, metadata, and the top-level tree
type commit = struct {
	parents: array<commit>
	author: string
	message: string
	snapshot: tree
}
```

这就是Git的简洁的结构实现。

## 对象即内容寻址 Objects and content-addressing
对象可以是blob，tree 或者一个提交commit
```
type object = blob | tree | commit
```
在 Git 数据存储中，所有对象都通过它们的 [SHA-1散列](https://en.wikipedia.org/wiki/SHA-1)进行内容寻址。
```
objects = map<string, object>

def store(object):
	id = sha1(object)
	objects[id] = object

def load(id):
	return objects[id]
```
blob, tree, commit都是以这种方式统一在一起的，但它们引用其他对象时，不是直接在磁盘上包含它们，而是通过散列查找来引用它们。

使用`git cat-file -p 698281bc680d1995c5f4caaf3359721a5a58d48d`该指令，可以可视化示例目录树结构：
```
100644 blob 4448adbf7ecd394f42ae135bbeed9676e894af85    baz.txt
040000 tree c68d233a33c5c06e0340e4c224f0afca87c8ce87    foo
```
`git cat-file -p 4448adbf7ecd394f42ae135bbeed9676e894af85`运行这个指令可以得到以下结果

```
git is wonderful
```

## 引用 Reference

虽然，所有的快照都可以按SHA-1散列表查询到，但是人们难以记住40个字符长度的复杂的地址。
Git解决方法是为这些地址赋上一个容易记得的名字，这些名字可以称作引用。引用是指向提交的指针，是可更改的（可以更新指向新的提交）。比如master引用就用来指向开发分支中最新的提交。
```
referenc = map<string, string>

def update_reference(name, id):
	reference[name] = id

def read_reference(name):
	return reference[name]

def load_reference(name_or_id):
	if name_or_id in references:
		return load(references[name_or_id])
	else:
		return load(name_or_id)
```
通过这种方法，Git 可以使用人类可读的名称(如“ master”)来引用历史记录中的特定快照，而不是长的十六进制字符串。

我们时常考虑，”我们在哪“--目前指针指向哪。在Git中，用到一个叫"HEAD"的特别指针来解决这个问题。

## Repositories 库
我们可以粗略的将Git库定义成对象和引用的集合。
在磁盘上，Git库存储的都是对象和引用。一切Git命令都是通过增删/修改对象或者引用来实现的（映射到对DAG[^2]的操作）

[^2]:有向无环图

无论何时输入任何命令，都要考虑命令对底层图形数据结构的操作。
相反，如果你试图对提交 DAG 进行某种特定的修改，例如“放弃未提交的修改，使‘ master’ref 点提交5d83f9e”，可能会有一个命令来做到这一点(例如，在这个例子中，git checkout master; git reset —— hard 5d83f9e)。

# Staging Area[^3]整装区

[^3]:Staging area一般翻译为暂存区，但我觉得这个翻译很容易对初学者造成混淆，它其实是扮演中介的角色，记录着我们准备要提交到本地版本库（Repository）里存放的档案的物件引用（Reference）的地方，因为是放引用而不是实体档案，所以才又被称为「索引」。Staging area的原意「【军】集结待命地区」还比较符合它的实际用途，因此「整装区」或「整备区」应该较符原意。

这是另一个与数据模型正交的概念，但它是创建提交的接口的一部分。

您可以想象一下，实现上述快照的一种方法是使用“创建快照”命令，该命令根据工作目录的当前状态创建一个新的快照。有些版本控制工具是这样工作的，但是 Git 不是。我们需要干净的快照，并且从当前状态创建快照可能并不总是理想的。例如，设想一个场景，您已经实现了两个独立的特性，并希望创建两个独立的提交，其中第一个引入了第一个特性，第二个引入了第二个特性。或者想象一个场景，在代码中添加了所有的调试 print 语句，同时还有一个 bug 修复程序; 您希望在丢弃所有 print 语句的同时提交 bug 修复程序。

Git 通过一种称为“临时区域”的机制，允许您指定下一个快照中应该包含哪些修改，从而适应了这种情况。

# Git 命令集 Git command-line interface

## Basics 基础指令
- `git help <command>` : 获取相应指令的帮助
- `git init` : 创建一个新的git库，并将数据存贮在`.git`目录下
- `git status` : 展示目前状态
- `git add <filename>` : 在整装区创建一个文件
- `git commit`: 创建一个新的提交
- `git log` : 展示历史记录
- `git log --all --graph --decorate` : 可视化历史记录为有向无环图
- `git diff <filename` : 显示对相应的整装区的更改
- `git diff <revision> <filename>` : 显示文件中快照之间的差异
- `git checkout <revision>` : 更新HEAD和当前的分支

## Branching and merging 分支与合并
- `git branch` : 显示已有分支
- `git branch <name>` : 创建一个分支
- `git checkout -b <name>` : 创建一个新的分支并指向这个分支
  - 相当于 `git branch <name>; git checkout <name>`
- `git merge <revision>` : 合并到当前分支
- `git mergetool` : 用一个花哨的工具来解决合并冲突
- `git rebase` : 用于将一个分支的提交历史移动到另一个分支上

## Remotes 远程仓库
- `git remote` : 罗列远程仓库列表
- `git remote add <name> <url>` : 添加远程仓库
- `git push <remote> <local branch>:<remote branch>` : 将对象更新进远程仓库，并更新远程仓库的引用
- `git branch --set-upstream-to=<remote>/<remote branch>` : 将本地仓库和远程仓库建立联系 ！！需查阅，未掌握
- `git fetch` : 从远程检索对象/引用
- `git pull` : 相当于`git fetch; git merge`
- `git clone` : 克隆远程仓库到本地

## Undo 撤销
- `git commit --amend` : 编辑提交的内容
- `git reset HEAD <file>` : 取消处理文件
- `git checkout -- <file>` : 放弃更改

## Advanced Git 进阶指令
- `git config` : 对git的高度自定义
- `git clone --depth=1` : 浅克隆，没有完整的版本历史记录
- `git add -p` : 部分地（patch）将文件的修改添加到暂存区, 互动。
- `git rebase -i`: 互动式
- `git blame` : 显示谁最后一次编辑了哪一行
- `git stash` : 暂时删除对工作目录的修改
- `git bisect` : 二进制搜索历史(例如递归)
- `.gitignore` : 指定要忽略的故意未跟踪的文件