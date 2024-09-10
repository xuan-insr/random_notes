---
date: 2024-09-05
categories:
    - :rainbow:Topics
    - :writing_hand_tone1:Git
---

# Daily Git

大多数开发者会频繁用到 Git。但一些同志没有系统学习过 Git，也有一些同志学习 Git 的时间可能比较古早，参考的资料也可能比较经典。因此，本文尝试为熟悉 Git 工作流的朋友提供一些我最近学到的 Git 的基础知识以及一些我觉得有用的技巧。

!!! warning
    本文尚未完善

<!-- more -->

从 `.git` 目录说起。

---

`branches` 是一个 deprecated 的目录，新的 git 版本已经不用了。

---

`objects` 目录存储所有的数据。每个文件的内容都会被压缩，然后以哈希值命名，放在这个目录下。

示例：`echo "Hello, World\!" > init.txt` `git add init.txt`
此时可以看到 `objects` 目录下多了一个文件

可以通过 `zlib-flate -uncompress < .git/objects/xx/xxxx` 看到其中的内容

再尝试 `zlib-flate -uncompress < .git/objects/xx/xxxx | sha1sum` 可以验证 SHA 一致

由于 [SHA1](https://shattered.io/) 已经被证明不安全，因此 Git 在 2.13 中做了补丁；在 2.29 中实验性支持 SHA-256；在 2.42 中正式支持 SHA-256；在 2.45 中实验性支持两种 SHA 的互操作

`git commit -m "init"` 之后，可以看到 `refs` 目录下多了两个文件，分别使用 `git cat-file -p` 查看可以看到一个 tree 和一个 commit；我们稍后讨论这个 tree 是怎么来的

---

`hooks` 目录存储一些钩子脚本，可以在特定的事件发生时执行；这个目录下默认会放一些示例脚本

看一下 `pre-commit.sample`；这里我们看到了 `git config` 的用法；事实上 `git config` 是可以自定义的，自定义配置就可以在 hook 之类的地方用到

`git config` 控制 git 的行为；主要可以在 4 个地方设置配置：`/etc/gitconfig`、`~/.gitconfig`、`.git/config`, `.git/config.worktree`，分别对应 `--system`、`--global`、`--local`、`--worktree`，优先级从低到高

> `git worktree` 是一个比较新的功能，解决的是大家 WIP 但是要转而开发个别的东西，但是又不想 `git stash` 的情况。但是它和我们的 monorepo + bazel 的玩法相性不是很好，而且我也不觉得 `git stash` 不好用，所以没学过
> 
> `git stash` 就不聊了，大家都知道

可以通过 `git config --list --show-origin` 查看所有配置。例如可以看到我有 `push.autosetupremote=true`，这个表示 `git push` 时如果 remote 上没有对应的分支，会自动创建一个，而不是报错。(如果你发现你的这个选项没生效，可以升级一下 git；现在开发机预装的版本是不够高的)

我还设置了一堆 alias。

可以在 [这些](https://github.com/jessfraz/dotfiles/blob/master/.gitconfig) [例子](https://gist.github.com/pksunkara/988716) [中](https://github.com/mgedmin/dotfiles/blob/master/gitconfig) 找到一些有用的配置；三个的风格各有不同，大家有兴趣可以自己探索一下

---

`refs` 存放本地分支、远程分支和 tags 的引用；每一个存放的其实是一个 commit 的哈希值

`git for-each-ref` 可以查看和筛选 tags or branches；例如 `git for-each-ref --sort=-creatordate --format="%(refname:short)" refs/tags/`
因此如果 `.. | grep "prod-release" | head -n 1` 就可以筛选出最新的含 `prod-release` 的 tag 了

什么是远程？其实远程非常简单，甚至可以在本地。我们可以 `git init --bare .` 创建一个 bare repo，然后在另一个目录将这个路径 `git remote add origin /path/to/repo`，这样就可以在本地操作远程仓库了。
bare repo 其实就是没有 working directory 的 repo，只有展开的 `.git` 目录；这是作为 remote 的 best practice，表示这个 repo 只是用来共享代码，不是用来开发的。

```
mkcd ~/git/remote
git init . --bare
tree -a ~/git/remote

mkcd ~/git/repo
git init .
git remote add origin ~/git/remote

echo "Hello, World\!" > init.txt
git add .
git commit -m "init"
git push
tree -a ~/git/remote

mkcd ~/git/another_local
git clone ~/git/remote .
tree . -a
```

展示一下 `git push` 的效果，展示一下 `git fetch`；所以其实 remote 其实就是在不同的几个 repo 之间交换 objects 和 refs。

---

`packed_refs` 存储一些 packed refs，这是一种优化，将一些 refs 打包在一起，减少文件数量，以提高性能。Git 2.45 引入了 [新的想法](https://github.blog/open-source/git/highlights-from-git-2-45/#preliminary-reftable-support)

---

`logs` 目录存储一些日志信息，包括 reflog，可以用来恢复一些误操作

例如当我们做了一些改写历史的操作之后，可以通过 `git reflog` 查看历史操作，然后 `git reset` 恢复之前的状态

但是要注意，`git reflog` 只会记录本地发生过的变更；如果是在 remote 上发生的、没有被 fetch 下来的变更，本地自然是没有的。

因此如果 `git push -f` 覆盖了别人的提交，除非去 remote 服务器上查看，或者去求同事重新 push 一遍，否则是没法恢复的（更坏的情况是根本没发现）

因此我们向大家隆重介绍 `git push --force-with-lease`，这个命令会检查 remote 上的 ref 是否和本地已知的 remote ref 一致，如果不一致就不会 push，以避免覆盖未被 pull 的提交。

> 请注意：这个指令的语义（具体做什么判断）可能还会变化，它「可能随 git 团队的经验而改变」；但其解决的问题是确定的。

---

`info` 目录存储一些额外的信息，例如 `info/exclude` 用来存储 `.gitignore` 之外的 ignore 规则；这表示我希望在这个 repo 中忽略的文件，但是不希望提交到 `.gitignore` 中

但是需要注意的是，`exclude` 和 `.gitignore` 都是针对 untracked 文件的；如果文件已经被 track 了，那么即使在其中新增了，未来的更新也不会被忽略

那如果我确实想要忽略一个已经被 track 的文件呢？可以使用 `git update-index --skip-worktree`，这个命令会将文件标记为 skip worktree，这样即使文件被修改了，也不会被提交。这个状态不会被提交到 remote
这对于「我想在本地改些配置」的情况非常有用，例如我在本地 clone 了两个仓库来分别启动两套服务，我会修改启动服务的脚本中的端口号，但是通过这个方式标记为不要提交，这样我就不会不小心提交到 remote 了

不过很遗憾的是，目前还没有一个方式能让所有开发者都不提交这个文件；所以这个方法只能用在个人开发的场景中

---

`config` 文件的用途，刚才已经聊过了

---

`HEAD` 文件存储当前所在的分支，或者当前所在的 commit

---

`index` 是 git 中特别重要的一个概念，它是 working directory 和 repo 之间的桥梁



---

可以在 [这里](https://ndpsoftware.com/git-cheatsheet.html#loc=workspace;) 看到一个很好的 interactive cheetsheet

其中 `git restore` 和 `git switch` 不是很全面

这两个指令用来解决「所有事情都是 `git checkout` 做的」的问题

---

`git add -i` 可以交互式地完成 `git add`，包括只添加一个文件的一部分；而 `git add -p` 则直接进入 patch 模式
值得一提的是 `git stash` 也有 `-p`。

如果在某个子目录，想要 `git add` 仓库中的所有文件而不只是当前目录下的文件，可以使用 `git add :/` 或者 `git add -A`。

`:/` 是 pathspec 的一种特殊形式，表示 repo root；例如 `:/app` 表示 repo root 下的 app 目录。

另外 `git add -u` 只会 add 已经被 track 的文件，不会 add 新文件。也可以使用 `git add -u :/`。

pathspec 的另一种有用形式是 `:!`，表示 exclude；例如 `git add . ":!app"` 表示 add repo root 下的所有文件，但是不 add app 目录。
- 在 zsh 中，`!` 需要转义写为 `\!`
- 当然这种 case 我一般会选择 `git add .` 然后 `git restore --staged app`

关于几种 `git add` 的区别，可以参考 [这里](https://stackoverflow.com/a/26039014/14430730)。

 

---

TODO

- https://github.com/tj/git-extras/blob/master/Commands.md
- https://github.com/git-tips/tips

---

使用 `git commit --fixup` 和 `git rebase --autosquash` 来 fixup 一个遥远的 commit：https://andrewlock.net/smoother-rebases-with-auto-squashing-git-commits/
自 2.44 起，不需要 `-i` 也可以使用 `--autosquash` 了！

