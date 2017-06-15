---
layout: post
title:  "Git Push 시 non-fast-forward 에러"
date:   2017-06-14 14:34:00 +0900
categories: Git
---

* content
{:toc}


git gui를 통해 Commit & Push 하려는데 에러난다.

Commit 하려는데
"You are about to commit on a detached head..."

Push 하려는데
"Updates were rejected because a pushed branch tip"

이러면서 **non-fast-forward**라는 에러를 뿜으며 안된다.

보통은 Local에 변경 사항이 남아있을때 발생한다고 하는데 다른게 없는데도 발생했다.

`gitk`로 보니
Git Local Branch Tip이 Remote Branch 보다 뒤에 있다. 즉 최신이 아니란다.

일단 checkout 해본다.
```sh
$ git checkout master
Switched to branch 'master'
Your branch is behind 'origin/master' by 1 commit, and can be fast-forwarded.
  (use "git pull" to update your local branch)
```

`git pull` 해보란다.

```sh
$ git pull
```

하고 나니 Updating을 한다.

다시 chekcout 해보니
```sh
$ git checkout master
Already on 'master'
Your branch is up-to-date with 'origin/master'.
```

이제 잘된다~~

정리하면 **non-fast-forward** 에러가 발생하면 `git pull` 해주면 된다.
