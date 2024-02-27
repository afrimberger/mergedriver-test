# mergedriver-test

Demo repository for analysing the execution of the merge-driver.
While updating Bitbucket from 7.x to 8.x and Git from 2.23 to 2.39, the merge driver
integration breaks.

# How to reproduce

```shell
$ git --version
git version 2.43.2
```

Clone this repo as bare repo to simulate Bitbucket's local repository storage 

```shell
$ cd /tmp
$ git clone --bare https://github.com/afrimberger/mergedriver-test.git mergedriver-test-bare
```

Copy and activate the test merge driver
```shell
$ cd /tmp/mergedriver-test-bare
$ git show HEAD:custom-merge-driver.sh > /tmp/custom-merge-driver.sh
$ chmod 777 /tmp/custom-merge-driver.sh
$ /tmp/custom-merge-driver.sh
>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>
Merge Driver is active
>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>
```

Define the merge driver settings in one of the "global" Git configs
```shell
$ cat <<EOF >> ~/.gitconfig
[user]
  name = John Doe
  email = john@doe.com
[merge "keeplocalversion"]
  name = "Keep local pom version"
  driver = /tmp/custom-merge-driver.sh
EOF
```

Create temporary Git repo to prevent changes in bare repo

```shell
$ mkdir /tmp/mergedriver-test
$ cd /tmp/mergedriver-test
$ git init
Initialized empty Git repository in /tmp/mergedriver-test/.git/
```

Reference objects from bare repo
```shell
$ echo "/tmp/mergedriver-test-bare/objects" > ".git/objects/info/alternates"
```

The working directory is empty
```shell
$ ls -al
total 12
drwxr-xr-x 3 root root 4096 Feb 23 18:10 .
drwxrwxrwt 1 root root 4096 Feb 23 18:10 ..
drwxr-xr-x 8 root root 4096 Feb 23 18:10 .git
```

Prepare merge

```shell
$ git reset 4814fce142b95b910d0cbe87c3c304b15f279a0a –
Unstaged changes after reset:
D             .gitattributes
D             .gitconfig-example
D             custom-merge-driver.sh
D             pom.xml
```


working directory is empty, but `HEAD:.gitattributes` exists
```shell
$ git show HEAD:.gitattributes
* merge=keeplocalversion
```

Execute `git merge-tree`. The merge driver isn't executed even though `HEAD:.gitattributes` exists.
```shell
$ git merge-tree --write-tree --allow-unrelated-histories --messages 561da86fa3990aa712aa3ec6681dfafd50def3a0 4814fce142b95b910d0cbe87c3c304b15f279a0a
8c1c8d51a82b07712c8e64109717f615a5e4a81e
100644 ded65fd3b11aa6a0b82f6ab170eaee5411c5ce2e 1	pom.xml
100644 29418c82b95793ee6b688bf6761ee53d8f42c1c5 2	pom.xml
100644 9bc48d5ca4b62d32cebf09f678148c8ef35b0820 3	pom.xml

Auto-merging pom.xml
CONFLICT (content): Merge conflict in pom.xml
```

**Expected Behaviour**: merge driver is executed

Proof merge driver is really active (Note: _ort_ doesn't respect `HEAD:.gitattributes`, but _recursive_ does)

```shell
$ git merge -s recursive -m "Automatic merge" --no-ff --no-log --allow-unrelated-histories --no-verify 561da86fa3990aa712aa3ec6681dfafd50def3a0 4814fce142b95b910d0cbe87c3c304b15f279a0a
>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>
Merge Driver is active
>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>
Auto-merging pom.xml
Merge made by the 'recursive' strategy.
```

Use "new" default merge strategy _ort_. Changed with Git 2.34 (breaks backwards compatibility).
_ort_ **doesn't** execute the merge driver

```shell
$ git merge -m 'Automatic merge' --no-ff --no-log --allow-unrelated-histories --no-verify 561da86fa3990aa712aa3ec6681dfafd50def3a0 4814fce142b95b910d0cbe87c3c304b15f279a0a
Auto-merging pom.xml
CONFLICT (content): Merge conflict in pom.xml
Automatic merge failed; fix conflicts and then commit the result.
```
