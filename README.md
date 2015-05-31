SVN Github Mirrors
==================
Provides a mirror of a collection of SVN repositories to Github.

Approach
--------
This script takes this approach:

* Use svn2git that relies on git-svn to download changes, but fixes the tags.

```
svn2git --rebase --metadata
```

* Remove large files with BFG. GitHub doesn't like files over 50MB.

```
bfg --strip-blobs-bigger-than 50M --no-blob-protection .
# Revert work copy to last commit, which might have been changed by BFG
rm -r ..bfg-report
git stash
git stash drop
```

* Push directly to GitHub, including local tags (no --mirror or --bare)
  --mirror will preserve the references to the inaccessible source repo. We just
  want the now-local tags to be migrated. Compare git branch -a vs git tag -l

```
git push -u --tags origin master
```

Other suggestions are here:

* https://help.github.com/articles/duplicating-a-repository#mirroring-a-repository
* https://gist.github.com/ticean/1556967
* http://jnrbsn.com/2010/11/how-to-mirror-a-subversion-repository-on-github
* http://danielpocock.com/automatically-mirroring-svn-repositories-to-git-and-github

This post helps understand git-svn:

* http://andy.delcambre.com/2008/03/04/git-svn-workflow.html


TODO
----
* SVN authors to authors.txt
* Re-nice the process to reduce resource consumption? It doesn't hurt for it to
  run slowly.


Setup
-----
### On the server
* Checkout the svn-github-mirrors repo

```
git clone https://github.com/esplinr/svn-github-mirrors.git svn-github-mirrors

```

* Setup SSH keys to login to GitHub. Setup ~/.ssh/config so that it uses that keyfile to connect to github.com

```
ssh-keygen -t rsa -C "<USER-EMAIL>" -f <OUTPUT KEY FILE>
```

* Install git-svn

```
sudo aptitude install git-svn
```

* Install svn2git: https://github.com/nirvdrum/svn2git

```
sudo aptitude install ruby rubygems
sudo gem install svn2git
```

* Install the cronjob

```
sudo echo "# Daily update of SVN Mirrors to GitHub" >> /etc/crontab
sudo sh -c "echo \"7  1 * * *  richard       /srv/svn-github-mirrors/update-mirrors.py\" >> /etc/crontab"

```

* Setup log rotation

```
# Copy so you can change permissions
cp /srv/svn-github-mirrors/logrotate.base /srv/svn-github-mirrors/logrotate.conf
sudo chown -R root.root /srv/svn-github-mirrors/logrotate.conf
sudo chmod a-w /srv/svn-github-mirrors/logrotate.conf
sudo sh -c "echo \"7 23 * * *  root          /usr/sbin/logrotate /srv/svn-github-mirrors/logrotate.conf\" >> /etc/crontab"
```

* Install BFG for cleaning git repos. Alias `bfg` to `java -jar /usr/local/lib/bfg.jar`
  http://rtyley.github.io/bfg-repo-cleaner/

* Setup each repo

### For each repo
* Create a directory in svn-clones, and move into it.
* Run svn2git to populate that directory with the repo
```
svn2git <SVN-URL> --trunk=HEAD --tags=<SVN-NAME> --metadata
```
On large repositories, this exits prematurely with the error "git-svn died of
signal 13". Restarting seems to work (outside of svn2git you would use "git svn fetch"), but there will be junk files in .git that need cleaning as well as extra entries in .git/config.
Here is how to wrap it in a loop to automatically restart:

```
svn2git <SVN-URL> --trunk=HEAD --tags=<SVN-NAME> --metadata; while [ $? -ne "0" ] ; do svn2git --rebase --metadata; done
```

* Create an empty GitHub repository: no wiki, and no issue tracker.
* Run BFG for the initial time

```
bfg --strip-blobs-bigger-than 50M --no-blob-protection .
rm -r ..bfg-report
git stash
git stash drop
```

* Add GitHub as the remote URL for push

```
git remote add origin <GITHUB-URL>
```

* Do an initial push

```
git push -u --tags origin master
```


Note
----
* Caution: It is easy to mess up the local repository after the mirror has
  started in such a way that the Git IDs change. This essentially trashes your
  GitHub mirror and forces you to start over, which is very inconvenient for anyone
  who is following your mirror.
* The update script can take hours to complete, even with few changes. This is
  due to svn2git needing to enumerate each tag for each revision, as well as the
  Git garbage collector being called both by svn2git and BFG.
* Git requires roughly 2x the repository size in order to do its processing, so
  make sure about 10GB are available on the server.
* It is possible to have GitHub recognize that a repository is a mirror of
  another Git repository, and pull automatically (see github.com/apache and
  github.com/mirrors). But this appears to have to be configured through GitHub
  support:
  http://stackoverflow.com/questions/11370239/creating-an-official-github-mirror
* We use the --metadata flag to svn2git in order to preserve the SVN revision ID
  in the git message. That makes it easier for a user to reference the correct SVN
  commit in bug reports.
* To change the push URL after the fact (to go from HTTP to SSH):

```
git remote set-url origin <NEW URL>
```

* I tried to rotate logs using Python's TimedRotatingFileHandler, but it  only works
  with long running processes. So I changed to logrotate, with all of the permissions
  mess that includes.


Thanks
------
* The first version of the synchronization script was donated by @Loftux.
* Subsequent work was sponsored by Alfresco Software.
