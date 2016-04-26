---
layout: post
title:  "Freebox automatic ruTorrent download"
date:   2016-04-26 18:00:00
categories: freebox
tags:
excerpt: >
  A couple of scripts I used to use so my Freebox would automatically download
  everything my seedbox had finished downloading.
---
I love Free. I think it is one of the best ISPs in the world, and even more so
in France. If I'm not mistaken, they invented the ISP-provided set top box that
seems so common place these days. Their [Wikipedia entry][wiki-free] explains
some of the reasons I like them so much, but to list a few:

- unmetered bandwidth,
- fixed IPv4 address, /60 IPv6 block,
- they allowed and encouraged people to run their own servers (reverse-IP
  customisation, every port available, etc),
- their router doesn't suck,
- fixed price policy: ~30 euros, all inclusive,
- the IPTV streams are viewable on your PCs through VLC,
- the list could go on for a long time.

All this being said, one of the major reason I loved their product so much was
because of their modem, the [Freebox][wiki-freebox]. I've been a long fan of
all their hardware (since the V4), but the Freebox Revolution takes the cake.

Anyway, back on topic. The Freebox Revolution Server allows you to set an RSS
feed, and all items in this RSS feed will be automatically downloaded by the
newsgroup/torrent/direct download client that is built into the router (yeah,
they have that built in).

I used to use a seedbox, but because I had a really bad DSL connection (4
Mbits), I preferred for my downloads to happen during the day---when nobody was
home---rather than having to painstakingly wait for the bits to transfer when I
was home. The other reason was also that downloading anything would immediately
saturate the link, and ruin browsing/YouTubing for anyone home at that point.
The other main point that should be stressed is that the Freebox Player (the
part connected to the TV) could stream any file stored on the Server's hard
drive (it acted as a NAS, out of the box). This meant that we were able to
watch the new GoT episode faster than most of my colleagues who had FTTH
connections, simply because we just had to turn the telly on, and click "Play".

I came up with the following script to generate an RSS feed of *downloaded*
files that my Freebox would then automatically pick up.

```python
#!/usr/bin/python

import cPickle as pickle
import xml.etree.cElementTree as et
# You can find PyRSS2Gen at:
# https://pypi.python.org/pypi/PyRSS2Gen
import PyRSS2Gen as RSS2
import urllib, mimetypes
from os import path, walk
from datetime import datetime

DIRECTORY = '/home/rtorrent/downloaded'
DATABASE = '/home/rtorrent/.rssdb'
EXTENSIONS = ['.avi', '.mkv', '.mp4', '.mpeg', '.mpg']
HOSTNAME = 'myseedbox.example.com'
PATH = 'downloads'
HTTPS = True
RSS_FILENAME = '/home/rtorrent/downloads.rss'

class FileList:
  def __init__(self, database):
    self.database = database
    self.load()

  def append(self, file, size, type):
    if file not in self.files:
      self.files[file] = {
        'date': datetime.now(),
        'size': size,
        'mimetype': type
      }

  def store(self):
    with open(self.database, 'w') as fp:
      pickle.dump(self.files, fp)

  def load(self):
    if path.exists(self.database):
      with open(self.database, 'r') as fp:
        self.files = pickle.load(fp)
    else:
      self.files = {}

  def get(self):
    """
    Return the files in reverse order in which we stored them.
    """
    return self.files

class DirectoryParser:
  def __init__(self, directory, extensions = []):
    self.directory = directory
    self.extensions = extensions
    mimetypes.init()

  def parse(self, store):
    for _path, directories, files in walk(self.directory):
      for file in files:
        if self.filter(file):
          type = self.get_filetype(file)
          full_path = path.join(_path, file)
          size = path.getsize(full_path)
          storeable_path = full_path[len(self.directory) + 1:]
          store.append(storeable_path, size, type)

  def filter(self, file):
    if len(self.extensions) is 0:
      return True

    elif path.splitext(file)[1] in self.extensions:
      return True

    else:
      return False

  def get_filetype(self, file):
    return mimetypes.types_map[path.splitext(file)[1]]

class RSSGenerator:
  def __init__(self):

    if HTTPS:
      self.url = 'https://'
    else:
      self.url = 'http://'
    self.url += '%s/%s/%%s' % (HOSTNAME, PATH)

  def generate(self, files):
    items = []

    for file in files:
      url = self.url % urllib.quote(file)
      item = RSS2.RSSItem(
        title = file,
        link = url,
        guid = RSS2.Guid(url),
        pubDate = files[file]['date'],
        enclosure = RSS2.Enclosure(url, files[file]['size'], files[file]['mimetype'])
      )
      items.append(item)

    self.rss = RSS2.RSS2(
      title = '%s downloads RSS feed' % HOSTNAME,
      description = 'The latest finished downloads at %s\'s ruTorrent' % HOSTNAME,
      link = self.url % '',
      items = items
    )

  def store(self, filename):
    with open(filename, 'w') as fp:
      self.rss.write_xml(fp)

if __name__ == '__main__':
  file_list = FileList(DATABASE)
  parser = DirectoryParser(DIRECTORY, extensions = EXTENSIONS)
  parser.parse(file_list)

  file_list.store()

  rss_generator = RSSGenerator()
  rss_generator.generate(file_list.get())

  rss_generator.store(RSS_FILENAME)
```

This was called by a cron job every 5 minutes:

```
# Generate the downloads.rss, every 5 minutes
*/5 * * * * /usr/local/bin/rssdirectory.py && /bin/sed -i 's!></enclosure!/!g' /home/rtorrent/downloads.rss
```

The reason that sed command is there is because the Freebox doesn't understand
the enclosure tag that is generated by PyRss2Gen. Yeah, that one took a while
to debug and figure out.

While this setup worked very nicely, every couple of months, the Freebox would
simply stop downloading new items. In the router's WebOS, I'd force updates to
the RSS feed, but nothing would show up. The feed would be empty. After some
time, I figured out that it was due to the Freebox not being able to cope with
the size of the RSS file. So here's the hack-ish remedy for that, you've
guessed it, another cron job:

```
# Remove the .rssdb once a week (garbage collection)
@weekly /bin/rm /home/rtorrent/.rssdb
```

This setup worked very nicely for the better part of 5 years, while we lived in
France.

So thank you Free, and all the engineers behind the amazing ISP that you are,
and all the engineers who make the Freebox possible. It's an amazing piece of
kit, and the hacker spirit that you guys show all over the place is
awe-inspiring, to say the least.

[wiki-free]: https://en.wikipedia.org/wiki/Free_%28ISP%29
[wiki-freebox]: https://en.wikipedia.org/wiki/Freebox
