---
layout: post
title:  "Hash anagram challenge, part 2"
date:   2015-02-03 20:40
categories: anagram challenge
tags:
---
This is part two of my attempt to solve the Hash Anagram challenge offered by
TrustPilot. Full disclosure: These articles are not written in chronological
order; I finished the challenge a few weeks ago. I'm merely documenting my
mental process, and trying to get back into proper writing. In this part, we'll
start writing a small script that crunches the words, and do some rough
benchmarking on it, to see whether our approach actually makes sense or not.

In [part 1][blog-part-1], we saw that we have a couple thousand words to work
with. We also saw that working from the letters in the seed anagram alone,
there were a few quintillion possible permutations.

So what kind of algorithms could we apply to this problem, to try and solve it?
Let's try to restate the problem in order to see things more clearly:

> Given a list of 1758 words, find a permutation of *n* words whose md5 hash
> computes to `4624d200580677270a54ccff86b9610e`.

First, let's lay down some observations:

1. There are 18 letters in the seed anagram, hence the number of letters in the
   target anagram will be 18 as well;
2. We don't know how many words the target anagram contains;
3. We don't know how many permutations of the words list are actual anagrams.

Technically, we could assume that the target anagram contains nothing more than
three words, but I don't want to make that assumption.

I see two main algorithmic paths:

- Generate all possible word permutations, and filter it down;
- Recursively append words to one another until an anagram is found.

So how do you check whether a string is an anagram of another string? I've found
the following algorithm in Python:

```python
def isAnagram(first, second):
    # If the strings are identical, they are anagrams
    if first == second:
        return True

    # Remove spaces, and convert `first` into a list
    first = first.translate({ord(' '): None})
    second = second.translate({ord(' '): None})

    # If they don't have the same length, they can't be anagrams
    if len(first) != len(second):
        return False

    else:
        first = list(first)

        try:
            # Remove all characters found in second
            map(first.remove, second)
        except ValueError:
            return False
        else:
            return True
```

And an equivalent implementation in C:

```cpp
// Return a copy of the letters a string (without spaces)
char * lttrdup(char const * str) {
  // Technically, we're wasting a few bytes here, but that's OK.
  char * copy = malloc(strlen(str) + 1);
  char * tmp = copy;

  do {
    if (*str != ' ')
      *tmp++ = *str;
  } while (*str++ != '\0');

  return copy;
}

bool is_anagram(char const * first, char const * second) {
  // If the strings are equivalent, they are anagrams
  if (strcmp(first, second) == 0)
    return true;

  else {
    // Remove the spaces from our strings
    char * cpy_first = lttrdup(first);
    char * cpy_second = lttrdup(second);

    // tmp_second is an iterator over cpy_second
    char * tmp_second = cpy_second;

    // If both strings don't have the same length, they can't be anagrams
    if (strlen(cpy_first) == strlen(cpy_second)) {

      // Iterate over cpy_second until the end is reached
      while (*tmp_second != '\0') {
        // find the character pointed by tmp_second in cpy_first
        char * match = strchr(cpy_first, *tmp_second);

        // if a match is found, overwrite it, and move on the to the
        // next character
        if (match != NULL) {
          *match = '.';
          tmp_second++;
        }

        // no match found, skip to cleanup
        else
          break;
      }

    }

    // if tmp_second doesn't point to the end of the string, we found a
    // character that doesn't exist in first, hence, no anagram
    bool result = *tmp_second == '\0';

    // cleanup
    free(cpy_first);
    free(cpy_second);

    return result;
  }
}
```

The main issue I see with both implementations is that we actually need to do
allocations in order to verify that two strings are anagrams of each other.
Granted, the allocations in C are way smaller and more manageable than the
Python ones, but still, not great. If you know of an algorithm that doesn't
require allocations, I'd be happy to hear about it.

Talking about allocations, what could we do to reduce their numbers? At the
moment, the only check we do before undergoing any allocation, both in Python
and C, is checking whether both strings are the same. How could we cut that
down? This is my train of thought:

Characters are encoded into their ASCII value, and for lowercase letters in the
alphabet, their value will range between 97 and 122. What happens if we add the
values of our seed anagram (letters only, no spaces)?

```python
anagram = "poultry outwits ants".translate(None, ' ')
sum(map(ord, anagram))
=> 2036
```

Regardless of the order of the letters, an anagram of "poultry outwits ants"
will always have the same order point sum. The reverse, however, isn't true: a
string that has an order point sum value of 2036 isn't necessarily an anagram of
"poultry outwits ants". Let's throw that into our code:

```python
def letterCount(word):
    return sum(map(lambda x: 1 if x != ' ' else 0, word))

def sumLetters(word):
    return sum(map(lambda x: ord(x) if x != ' ' else 0, word))

def isPlausibleAnagram(first, second):
    return letterCount(first) == letterCount(second) and \
        sumLetters(first) == sumLetters(second)
```

Excellent. Let's try to do the same in C:

```cpp
bool is_plausible_anagram(char const * first, char const * second) {
  unsigned int first_length = 0;
  unsigned int second_length = 0;
  unsigned int first_ord = 0;
  unsigned int second_ord = 0;

  do {
    if (*first != ' ') {
      first_ord += *first;
      first_length++;
    }
  } while (*first++ != '\0');

  do {
    if (*second != ' ') {
      second_ord += *second;
      second_length++;
    }
  } while (*second++ != '\0');

  return first_length == second_length && first_ord == second_ord;
}
```

I decided to write the C version as a single function, so as to only have to
loop over the two strings once. The Python version uses a more functional
approach.

Great, so we now have a way to approximate whether a string might be an anagram
without doing any allocations, and a way to know whether a string really is an
anagram of another string. Let's put together a quick script that gives us all
the permutations and starts pumping out anagrams, but first, let's use our
newfound quicktester to spare some CPU cycles:

```python
def isAnagram(first, second):
    if not isPlausibleAnagram(first, second):
        return False

    # If the strings are identical, they are anagrams
    if first == second:
        return True

    first = first.translate({ord(' '): None})
    second = second.translate({ord(' '): None})

    # If they don't have the same length, they can't be anagrams
    if len(first) != len(second):
        return False

    else:
        first = list(first)

        try:
            # Remove all characters found in second
            map(first.remove, second)
        except ValueError:
            return False
        else:
            return True
```

And now for the main function:

```python
def main():
    words_url = 'http://followthewhiterabbit.trustpilot.com/cs/wordlist'
    anagram = 'poultryoutwitsants'

    words = getEnglishWords(words_url)
    words = filterWords(words, anagram)

    length = 0
    while True:
        length += 1
        print("Testing permutations of %d words" % length)
        generator = itertools.permutations(words, length)

        for permutation in generator:
            permutation = ' '.join(permutation)
            if isAnagram(permutation, anagram):
                print("--> Found anagram:", permutation)
```

I let this run for a few minutes, and as luck would have it, it actually spat
out the answer:

```
python permutations.py
Splitting file into list()
Removing words that contain letters not found in anagram
Original set of words length: 99174
Filtered set of words length: 1658
Testing permutations of 1 words
Testing permutations of 2 words
Testing permutations of 3 words
--> Found anagram: want spit tortuously
--> Found anagram: want pursuit tolstoy
--> Found anagram: want tortuously spit
--> Found anagram: want tortuously pits
--> Found anagram: want tortuously tips
--> Found anagram: want pits tortuously
--> Found anagram: want tolstoy pursuit
--> Found anagram: want lousy outstript
--> Found anagram: want yous trustpilot
--> Found anagram: want trustpilot yous
```

Wait, `trustpilot`? Would the answer just be "*trustpilot wants you*"? Nevermind,
I haven't tested any md5 hashes or anything, so I won't cheat. The major issue
with using `itertools.permutations` is that it's pretty difficult to run on
a cluster, and looking at the speed, it's not going to complete quickly. What I
mean by speed is that a quick benchmark shows that the Python script running on
my netbook is capable of rougly 50k permutations per second. Going back to the
number crunching we did in part 1, how many different permutations are there for
*n* ranging from 1 to 5?

    1658! / (1658-1)! + 1658! / (1658-2)! + 1658! / (1658-3)! + \
        1658! / (1658-4)! + 1658! / (1658-5)!
    => 12 461 304 888 660 100

I don't even need to calculate how long that would take to know it would be a
very, very long time (hint: they should've started back in the [Copper
Age][wolfram-copper]). Now obviously, I wouldn't need to test *every single*
permutation; especially if I check the md5 hash every time I find an anagram.

So how can we split the work across multiple cores? I know just the library for
the job: [ZeroMQ][zeromq]. ZeroMQ (also written &Oslash;MQ) is a great tool to have on
your belt. It doesn't require much, other than a good ol' paradigm shift (POP!).

![Dilbert: Paradigm](/assets/article_images/2015-02-03-hash-anagram-challenge-part-2/dilbert-paradigm.gif)

ZeroMQ makes multithreading and multiprocessing easy. Well, it doesn't, it just
makes communicating between multiple threads, processes or servers a piece of
cake. The [PyZMQ bindings][pyzmq] are excellent, and they make everything an absolute
pleasure to work with. ZeroMQ takes care of managing the sockets, establishing
or receiving connections, polling and whatnot. Instead of working with dumb
sockets, ZeroMQ offers pre-defined socket types that behave in certain ways
(request/response, router, publish/subscribe, etc.).

Here's the architecture we'll deploy, simple and concise:

- The server will take care of downloading and reducing the wordlist;
- When a client asks for a word, the server will sequentially go through its
  words list, and send the next word to a client;
- Upon exhausting all possibilities for a given word, the client will send back
  all the anagrams it found, and request a new word.

Later on, we'll teach clients to also switch to md5 computing tasks if there are
any. This will allow us to not spend the next 8000 years computing, but also
completing the challenge.

ZeroMQ gives us a socket we can talk on between the clients and the server, but
what dialect should we employ? These days, I usually default to JSON; it's very
well supported and can encode nearly all the types of data one usually needs. It
compresses fairly well, and isn't too verbose, yet remains human-readable. And
hey look! PyZMQ offers convenience functions to read/write in JSON directly!

The other question we should probably answer is "What kind of sockets are we
going to use?" For the sake of simplicity, we're going to use one of ZeroMQ's
simplest socket types: the Request-Response combo. Let's take a moment to define
the grammar of our dialect:

    --> {'type': 'initialise'}
    <-- {'seed': 'poultryoutwitsants', 'words': ['a', 'abaci', 'aback', ... ]}
    
    --> {'type': 'next_word'}
    <-- {'word': 'truants'}
    
    --> {'type': 'found_anagrams', 'result': ['truants outwit slopy', ...]}
    <-- {}

Let's start by creating our server. Most of the stuff will remain the same,
we're just going to add a small class that keeps track of the current word, and
stuff like that:

```python
class WordsList(object):
    """I download a list of words, clean it up and iterate through it."""

    def __init__(self, filter, url):
        self._words = getEnglishWords(url)
        self._words = filterWords(self._words, filter)
    # A copy for future reference
    self._all_words = list(self._words)

    def nextWord(self):
        """I give the next word in the words list, or None if the end has been
        reached"""

        try:
            return self._words.pop()
        except IndexError:
            return None

    def getWords(self):
        """I give access to the whole words list."""
        return self._all_words

    def endReached(self):
        """I'm a helper function that tells you whether the end of the list has
        been reached or not"""
        return len(self._words) == 0
```

And this is again where I'm going to call it a night ;) I'll finish writing this
article tomorrow or something. Thanks for reading, anyway.

[blog-part-1]: https://blog.wedrop.it/hash-anagram-challenge-part-1.html
[wolfram-copper]: http://www.wolframalpha.com/input/?i=%281658!+%2F+%281658-1%29!+%2B+1658!+%2F+%281658-2%29!+%2B+1658!+%2F+%281658-3%29!+%2B+1658!+%2F+%281658-4%29!+%2B+1658!+%2F+%281658-5%29!%29+%2F+%2850000+per+second%29
[pyzmq]: http://zeromq.github.io/pyzmq/
[zeromq]: http://zeromq.org/
