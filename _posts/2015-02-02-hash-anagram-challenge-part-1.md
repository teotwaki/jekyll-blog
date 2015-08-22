---
layout: post
title: "Hash anagram challenge, part 1"
date:  2015-02-02 21:06
categories: anagram challenge
tags:
---
As I mentionned previously, I've been working on a technical challenge I found
a few days ago. Just to be clear: this is a low-priority challenge for me. I
don't expect anything out of this, and I don't consider it to be anything more
than yet another toy-project to keep my mind busy whenever I have a moment to
spare.

[The challenge][challenge] is as follows:

> What is the anagram of "poultry outwits ants" whose md5 hash computes down to
> `4624d200580677270a54ccff86b9610e`?

Sounds simple enough, right? That's what I thought as well. Let's start by
doing some math:

There are 18 letters in the seed anagram, not including spaces. Let's leave the
spaces aside for the moment and just focus on those 18 letters. How many
permutations of letters are there? Well, that's fairly easy, let's head up to
[WolframAlpha][wolfram] and calculate some [factorials][wiki-factorial]:

    18!
    => 6 402 373 705 728 000

Just shy of [6.5 quadrillion][wolfram-18] possibilities? Not bad. Now, let's
see what throwing in those spaces changes...

    21!
    => 51 090 942 171 709 440 000
    
    23!
    => 25 852 016 738 884 976 640 000

Between [51 quintillion][wolfram-21] possibilities and [25 sixtillion
possibilities][wolfram-23], assuming there are respectively 3 to 5 words in the
actual anagram. Wow. We now know how many possible permutations there are.
These are purely permutations (e.g.: theoretical anagrams), and not actually
matched to real, existing words. That's another step. Let's just go with these
numbers and see how long it would take to calculate all of these.

How fast is my computer? I'm writing this on a small, cheap netbook that runs
off battery power. I'm not expecting much out of it. Let's see what [John the
ripper][john] thinks of my CPU.

    $ john -test | grep -A md5
    Benchmarking: md5crypt [MD5 32/64 X2]... DONE
    Raw:	5383 c/s real, 5427 c/s virtual

I'm taking md5crypt as a low baseline, as I'm guesstimating it to be more
computationally intensive than just calculating the md5 sum of small strings.
Let's assume I get a thousand boxes like mine, slightly more powerful (four
cores instead of two):

    5000 hashes/s * 1 000 000 * 4 cores
    => 20 000 000 000

20 billion hashes per second, that's pretty good, and in our era of cloud
computing, surely not that impossible to achieve. How long would it take me to
crack the challenge, by simple, honest to goodness bruteforce?

    21! / (5000 * 4 * 1 000 000 per second)
    => 709 596 hours
    => 29 567 days

So, assuming perfect workload distribution, and assuming no overhead from the
software, it would [take around 80 years][wolfram-80-years] to calculate the
md5 hash of every single possible anagram. Now, I realise those are very, very
low values; maybe more worthy of vapour that cloud computing. [According to
Andrew Dunham][dunham], it is possible to get up to 2.3 billion md5 hashes per
second per Amazon EC2 GPU cluster. So assuming an hourly rate of USD 0.7, it
would only cost [under USD 5 million][wolfram-5-million] to calculate
everything, but it would still [take nearly as long][wolfram-9-months] as to
put a human on this planet. Note that those numbers are for 1000 GPU clusters;
The reason I didn't go for the million-figure in this estimate is because I
somehow strongly doubt the fact that Amazon has a spare million GPU clusters
lying around.

I love a good challenge as much as the next guy, but I don't have that kind of
time or money to throw at just something to keep my brain idling. So where do
we go from now? Let's look at the challenge again. Hey, they gave us a [list of
words][wordslist]! That *must* be useful, right? Of course it is!

How many words did they give us?

```bash
$ wc -l wordlist
99174 wordlist
```

Wait, how many unique words did they give us?

```bash
$ sort -u wordlist | wc -l
96316
```

OK, so let's remove any word that contains a letter that doesn't exist in our
seed anagram: *poultry outwits ants*. Time to let go of bash and friends, and
call in the big guns: Python.

```python
#!/usr/bin/env python

from __future__ import (absolute_import, print_function,
                        unicode_literals)

import os.path
import urllib2


"""
Unoptimised file downloader with minimal cache management. It expects a
newline-delimited file and removes any line that contains letters not found in
an arbritrary string
"""


def fileFromHTTP(url, filename):
    """I download a file and save it to the provided filename"""

    print("Downloading file:", url)
    file = urllib2.urlopen(url).read()

    with open(filename, 'wb') as fp:
        fp.write(file)


def fileToList(file):
    """I know how split a file into a list(). Yay me."""

    print("Splitting file into list()")
    lines = [x.decode('utf-8') for x in file.splitlines()]
    return lines


def getEnglishWords(url):
    """I check if the cache is warm, and download the file if it can't be found
    locally"""

    filename = os.path.basename(url)

    if not os.path.isfile(filename):
        fileFromHTTP(url, filename)

    with open(filename, 'rb') as fp:
        file = fp.read()

    return fileToList(file)


def filterWords(haystack, needles):
    """I take a a list of words, and remove any that can't be found in the
    needles string"""

    print("Removing words that contain letters not found in anagram")
    print("Original set of words length:", len(haystack))

    # The list(set()) operation is done to remove dupes
    haystack = list(set(haystack))

    for word in list(haystack):
        if not existsInPool(needles, word):
            haystack.remove(word)

    print("Filtered set of words length:", len(haystack))
    return haystack


def existsInPool(pool, word):
    """I check whether `word` could possibly exist in `pool`"""
    pool = list(pool)

    try:
        for c in word:
            pool.remove(c)
    except ValueError:
        return False

    return True


def main():
    words_url = 'http://followthewhiterabbit.trustpilot.com/cs/wordlist'
    anagram = 'poultryoutwitsants'

    words = getEnglishWords(words_url)
    words = filterWords(words, anagram)


if __name__ == '__main__':
    main()
```

Let's give it a run and see how many words we managed to squash out:

```bash
$ python foo.py
Splitting file into list()
Removing words that contain letters not found in anagram
Original set of words length: 99174
Filtered set of words length: 1658
```

OK, we managed to remove 98.32% of the words list provided by Trustpilot. This
definitely reduces the amount of calculations we're going to have to perform,
but it doesn't really make things that much easier. Let's look at why.

The usual way to calculate the number of permutations will again be based
around factorials, as we used previously. As before, let's assume the target
anagram is made of 3 words, just like the seed anagram. What would be the
formula to calculate that? Well actually, the [Python
documentation][itertools-permutations] gives it right to us:

    # for n the pool length, and r the number of permutations:
    n! / (n-r)!

So applied to our problem:

    1658! / (1658 - 3)!
    => 4 549 538 736
    
    # 18 bytes per anagram, 3 bytes for spaces, 32 bytes for the md5 hash
    4549538736 * (18 bytes + 3 bytes + 32 bytes)
    => 241 125 553 008

I guess it would be possible to load up a Cassandra cluster with 240GB of data
and use Apache Spark to carve through it, but it doesn't really seem like the
right way to do it.

In the next article ([part 2][blog-part-2]), we'll write a quick python script
that starts crunching through the list of words and find a way to discard
impossible permutations.

[blog-part-2]: https://blog.wedrop.it/anagram/challenge/2015/02/03/hash-anagram-challenge-part-2.html
[challenge]: http://followthewhiterabbit.trustpilot.com/cs/step1.html
[dunham]: http://du.nham.ca/blog/posts/2013/03/08/password-cracking-on-amazon-ec2/
[john]: http://www.win.tue.nl/~aeb/linux/john/john.html
[itertools-permutations]: https://docs.python.org/2/library/itertools.html#itertools.permutations
[ssl-for-everyone]: https://blog.wedrop.it/free-ssl-for-everyone.html
[wiki-factorial]: http://en.wikipedia.org/wiki/Factorial
[wolfram]: www.wolframalpha.com
[wolfram-18]: http://www.wolframalpha.com/input/?i=18!
[wolfram-21]: http://www.wolframalpha.com/input/?i=21!
[wolfram-23]: http://www.wolframalpha.com/input/?i=23!
[wolfram-80-years]: http://www.wolframalpha.com/input/?i=21!+%2F+%28%285000+*+4+*+1000000%29+per+second%29
[wolfram-5-million]: http://www.wolframalpha.com/input/?i=21!+%2F+%281000+*+2.3+billion+per+second%29+*+%281000+*+%240.7+per+hour%29
[wolfram-9-months]: http://www.wolframalpha.com/input/?i=21!+%2F+%281000+*+2.3+billion+per+second%29
[wordslist]: http://followthewhiterabbit.trustpilot.com/cs/wordlist
