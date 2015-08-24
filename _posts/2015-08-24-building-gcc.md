---
layout: post
title:  "Building GCC"
date:   2015-08-24 14:59
categories: blog
excerpt: >
  A look at how to build a backup version of GCC in case something goes wrong
  with the version provided by your distribution's package manager.
---
There's a [nasty bug][gcc-bug] that plagues GCC 5.1 (the current version of GCC
in Fedora 22), which blocks the unit tests for Realm. Instead of going to the
very bleeding edge, I've decided to compile version 4.9.3 and keep that as a
backup to run tests and whatnot. The bug has been fixed and should be part of
the GCC 5.3 release, so we'll just have to be patient until then.

Sometimes, Fedora's race-to-the-new can bite you a tiny bit.

There are a couple of dependencies needed other than the usual build tools:

    sudo dnf install -y gmp-devel mpfr-devel libmpc-devel

It's fairly simple to build GCC. I wouldn't expect otherwise, frankly:

```
cd src/foss && \
  wget http://mirror1.babylon.network/gcc/releases/gcc-4.9.3/gcc-4.9.3.tar.bz2
tar xf gcc-4.9.3.tar.bz2 && cd gcc-4.9.3

./configure \
  --prefix=$HOME/bin/gcc-4.9.3
  --program-suffix=-4.9.3
make -j4 && make install
```

The `$HOME/bin` directory is already in my `PATH`, so I'm just going to create
a small symlink that will allow me to call `gcc-4.9.3` from anywhere.

    cd $HOME/bin
    ln -s gcc-4.9.3/bin/gcc-4.9.3 gcc-4.9
    ln -s gcc-4.9.3/bin/g++-4.9.3 g++-4.9

Be aware that compiled, GCC does take a fair amount of space:

    du -sh $HOME/bin/gcc-4.9.3
    1.1G	/home/slau/bin/gcc-4.9.3
    du -sh $HOME/src/foss/gcc-4.9.3
    5.5G	/home/slau/src/foss/gcc-4.9.3

So yeah, there's that to keep in mind.

[gcc-bug]: https://gcc.gnu.org/bugzilla/show_bug.cgi?id=66857
