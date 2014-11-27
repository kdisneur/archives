---
layout: post
title: Rsync to the rescue of SCP
tags:
  - system
  - unix
  - tools
---
Do you ever try to download a big file over Internet? I do it sometimes to copy a database dump for example.
This kind of files can have a size over 1Gb and, at work, we have lot of troubles with our broadband. It means I have
really few chances my SCP command finish successfully.

I usually copied my files with this kind of command:

```bash
$ scp user@server:/a/path/to/a_big_dump.tar .
a_big_dump.tar 42%  612MB 443.2KB/s - stalled
```

And after trying 5 times and prying every god I have already heard about, I found another command. The kind of command you
can run and if your connection is stalled or you have to go out, you can kill it and resume it as soon as you have network
access.

```bash
rsync --rsh='ssh' --progress --partial user@server:/a/path/to/a_big_dump.tar .
receiving file list ...
1 file to consider
a_big_dump.tar
     2293760   0%    1.06MB/s    0:07:06
```

I can explain it even if it is really straightforward:

* `--rsh`: is just an option to specify you want to use the rsync daemon via a remote shell connection. Our remote shell
is SSH, the secure one.
* `--progress`: is another user friendly option to display the current progress of the transfer
* `--partial`: is **the interesting option**, it is the option that allows to resume a transfer instead of restarting from
the beginning.
