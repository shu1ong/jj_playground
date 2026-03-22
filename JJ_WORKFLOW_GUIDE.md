# Jujutsu (`jj`) 工作流演示文档

这份文档基于当前仓库 `/home/shulong/Documents/jj_test/jj_playground` 的实际操作记录整理，目标是：

- 在一个最小仓库里走通一次完整的 `jj` 工作流
- 用真实命令解释 `jj` 的核心使用方式
- 对比 `jj` 和 `git` 在日常协作上的主要差异

如果你之前主要使用的是 `git`，可以把 `jj` 理解为：**一个更擅长“改历史、叠提交、恢复误操作”的版本控制前端**。它可以和 Git 远端协作，但本地工作体验与 Git 很不一样。

---

## 1. 当前仓库里已经演示过什么

这个仓库里已经实际执行过以下动作：

1. 从 `main` 起步创建功能提交
2. 连续创建两层 stacked commits（叠加提交）
3. 创建 `feature` bookmark
4. 对提交栈做 `squash`
5. 模拟 `main` 前进
6. 把 `feature` 提交栈 rebase 到新的 `main`
7. 在 rebase 过程中处理冲突
8. 演示 `jj undo` 撤销一次误操作
9. 使用 `jj git push --dry-run` 预演推送

当前仓库中的逻辑结构可以理解为：

```text
main    -> chore: advance main baseline
  |
  +-- feat: add jj workflow draft
        |
        +-- feature -> feat: add jj vs git notes
```

你可以用下面的命令再次查看：

```bash
jj log -r 'main::feature | main | feature | @'
```

---

## 2. 先理解 `jj` 的几个核心概念

在学习命令之前，先掌握几个和 Git 非常不同的概念。

### 2.1 `@`：工作副本提交

在 `jj` 里，你总是站在一个“工作副本提交”上，也就是 `@`。

- 它通常是一个可编辑的提交
- 你修改文件后，改动会归属于当前工作副本提交
- 执行 `jj commit` 后，`jj` 会自动为你创建下一个新的空工作副本提交

这和 Git 很不一样：

- Git 里你通常在分支上工作
- 你需要 `git add` 把内容放入暂存区，再 `git commit`
- 提交完成后，HEAD 指到新提交，但没有自动生成一个新的“空工作提交”供你继续写

可以把它理解成：`jj` 默认把“下一个要编辑的提交”提前准备好了。

### 2.2 `bookmark`：书签，而不是本地分支

在 `jj` 中，常见做法是：

- 本地历史随时改写
- 用 `bookmark` 标记你想对外暴露的提交
- push 时推送 bookmark 到 Git 远端

例如：

```bash
jj bookmark create feature -r @-
```

这条命令的含义是：

- 创建一个叫 `feature` 的 bookmark
- 指向当前工作副本的父提交 `@-`

和 Git 对比：

- Git 本地分支既是“工作位置”，又常常被当成“对外可见引用”
- `jj` 倾向于把“本地改写历史”和“对外暴露引用”这两件事分开

### 2.3 `revset`：比 Git 更强的修订选择语言

`jj` 的很多强大能力，来自 revset 表达式。

比如：

```bash
jj log -r 'main::feature | main | feature | @'
```

它表示：

- `main::feature`：从 `main` 到 `feature` 之间的祖先/后代路径
- 再把 `main`、`feature` 和 `@` 一起显示

再比如：

```bash
jj rebase -s 'roots(main..feature)' -o main
```

它表示：

- 找出 `main..feature` 这段差异中的根提交
- 从这些根开始，把整段提交栈 rebase 到 `main`

如果你熟悉 Git，会发现这比手写一连串 commit hash 顺手很多。

### 2.4 `op log` / `undo`：操作可撤销

`jj` 最让人安心的一个特性是：**很多“改历史”的动作都能撤销**。

比如你执行了错误的：

```bash
jj describe feature -m "WIP WRONG MESSAGE"
```

可以直接：

```bash
jj undo
```

这撤销的是“上一次仓库操作”，而不只是某个文件内容。

这和 Git 差别很大：

- Git 里很多历史改写也能补救，但通常需要 `reflog`、`reset`、`restore`、`checkout`、`rebase --abort` 等组合技
- `jj` 把“回到上一次操作之前”变成了一个一等能力

---

## 3. 这次在仓库里实际走过的完整工作流

下面按照真实操作过程，重新解释一遍。

### 步骤 1：查看初始状态

```bash
jj status
```

输出的重点通常是：

- `Working copy (@)`：当前工作副本提交
- `Parent commit (@-)`：当前工作副本的父提交

在本仓库刚开始时，`@` 是一个空的工作副本提交，父提交是 `main` 上的 `Initial commit`。

这正是 `jj` 的常见初始状态：

- 你不是“站在 main 分支上改文件”
- 而是站在 `main` 之后的一个工作副本提交上继续工作

### 步骤 2：创建第一个功能提交

我们先修改了两个文件：

- `README.md`
- `demo/workflow.md`

然后提交：

```bash
jj commit -m "feat: add jj workflow draft"
```

执行后会发生两件事：

1. 当前修改被固化成一个新提交
2. `jj` 自动生成下一个新的空工作副本提交

这就是为什么 `jj` 很适合做细粒度、小步快跑的提交。

### 步骤 3：继续叠第二个提交

接着又增加了一些内容，并创建第二个提交：

```bash
jj commit -m "feat: add jj vs git notes"
```

这样提交关系就变成：

```text
main
  -> feat: add jj workflow draft
      -> feat: add jj vs git notes
          -> @
```

这就是典型的 **stacked commits**：

- 每个提交只表达一个小改动
- 提交之间形成清晰的依赖顺序
- 后续可以单独改写某一层，而不是把所有改动揉成一个大提交

### 步骤 4：给功能栈建立 bookmark

```bash
jj bookmark create feature -r @-
```

这一步的目的是：

- 让 `feature` 指向刚刚完成的功能提交
- 后续 push 时能明确对外暴露哪一条线

在 `jj` 心智模型里：

- 提交栈可以继续变
- bookmark 是“我要对外追踪/同步的点”

### 步骤 5：做一次历史整理（`squash`）

随后我们又补了一次小修正，并把它压回前一个提交：

```bash
jj commit -m "fix: polish jj vs git bullets"
jj squash --from @- --into @-- --use-destination-message
```

可以这样理解：

- 先创建一个单独的修正提交
- 然后把这个修正的内容并回前一个提交

在 Git 里你也能做类似事情，但通常需要：

- `git commit --fixup`
- `git rebase -i --autosquash`

而 `jj squash` 直接把“把这层改动移到另一层提交”做成了显式命令。

### 步骤 6：模拟主线前进

为了演示 stacked commits 在协作中的优势，我们让 `main` 先向前走一步：

```bash
jj new main
jj commit -m "chore: advance main baseline"
jj bookmark move main --to @-
```

含义是：

- 从 `main` 开一个新的工作副本
- 增加一条主线变更
- 把 `main` 这个 bookmark 移到新提交上

这就模拟了现实中的常见情况：

- 主线有了新提交
- 你的功能栈需要重新挂到最新主线上

### 步骤 7：把功能栈 rebase 到新的 `main`

```bash
jj rebase -s 'roots(main..feature)' -o main
```

这是本次演示里最能体现 `jj` 风格的一步。

它不是“手动挑某个 commit hash 做交互式 rebase”，而是：

- 用 revset 精确描述“我要搬哪一段历史”
- 再明确说“我要挂到哪里去”

执行后，`feature` 这条提交栈整体被搬到了新的 `main` 之后。

### 步骤 8：处理 rebase 冲突

这次 rebase 中，`README.md` 发生了冲突。

`jj` 的推荐处理方式之一是：

1. 在冲突提交上新建一个工作副本
2. 修复文件内容
3. 再把修复 squash 回冲突提交

实际命令如下：

```bash
jj new mrtytmnr
# 手动修复 README.md
jj squash --into mrtytmnr --use-destination-message
```

这里有一个很重要的思想：

- 你不是直接“在冲突对象本体上硬修”
- 而是新建一个工作副本来解决问题
- 修复完成后，再把结果归并回目标提交

这让冲突解决过程更清晰，也更符合 `jj` 的提交导向思路。

### 步骤 9：演示误操作回滚

我们故意做了一次错误描述修改：

```bash
jj describe feature -m "WIP WRONG MESSAGE"
```

然后立刻回退：

```bash
jj undo
```

这一步展示的是 `jj` 最有辨识度的能力之一：

- 改写历史并不可怕
- 因为仓库操作本身也是可回退的

### 步骤 10：预演推送到 Git 远端

最后没有直接真的 push，而是做了 dry-run：

```bash
jj git push --bookmark feature --dry-run
```

输出显示本次会：

- 向 `origin` 新增 `feature` bookmark 对应的 Git 引用

这样做很适合演示和培训，因为不会真的改远端状态。

---

## 4. `jj` 和 `git` 的核心差异

下面按日常使用场景来对比。

### 4.1 工作模型不同

**Git：**

- 以 branch 为中心
- 以 staging area（暂存区）组织下一次提交
- 提交后继续在当前分支上改

**Jujutsu：**

- 以 changeset / commit stack 为中心
- 以工作副本提交 `@` 承载当前改动
- 每次提交后自动生成下一个空工作副本提交

结果是：

- Git 更像“在一个分支上不断做快照”
- `jj` 更像“把一串小变更组织成可以随时改写的提交栈”

### 4.2 改历史在 `jj` 里是默认能力

在 Git 中，很多人对改历史比较谨慎，因为：

- 命令组合多
- 心理负担大
- 一不小心就需要从 `reflog` 救火

在 `jj` 中，改历史是家常便饭：

- `jj squash`
- `jj rebase`
- `jj describe`
- `jj split`
- `jj abandon`
- `jj undo`

这让你更容易保持“每个提交都干净、独立、可审查”。

### 4.3 `bookmark` 和 `branch` 的定位不同

Git 里的 branch 既承担：

- 本地工作位置
- 远端同步对象
- 团队协作命名入口

而在 `jj` 中：

- 工作主要围绕提交栈进行
- bookmark 更像“给某个提交打一个可同步的名字”

所以你会发现：

- 本地可以频繁改写历史
- 对外只维护少量 bookmark

### 4.4 冲突处理思路不同

Git 里常见做法是：

- rebase/merge 过程中直接修文件
- `git add`
- `git rebase --continue`

`jj` 更强调：

- 在目标提交上下文中修复冲突
- 再把修复以提交方式归并回去

这更贴近“每个提交都应该能独立成立”的思想。

### 4.5 误操作恢复体验不同

Git 有很强的恢复能力，但你得知道去哪找：

- `reflog`
- `reset`
- `restore`
- `checkout`
- 各类 `--abort`

`jj` 则把“撤销上一步仓库操作”变成：

```bash
jj undo
```

对很多刚开始频繁改写历史的人来说，这个体验差异非常大。

---

## 5. 如果你从 Git 迁移到 `jj`，最值得改变的习惯

### 习惯 1：少依赖暂存区，多依赖小提交

Git 用户经常会这样想：

- 我先改很多文件
- 再慢慢挑到暂存区里

在 `jj` 里更自然的方式是：

- 先做一个小的、边界清晰的改动
- 直接 `jj commit`
- 后续再通过 `squash` / `split` / `rebase` 整理结构

### 习惯 2：把提交栈当成可编辑草稿，而不是最终成品

在 `jj` 里，提交不是“做完就不能碰”的东西。

更好的心态是：

- 先把思路拆成一串提交
- 随着理解变深，再不断整理提交边界
- 最后再把 bookmark 推出去

### 习惯 3：先学会 `undo`

刚开始用 `jj` 最重要的不是背所有命令，而是先建立信心：

```bash
jj undo
jj op log
```

知道自己能回来，才敢真正用上 `jj` 的历史改写能力。

---

## 6. 一个适合日常开发的 `jj` 工作流模板

下面是一套很实用的日常模板。

### 场景 A：开发一个功能

```bash
# 查看当前状态
jj status

# 修改文件

# 提交第一层
jj commit -m "feat: add api skeleton"

# 继续改
jj commit -m "feat: add validation"

# 再继续改
jj commit -m "test: add api tests"

# 如果某个提交内容边界不对，再整理
jj squash
# 或
jj split
```

### 场景 B：主线更新后同步你的提交栈

```bash
jj git fetch
jj rebase -s 'roots(main..my-bookmark)' -o main
```

### 场景 C：准备推送

```bash
jj bookmark create my-feature -r @-
jj git push --bookmark my-feature
```

### 场景 D：做错了，想回退上一步

```bash
jj undo
```

---

## 7. 本仓库这次演示里最值得记住的命令

```bash
# 查看状态
jj status

# 查看关键提交栈
jj log -r 'main::feature | main | feature | @'

# 创建提交
jj commit -m "feat: ..."

# 创建书签
jj bookmark create feature -r @-

# 把一个提交的改动压回另一个提交
jj squash --from @- --into @-- --use-destination-message

# 把提交栈整体 rebase 到 main
jj rebase -s 'roots(main..feature)' -o main

# 在冲突提交上新开工作副本
jj new <conflicted-revision>

# 解决完后压回目标提交
jj squash --into <conflicted-revision> --use-destination-message

# 撤销上一步仓库操作
jj undo

# 预演推送
jj git push --bookmark feature --dry-run
```

---

## 8. 什么时候 `jj` 比 Git 更有优势

如果你符合下面这些情况，`jj` 往往会明显更顺手：

- 你喜欢把一个功能拆成多个可审查的小提交
- 你经常需要整理、改写、合并、拆分提交
- 你不喜欢 `git add -p` / interactive rebase 的操作负担
- 你想更放心地做历史改写，因为可以 `undo`
- 你的团队评审更重视提交结构，而不是“最后只有一个大提交”

反过来说，如果你的工作方式一直是：

- 改完一大坨
- `git add .`
- `git commit -m "update"`

那 `jj` 的优势短期内不一定马上体现出来。它最大的价值在于：**让提交历史本身成为可持续维护的产物**。

---

## 9. 一句话总结

如果用一句话概括：

> Git 擅长“记录历史”，而 `jj` 更擅长“持续整理历史”。

在这个仓库中，你已经看到了 `jj` 的完整闭环：

- 创建提交栈
- 建立 bookmark
- 改写历史
- 跟随主线 rebase
- 处理冲突
- 撤销误操作
- 对接 Git 远端

这正是 `jj` 最有价值的地方。

