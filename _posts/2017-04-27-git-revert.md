---
layout: post
title:  "Git Revert "
date: 2017-04-27
categories: git
---

Sometimes you commit some changes to master that you should not have. Sometimes you push those changes to a remote. And sometimes in between those regretful pushes someone else pushes some commits to that same remote. 

Now, here's the problem; You want to remove all of those commits from master, yet keep your colleague's commits. The log will look something like this:

{% highlight shell %}
morgan@toaster:~/work/git_revert$ git log --oneline
6cd4307 D
a637eb6 keep this commit.
f3c12ce C
3422c11 B
316319e A
{% endhighlight %}

Commits B-D are bad, maybe work that should've been committed to a branch. You want to get rid of the changes introduced in those commits, but you want to keep the changes introduced in a637eb6. 

The following is a simple approach:
1. branch so have a refernce to the current HEAD (if you want).
2. switch back to master.
3. [revert](https://git-scm.com/docs/git-revert) the changes introduced from commits in the range A-D.
4. [cherry-pick](https://git-scm.com/docs/git-cherry-pick) the commits you want to keep.

### A note about commit ranges ###
* You can reference the current branch with HEAD.  
* You can reference the parent of the current HEAD with HEAD^.  
* You can reference the nth parent HEAD^n (only useful in merge commits).
{% highlight shell %}
morgan@toaster:~/work/git_revert$ git show HEAD^
commit a637eb6969e0ba18d34a3e1c8bd338903ac61376
Author: morgan <morgan@unlisted.org>
Date:   Tue May 2 17:33:50 2017 -0400

    keep this commit.

diff --git a/keep_this.txt b/keep_this.txt
new file mode 100644
index 0000000..e69de29
morgan@toaster:~/work/git_revert$ git show HEAD^1
commit a637eb6969e0ba18d34a3e1c8bd338903ac61376
Author: morgan <morgan@unlisted.org>
Date:   Tue May 2 17:33:50 2017 -0400

    keep this commit.

diff --git a/keep_this.txt b/keep_this.txt
new file mode 100644
index 0000000..e69de29
{% endhighlight %}
If you want to see the nth ancestor of HEAD you can use ~ instead of ^
* HEAD~ is the first parent of HEAD
* HEAD~1 is also the first parent of HEAD
* HEAD~n is the nth parent of HEAD

{% highlight shell %}
morgan@toaster:~/work/git_revert$ git show HEAD
commit 6cd4307950afbdd719971f0b5d7c880b767671f8
Author: morgan <morgan@unlisted.org>
Date:   Tue May 2 17:34:08 2017 -0400

    D

diff --git a/file5.txt b/file5.txt
new file mode 100644
index 0000000..e69de29
morgan@toaster:~/work/git_revert$
morgan@toaster:~/work/git_revert$ git show HEAD~2
commit f3c12cea2a0131a0cc7f9832fcf52bf49fe72c46
Author: morgan <morgan@unlisted.org>
Date:   Tue May 2 17:33:08 2017 -0400

    C

diff --git a/file3.txt b/file3.txt
new file mode 100644
index 0000000..e69de29
{% endhighlight %}

Now we can use the double dot notation to reference a range of commits. The [documentation](https://git-scm.com/book/en/v2/Git-Tools-Revision-Selection) explains that double dots resolves to all commits reachable from one commit, but not reachable from the other. 

I think it makes more sense when working with multiple braches, but it's also useful for resolving ranges of commits on the same branch.
{% highlight shell %}
morgan@toaster:~/work/git_revert$ git checkout -b new_feature
Switched to a new branch 'new_feature'
morgan@toaster:~/work/git_revert$ echo "testing123" > test123.txt
morgan@toaster:~/work/git_revert$ git add test123.txt
morgan@toaster:~/work/git_revert$ git commit -m "testing123"
[new_feature bdfacfb] testing123
 1 file changed, 1 insertion(+)
 create mode 100644 test123.txt
{% endhighlight %}
We can see all the commits reachable by new_feature but not by master.
{% highlight shell %}
morgan@toaster:~/work/git_revert$ git log master..new_feature
commit bdfacfb48d402ddb7b2021e6a5b7752fa6cef59b
Author: morgan <morgan@unlisted.org>
Date:   Thu May 4 11:43:38 2017 -0400

    testing123
{% endhighlight %}
And vice vera, all commits reachable by master but not by new_feature. There should be none.
{% highlight shell %}
morgan@toaster:~/work/git_revert$ git log new_feature..master
morgan@toaster:~/work/git_revert$
{% endhighlight %}
### Back to revert ###
Remember we want to revert the range of commits from B-D.
In this case those commits happen to be the 4th parent of HEAD to HEAD.
{% highlight shell %}
morgan@toaster:~/work/git_revert$ git log --oneline HEAD~4..HEAD
6cd4307 D
a637eb6 keep this commit.
f3c12ce C
3422c11 B
{% endhighlight %}
Using commit hashes we want to revert commits reachable by 6cd4307 but not by 316319e, or 316319e..6cd4307. Remember the range syntax is not_reachable..reachable 
{% highlight shell %}
morgan@toaster:~/work/git_revert$ git log --oneline 316319e..6cd4307
6cd4307 D
a637eb6 keep this commit.
f3c12ce C
3422c11 B
{% endhighlight %}
And using HEAD reference 
{% highlight shell %}
morgan@toaster:~/work/git_revert$ git log --oneline HEAD~4..HEAD
6cd4307 D
a637eb6 keep this commit.
f3c12ce C
3422c11 B
{% endhighlight %}
Now we can revert the range and apply the commit we wish to save on top. Use -n with revert to prevent a commit for each revert from occuring. 
{% highlight shell %}
morgan@toaster:~/work/git_revert$ git revert -n  HEAD~4..HEAD
morgan@toaster:~/work/git_revert$ git status
On branch master
Your branch is up-to-date with 'origin/master'.
You are currently reverting commit 3422c11.
  (all conflicts fixed: run "git revert --continue")
  (use "git revert --abort" to cancel the revert operation)

Changes to be committed:
  (use "git reset HEAD <file>..." to unstage)

        deleted:    file2.txt
        deleted:    file3.txt
        deleted:    file5.txt
        deleted:    keep_this.txt
{% endhighlight %}
There are no conflicts to fix since the revert simply deleted files that were added in previous commits. You may have to manually fix any conflicts introduces by the revert.

We can cherry-pick the commit we want to apply on top of the revert, but our working copy must be clean. So first complete the revert.

{% highlight shell %}
morgan@toaster:~/work/git_revert$ git revert --continue
[master 1e9cc73] revert commits B-D
 4 files changed, 0 insertions(+), 0 deletions(-)
 delete mode 100644 file2.txt
 delete mode 100644 file3.txt
 delete mode 100644 file5.txt
 delete mode 100644 keep_this.txt
{% endhighlight %}
And then cherry-pick
{% highlight shell %}
morgan@toaster:~/work/git_revert$ git cherry-pick -e  a637eb69
[master 01fc753] keep this commit.
 Date: Tue May 2 17:33:50 2017 -0400
 1 file changed, 0 insertions(+), 0 deletions(-)
 create mode 100644 keep_this.txt
morgan@toaster:~/work/git_revert$ git show 01fc753
commit 01fc753710de2a8fd3d7a126592b7a9d2f6f15fe
Author: morgan <morgan@unlisted.org>
Date:   Tue May 2 17:33:50 2017 -0400

    keep this commit.

diff --git a/keep_this.txt b/keep_this.txt
new file mode 100644
index 0000000..e69de29
{% endhighlight %}
Alternatively you can do it all in one commit using -n to cherry-pick.
{% highlight shell %}
morgan@toaster:~/work/git_revert$ git revert -n  HEAD~4..HEAD
morgan@toaster:~/work/git_revert$ git cherry-pick -e -n  a637eb69
morgan@toaster:~/work/git_revert$ git commit
morgan@toaster:~/work/git_revert$ git log --oneline
4c2ceed Revert B-D and apply a637eb6 on top.
6cd4307 D
a637eb6 keep this commit.
f3c12ce C
3422c11 B
316319e A
morgan@toaster:~/work/git_revert$ git show 4c2ceed
commit 4c2ceed41d7df12da7554ba1c655d79dd082d067
Author: morgan <morgan@unlisted.org>
Date:   Fri May 5 16:31:41 2017 -0400

    Revert B-D and apply a637eb6 on top.

diff --git a/file2.txt b/file2.txt
deleted file mode 100644
index e69de29..0000000
diff --git a/file3.txt b/file3.txt
deleted file mode 100644
index e69de29..0000000
diff --git a/file5.txt b/file5.txt
deleted file mode 100644
index e69de29..0000000
morgan@toaster:~/work/git_revert$ ls
file1.txt  keep_this.txt
{% endhighlight %}
  
  
  

#### References ####
[git revert](https://git-scm.com/docs/git-revert)  
[git cherry-pick](https://git-scm.com/docs/git-cherry-pick)  
[git revision selection](https://git-scm.com/book/en/v2/Git-Tools-Revision-Selection)  
