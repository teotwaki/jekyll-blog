---
layout: post
title:  "RPM versioning with Jenkins"
date:   2014-02-01 22:27
categories: jenkins
tags:
---
I'm currently deploying Jenkins at work, in order to simplify the build
process. At the moment, we have a fairly complicated process that involves a
couple dozen steps, which, even though each step is fairly braindead, does
require a good amount of focus and planning in order to get everything just
right.

[Jenkins][1] provides a build number for all the different projects you might
configure. Each build number is unique in a given project; this ensures that
you can refer to specific builds without too much of a hassle. That's great,
but knowing that I'm trying to achieve fully automated RPM packaging, it
doesn't help me much.

The versioning system in our company (defined by moi, pretty much), is heavily
inspired by the usual RPM rules: release numbers are reset to 1 whenever the
version number changes. There's no real way, as far as I know, to use Jenkins'
build number for this, as there is no programmatic way to reset the build
number in Jenkins.

# Multi-instance exclusive lock in Python #

The first issue I tried to tackle was the fact that the Jenkins server will run
multiple builds at the same time, possibly for the same project. What this means
is that simply having stuff in a file isn't really sensible due to concurrency
issues. I know that [portalocker][2] roughly does the same as what I tried to
do, but not completely. portalocker provides a lock to a file; my implementation
provides a transaction-kinda lock, regardless of whether you want to access
files or databases or whatever.

Without further ado, here's the class:

```python
import os, time

class Lock(object):
    """
    Manage a system-wide lock used to inform other processes they should wait.
    The lockfile argument is the token that should be shared amongst all the
    processes wanting to acquire exclusive locks to the same resource.
    """

    polling = 0.1

    def __init__(self, lockfile):
        self._lockfile = '%s.lock' % lockfile
        # I only target linux systems... Sorry.
        self._path = os.path.join('/tmp', self._lockfile)
        self.locked = False

    def __enter__(self):
        self.acquire()
        return self

    def __exit__(self, _, __, ___):
        self.release()

    def acquire(self):
        """
        Try to acquire a lock. This is a blocking call, until the lock is
        acquired. If you want a non-blocking call, try nb_acquire().
        """
        if self.locked:
            """ TODO: Raise """
            pass

        while not self._acquire():
            time.sleep(self.polling)

        self.locked = True

    def nb_acquire(self):
        if self.locked:
            """ TODO: Raise """
            pass

        self.locked = self._acquire()
        return self.locked

    def _acquire(self):
        try:
            os.mkdir(self._path)
        except OSError:
            return False
        return True

    def release(self):
        if not self.locked:
            """ TODO: Raise """
            pass

        os.rmdir(self._path)

        self.locked = False
```

There's a couple of TODOs, and I might push this stuff to my github account at
some point or another, but it gives a general idea on what I tried to do. The
jist of it is that I use mkdir as an atomic locking operation. What you lock
against is basically just an identifier, which means that whether you want to
read/write to files, access a database, fiddle your toes doesn't realy matter.
If the lock acquisition fails, the `acquire()` method will block until it is
able to acquire the lock. If you don't want it to block, there's always
`nb_acquire()` which will just return a boolean indicating the lock status.

## How to use it ##

Here's how it could be used:

```python
lock_name = 'my-lock'

# Primary usage
with Lock(lock_name):
    print "We should be locked!"
```

And that's really just it. The locking magic happens in the `with` statement. As
soon you enter the `with` statement, you are guaranteed to be the sole executor
of whatever code you have running. As soon as you leave the scope of the `with`
statement, the lock is released. Below are a few other scenarios of how you
could use the above class:

```python
lock_name = 'my-lock'

# Another kind of usage
lock = Lock(lock_name)
lock.acquire()
print "We should be locked!"

# If you do not call the following function, you're going to have bugs.
lock.release()

# Yet another kind of usage (non-blocking, this time)
if lock.nb_acquire():
    print "We should be locked!"
    lock.release()
else:
    print "We're most definitely not locked!"
```

So, now we can execute a block of code without having to worry about other
instances of the script mucking about our stuff. Let's work on the RPM
versioning now.

# Efficient RPM release numbers #

RPM versioning works like this: `1.2.3-4` where `1.2.3` is the version number,
and `4` is the release. For a more in-depth overview of RPM versioning, check
out Fedora's wiki, or something. I usually put the version number of my software
in a file somewhere, so getting that isn't usually a problem. However, the
release part of the RPM version is a bit more tricky. The way we use it is that
every time we release something to the QA team (or directly to production if
it's a very urgent bugfix), we increment the release number if the version
number hasn't changed. This is the case, for example, if you have to fix
something in the packaging process, or if you are applying successive fixes to
the release branch (according to [nvie's branching model][3]).

This also means that as soon as the version number changes, the release number
should be reset to 1. I wrote the following tool, which uses the `Lock` class
above, to keep track of all the different releases that have been built so far,
based on the package name and version number:

```python
import argparse, json
from lock import Lock
from os.path import expanduser, join

class RPMReleases(object):

    def __init__(self):
        self._path = join(expanduser('~'), 'iv-rpm-releases')
        self._releases = {}

    def __enter__(self):
        self.load()
        return self

    def __exit__(self, _, __, ___):
        self.dump()

    def load(self):
        try:
            with open(self._path, 'r') as f:
                self._releases = json.load(f)
        except IOError:
            pass

    def dump(self):
        with open(self._path, 'w+') as f:
            json.dump(self._releases, f)

    def get_next_release(self, package, version, increment = True):
        if package not in self._releases:
            self._releases[package] = {}

        if version not in self._releases[package]:
            self._releases[package][version] = 0

        if increment:
            self._releases[package][version] += 1

        return self._releases[package][version]

    def set_release(self, package, version, release):
        if package not in self._releases:
            self._releases[package] = {}

        self._releases[package][version] = release

if __name__ == '__main__':
    parser = argparse.ArgumentParser()
    parser.add_argument("package",
        help = "The software package for which you want a release number")
    parser.add_argument("version",
        help = "The version of the package for which you want a release number")

    parser.add_argument("--current", action = "store_true",
        help = "Don't increment any counters, just print the current value")
    parser.add_argument("--full", action = "store_true",
        help = "Print the full package name")

    args = parser.parse_args()

    with Lock('iv-rpm-releases'):
        with RPMReleases() as releases:
            inc = not args.current
            release = releases.get_next_release(args.package, args.version, increment = inc)

            if not args.full:
                print release

            else:
                print '%s-%s-%s' % (args.package, args.version, release)
```

## How to use it ##

The tool is used as follows:

```
$ python rpm-release-number.py -h
usage: rpm-release-number.py [-h] [--current] [--full] package version

positional arguments:
  package     The software package for which you want a release number
  version     The version of the package for which you want a release number

optional arguments:
  -h, --help  show this help message and exit
  --current   Don't increment any counters, just print the current value
  --full      Print the full package name
```

For example:

```
$ python rpm-release-number.py foo 1.1
1
$ python rpm-release-number.py foo 1.1
2
$ python rpm-release-number.py foo 1.1
3
$ python rpm-release-number.py foo 1.1 --current
3
$ python rpm-release-number.py foo 1.1 --current 
3
$ python rpm-release-number.py foo 1.1 --current --full
foo-1.1-3
```

So, there we have it. A quick tool that keeps track of what release every
package is supposed to have, and stores it in an easy-to-maintain location (a
JSON file in the user's home directory). This tool is easily usable in Jenkins
scripts, but that'll be for a different blog entry.

[1]: http://jenkins-ci.org/
[2]: https://pypi.python.org/pypi/portalocker
[3]: http://nvie.com/posts/a-successful-git-branching-model/
