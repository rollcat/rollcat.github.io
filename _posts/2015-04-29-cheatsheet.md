---
title: The Tech Cheat Sheet
layout: post
category: tech
---

For whenever I forget random silly things like "how to exit `vi`" (I'm an Emacs
guy).

## [ISO](https://en.wikipedia.org/wiki/List_of_International_Organization_for_Standardization_standards)'s

- [ISO 3166](https://en.wikipedia.org/wiki/ISO_3166-1#Current_codes)
  country codes
- [ISO 8601](https://en.wikipedia.org/wiki/ISO_8601) timestamps
    - `2015-04-29T15:30:00+02:00`

## HTTP

### RTFM

- [Status codes](https://en.wikipedia.org/wiki/List_of_HTTP_status_codes)
- [RFC 1945](https://tools.ietf.org/html/rfc1945) HTTP/1.0
- [RFC 6265](https://tools.ietf.org/html/rfc6265)
  [Cookies](/tech/nocookie.html)
- [RFC 7230](https://tools.ietf.org/html/rfc7230) Message Syntax and Routing
- [RFC 7231](https://tools.ietf.org/html/rfc7231) Semantics and Content
- [RFC 7232](https://tools.ietf.org/html/rfc7232) Conditional Requests
- [RFC 7233](https://tools.ietf.org/html/rfc7233) Range Requests
    - Request: `Range: bytes=21010-47021`
    - Response: `Content-Range: bytes 21010-47021/47022`
- [RFC 7234](https://tools.ietf.org/html/rfc7234) Caching
    - Disable: `Cache-Control: no-cache`
- [RFC 7235](https://tools.ietf.org/html/rfc7235) Authentication
    - Response: `WWW-Authenticate: Basic realm=example.com`
    - Request: `Authorization: Basic dXNlcjpwdw==` with `base64("user:pw")`

### `curl`

Dump headers too

    curl -D/dev/stderr http://httpbin.org/ip

Ignore SSL errors (watch out!)

    curl -k https://httpbin.org/ip

Continue interrupted download

    curl -C- -o file http://httpbin.org/ip

Basic auth

    curl -u user:pw http://httpbin.org/ip

## Sharing a bunch of files

### Simple HTTP with Python

    python -m SimpleHTTPServer 5380
    python3 -m http.server

### FTP

Save time - [use HTTP](#simple-http-with-python) to simply serve some
files, use [`scp` (aka `sftp`)](#sftp-aka-scp) to give authenticated
read-write access.

### SFTP aka `scp`

Amend `/etc/ssh/sshd_config`:

```
Subsystem sftp internal-sftp
Match Group sftpusers
    X11Forwarding no
    AllowTcpForwarding no
    ChrootDirectory /jail
    ForceCommand internal-sftp
```

Add the `sftpusers` group, add a default `sftp` user, create a jail
with data:

    groupadd sftpusers
    useradd -g sftpusers -d /jail/data -s /sbin/nologin sftp
    mkdir -p /jail/data && chown root:root /jail && chmod 755 /jail

## DNS

[Closest OpenNIC public access servers](http://wiki.opennicproject.org/ClosestT2Servers)

If you like Google:

    nameserver 8.8.8.8
    nameserver 8.8.4.4

### `dig`

- Quick lookups
    - `dig rollc.at +short`
    - `dig www.rollc.at +short`
    - `dig rollc.at MX +short`
- Chatty lookup
    - `dig rollc.at ANY +noall +answer`

### `bind`

Gentle restart:

    named-checkconf
    rndc reload

## `vim`

No Emacs cheat sheet. It's already wired in my brain, like breathing.

- Most importantly
    - `Esc` goes to command mode
    - [`:wq`](http://whatthecommit.com/95490aaa544d9c5738f3e797d1e72207):
      save and exit
    - `:q!`: "I don't even know what I've screwed up; exit and don't save"
- Basic editing
    - `i`: insert mode
    - `u`: undo
- Movement
    - `hjkl`: navigation
    - `^` `$`: jump do beginning/end of line
- Copypasta
    - `x` `X`: delete forward, backward
    - `d` + direction: cut
    - `p` `P`: paste forward, backward
- Search / replace
	- `/`: search
    - `n`: next
    - `:%s/a/b/g`: replace all `a`'s with `b`

## `git`

### Find the commit that broke something

[`git-bisect(1)`](https://linux.die.net/man/1/git-bisect) to the
rescue.

(Optional, but simplifies things a lot) Write a short script to test
for the broken thing. Save as `./test-broken`.

Go back to repo root and start bisecting:

    git bisect start
    git bisect bad

Go back to the last known working version, verify as good, mark as good:

    git checkout 1234abcd
    ./test-broken
    git bisect good

Option 1: if you wrote the script:

    git bisect run ./test-broken

Option 2: become human for-loop. Check if it works, and run either
`git bisect good`, `git bisect bad`, or `git bisect skip` until there
is only one suspect left.

When done:

    git bisect reset

### Rewrite history

[`git-rebase(1)`](https://linux.die.net/man/1/git-rebase) to the
rescue.

First off,
[find the commit that broke something](#find-the-commit-that-broke-something).

Start an interactive rebase, add `~1` after the commit hash to go one
commit earlier.

    git rebase --interactive badb33f5~1

Your editor will pop up. You'll see something like this:

    pick f00dc4f3 let's have coffee
    pick 1043d1d0 I love you
    pick badb33f5 break stuff

Go to the line with `badb33f5` and change the leading word `pick` to
`edit` (or `e`).
Save file and close it.

Fix the broken code. Did you write `./test-broken`? Run it.

Amend the broken commit:

    git commit --amend -m "don't break stuff"

Continue rebasing, fixing any merge conflicts on the way:

    git rebase --continue

### Rewrite published history

    git push --force

### Configure branch to track a remote

In `.git/config`:

```
[branch "master"]
	remote = origin
	merge = refs/heads/master
```

### Being a douche

Can be used with `--amend` on existing commits.

#### Impersonate someone else

    git commit --author "Kamil's Evil Twin <kamil@trollcat.example.com>"

#### Forge commit timestamp

    git commit --date=2015-04-01T00:00

## UNIX

### `Makefile`

"Phony" rules that never produce files:

    .PHONY: all lint run deploy clean
    .PHONY: *

Shellout:

    BIN	= ${shell find bin -type f -perm /u=x} ${shell find . -name "*.py"}

List of files with extension `.a` that must become files with extenion
`.b`:

    B_FILES = ${A_FILES:.a=.b}

More complex variable substitution is possible using `%` pattern
matching:

    DEST_FILES = ${SRC_FILES:%=${DEST}/%}

Make `.b` files from `.a` files and some other stuff:

```
%.b: %.a otherfile.c
    atob --in $< --out $@ --extra-c otherfile.c
```

Suppress echo:

```
run:
    @ ./dev_app.py
```

Ignore exit status:

```
clean:
    - rm .coverage
```

Current git branch as a slug:

    BRANCH_SLUG ?= $(shell git rev-parse --abbrev-ref HEAD | sed -E 's/[_/]+/-/g')

Print any makefile variable (use e.g. `make print-BIN`):

    print-%: ; @echo $*=$($*)

Remove all default suffixes (implicit rules):

    .SUFFIXES:

### `sed`

Change a line:

    sed -E 's/pattern/substitution/g'

Delete a matching line:

    sed -E '/pattern/d'

Backups:

    sed -i~

## RAID

Disks are cheap! **TLDR**: if you care about the data, use RAID1; if
you also care about performance, use RAID10; if you only need scratch
space or are tight on budget, RAID5 is adequate.

### 0

- Needs 2+ disks
    - 100% capacity
- Stripes `[AB...]`
    - `#yolo`
    - Best IO

### 1

- Needs 2+ disks
    - 50% capacity (or 1/N where N=mirrors)
- Mirrors `[A...]+A`
    - All but one disk can be lost
    - Meh writes, good reads
- Rebuilds are OK

### 10

- Needs 4+ disks
    - 50% capacity (or 1/N where N=mirrors)
- Mirrors and stripes `[AB...]+AB...`
    - Some disks can be lost (stripes must survive)
    - Good IO
- Rebuilds are OK
- What you usually want for busy and available services

### 5

- Needs 3+ disks
    - "`N - 1`" capacity
- Parity `[AB...]+P`
    - One disk can be lost
    - Crap writes, OK reads
- Rebuilds are costly and can result in 2nd disk failure; see RAID6
  for big arrays
- What you usually want if you can afford data loss or temporary
  unavailability

### 6

- Needs 4+ disks
    - "`N - 2`" capacity
- Extra parity `[AB...]+PR`
    - Two disks can be lost
    - Super crap writes, OK reads
- Rebuilds are costly; somewhat mitigates risk of 2nd disk failure for
  big arrays

## Email

### Important headers

```
X-Priority: 1 (Highest)
X-MSMail-Priority: High
Importance: High
```

### With `mail` / `sendmail`

```
$ sendmail -t
From: me
To: test@example.com
Subject: Subject

Message

.
```

```
$ mail -s "Subject" test@example.com
Message

.

```

### Reading `mbox` files, for humans

    mutt -f ./mbox

## Discovering things

Did it ever occur to you that you've been restarting a machine so
rarely that you forgot which bootloader it had installed?

    dd if=/dev/sda count=512 | strings | grep -oiE 'grub|lilo'

## Stupid hacks

### `furl`

For some very odd and troubling reason, stuck without `curl`, `wget`,
or any other HTTP client?

```
furl() {
    host=$1
    url=$2
    printf "GET $url HTTP/1.1\r\nHost: $host\r\nConnection: close\r\n\r\n" |\
            nc $host 80
}
```

    furl example.com /index.html

## Layer 2 networking

- [Private networks](https://en.wikipedia.org/wiki/Private_network)
    - IPv4: ([RFC1918][])
        - `10.0.0.0/8` - single block `/8` (A class)
        - `172.16.0.0/12` - 16 blocks, each `/16` (B class)
        - `192.168.0.0/16` - 256 blocks, each `/24` (C class)
    - IPv6: `fd00::/8`
- Loopback network
    - IPv4: `127.0.0.0/8`
    - IPv6: `::1/128`
- [Link-local (unicast)](https://en.wikipedia.org/wiki/Link-local_address)
    - IPv4: `169.254.0.0/16` - [RFC3927][]
    - IPv6: `fe80::/10` (but use `fe80::/64`!) - [RFC4862][],
      [RFC4941][]
- [Multicast](https://en.wikipedia.org/wiki/Multicast_address)
    - IPv4: `224.0.0.0/4`
    - IPv6: `ff00::/8`

[RFC1918]: https://tools.ietf.org/html/rfc1918
[RFC3927]: https://tools.ietf.org/html/rfc3927
[RFC4862]: https://tools.ietf.org/html/rfc4862
[RFC4941]: https://tools.ietf.org/html/rfc4941
