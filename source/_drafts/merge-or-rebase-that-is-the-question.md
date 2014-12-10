---
layout: post
title: Merge or Rebase, that's the question
tags:
  - tools
  - git
---
Among my previous companies, I see a common pattern about Git. A lot of Git users stick on one command to update their
branches: `merge` or `rebase`. Sometimes they are using one and change their mind to use the other one because
something "weird" happens. The reality is, you should use both.

The `merge` command is used to add the content of one branch to another one. For example, if you want to add the content
from feature/add_client_account into the master branch, you should merge your feature into master.

The `rebase` command can do the same thing that `merge` but, moreover, it rewrites the history. It means, if you `rebase`
your commits, your Git commits will look the same (same message, same content,..) but will not be the same (different SHA).
You've to avoid this command when you're working on a shared branch like master, develop, ... Otherwise your commits
will not be the same as the commits of your coworkers.
Hopefully, Git does not authorize you to push this kind of things.
You will have to force the push (protips: if you have to force push, you're currently doing something really bad).

## When rebasing your branch?

### Shared branch

I know, I said you should avoid this command on a shared branch. It's not totally true. You should use this command
instead of `merge` when you have some commits in your branch that are not yet pushed and Git rejects your push because
another coworker already pushed some code.

For example, you are on the branch feature/add_client_account, you did one commit to add tests on this feature and, in
the meantime, your teammate did a commit to fix some broken css.

If you try to push, Git will display: "Updates were rejected
because the tip of your current branch is behind its remote counterpart. Merge the remote changes".

In this case you have some commits on your remote Git repository and you have different commits on your local Git repository.
If you merge the remote in your local, it will work but you will have a clunky history like the following one:

```bash
* acc335f - (HEAD, github/name-of-my-branch, name-of-my-branch) commit 3 (2014-12-10 12:00:36) <Richard>
*   330a695 - Merge branch 'name-of-my-branch' of github.com:compnay/project into name-of-my-branch (2014-12-10 10:18:45) <Andrew>
|\
| * 659716b - commit 2 (2014-12-10 10:01:22) <Richard>
* | 88cc235 - commit 2 bis (2014-12-10 10:18:21) <Andrew>
|/
* ab01dcd - commit 1 (2014-12-09 15:04:38) <Richard>
```

As you can see the history is difficult to read. In this case Richard did one commit the 9th at 15:00. Andrew was working
on this version too. The 10th, each developer did a commit. One at 10:01 and the other one at 10:18. Richard pushed his
commit before Andrew. So when Andrew tried to push, Git rejected his commit. So Andrew merged his branch in itself.
It does not cause any Git trouble, but it's hard to read the history because you have useless commit (#330a695) and useless
branching. Instead, **in this case**, you should `rebase` your branch. **In this case**, it's not dangerous. The only history
you will rewrite is your own history. So you can't break your whole Git. In this case you should run `git pull --rebase`. This
command will:

* generate patchs for your commits
* "remove" your commits
* apply the remote commits on your branches
* replay your commit patchs over the branch.

Because Git needs to replay your commits, your SHA will change.

In this case you could have an history like:

```bash
* acc335f - (HEAD, github/name-of-my-branch, name-of-my-branch) commit 3 (2014-12-10 12:00:36) <Richard>
* a24d2c5 - commit 2 bis (2014-12-10 10:18:21) <Andrew>
* 659716b - commit 2 (2014-12-10 10:01:22) <Richard>
* ab01dcd - commit 1 (2014-12-09 15:04:38) <Richard>
```

Much more readable, right?

### At the end of your feature

I don't know you but I, each time I've something working, I commit. It leads to a lot of commits at the end of my
features. Example:

```bash
* 930f433 - (HEAD, github/feature/add-pfi-alert, feature/add-pfi-alert) Refactor some order alert workers (2014-12-01 10:13:20 +0000) <Kevin>
* 7377c49 - Refactor customer alert workers (2014-11-28 14:43:07 +0000) <Kevin>
* ebf281b - some hysteria: single quotes, alignment,... (2014-11-28 11:16:28 +0000) <Kevin>
* 0bd2a45 - Remove commented code in order fraud check worker (2014-11-28 10:52:01 +0000) <Kevin>
* d3ab592 - Close all alerts for high rating users when user has a high rating (2014-11-28 10:44:29 +0000) <Kevin>
* 3548059 - Wraps product price import worker with a site (2014-11-25 10:46:25 +0000) <Kevin>
* e65e8af - Remove useless information in factort alert creation (2014-11-24 16:21:08 +0000) <Kevin>
* e10a3b1 - Move close alert worker to a customer specific workers namespace (2014-11-24 11:59:07 +0000) <Kevin>
* 9d40c35 - Require files instead of loading them because load does not care if the file is already loaded (2014-11-24 10:10:38 +0000) <Kevin>
* 623b9f3 - Add a task to close alert for high rating users (2014-11-19 11:40:21 +0000) <Kevin>
* 3fd327f - Refactor pfi order value worker (2014-11-19 10:00:38 +0000) <Kevin>
* 27c0621 - Move misc alert workers tests in the good folder (2014-11-19 09:53:21 +0000) <Kevin>
* 2a2a919 - Move oder alert workers tets in the good folder (2014-11-19 09:52:41 +0000) <Kevin>
* 090f9fc - Refactor customer low rating worker (2014-11-19 09:35:16 +0000) <Kevin>
* e1544fd - Refactor customer high volume worker (2014-11-19 09:33:20 +0000) <Kevin>
* a25caca - Refactor customer email worker (2014-11-18 18:14:40 +0000) <Kevin>
* 3ba1e5f - Do some basic refactoring on OrderAlertWorker job (2014-11-18 17:48:08 +0000) <Kevin>
* 5d79cdb - Split alert order worker to many files (2014-11-18 15:06:24 +0000) <Kevin>
* 29efa07 - Add last tests for Alert Workers (2014-11-14 17:06:11 +0000) <Kevin>
* b19e25a - Add test for the order fraud checker (2014-11-14 12:03:14 +0000) <Kevin>
* 9c0f6b5 - WIP: add tests for customer/order alert workers (2014-11-10 08:44:01 +0000) <Kevin>
* f1faa3c - Add PFI alert for orders over $80, resolves: #4852 (2014-11-10 08:44:01 +0000) <Kevin>
```

It's nice in your development process because you can rollback to a previous step easily. Eventually, it can also help
your coworkers to follow your mind (if possible) when they have to read your Pull Request. Moreover, except if you're
really the best developer ever, you could have, sometimes, to change your code after some discussions in the Pull Request.
All these commits are normal in a development process but shouldn't appear in your Git history. They does not add any useful
informations when come the time to find something in your history.

Once the Pull Request is finished and **you're sure** nobody will push anything on this branch, you should [rewrite your
history](http://git-scm.com/book/en/v2/Git-Tools-Rewriting-History) to have a clean one, instead of just clicking the
"Merge pull request" button on GitHub. Rewriting your history consists in merging commits together, removing some commits,
rewriting a commit message, reordering the commits,... For example, the previous example could be rewritten like:

```bash
* c627dbe - (HEAD, github/feature/add-pfi-alert, feature/add-pfi-alert) Split the order alert workers tasks in small files (2014-11-14 12:03:14 +0000) <Kevin>
* 8b2c64a - Add tests for all the tasks of the order alert workers (2014-11-10 08:44:01 +0000) <Kevin>
* d5bc45f - Add PFI alert for orders over $90, resolves: #4852 (2014-11-10 08:44:01 +0000) <Kevin>
```

When your commits are rebased and the history is clean, you can push your changes to the remote. In this case, you will
have the same error message than previously. It's **the only case** you should allow yourself to force a push. Once you
finished to rewrite the history, you can merge your branch on your development branch as usual.

## When merging your branch?

In all other cases, you should merge. It's safer for your repository.
