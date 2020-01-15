---
title: UUIDs as invoice numbers - abusing a spec for fun and profit
layout: post
---

I'm a big fan of [UUIDs][uuid]. They make life better, wherever I need to
organise things - correlating objects between vastly different data sources,
storage formats, structures, non-structures, databases, caches, etc. I already
use them as PKs in Postgres, filenames in S3, and many more; today I wanted to
spread their usage to accounting.

[uuid]: https://en.wikipedia.org/wiki/Universally_unique_identifier

At [Intio](https://www.intio.tv/), we have an internal rule, that all objects
(wherever reasonable) should be addressed with a UUID, which then must be kept
consistent across all systems / services. So, whenever an object crosses a
service boundary (e.g. a row in a database, directly related to an asset in S3),
there is no confusion about the relationship - this is the video file, and this
is its metadata: encoding parameters, number of user impressions, edit history,
etc. The benefits are not always immediately obvious; for example: when we
started tackling business intelligence, having these same UUIDs stored in our
access logs saved us from an extra layer of indirection in our analytical
queries.

This is great, as long as other parties you communicate and exchange data with,
are willing to accept this schema. And perhaps the most important kind of party
a company interacts with, are their customers. We already label each partner
with a UUID - so can we take it a step further, and (ab)use UUIDs as invoice
numbers?

An invoice number, to be practical, should be readable and meaningful to humans.
I'd like it to sort chronologically, and perhaps also to include some metadata,
such as the customer ID, currency, whether it corrects an existing invoice, etc;
and, ideally, to have the most important metadata exposed to the naked eye.

## The structure

Let's look at a UUID:

    858319b7-b708-4c8b-b4d7-7c4af0a13e2b

This pattern of hyphens already suggests, that some of these sections could be
abused to squeeze in our own metadata! Maybe we could do something like this, to
encode date and time?

    20200515-1245-0000-0000-000000000000
    dddddddd-tttt-xxxx-xxxx-xxxxxxxxxxxx

So I've started my research. While a UUID looks completely random, there is a
certain structure to the string of hex digits. If you're interested, you should
read [RFC 4122][] and the [Wikipedia article][uuid], but the TL;DR is there are
four variants, with the common (RFC 4122) subdivided into versions, like so:

    xxxxxxxx-xxxx-Mxxx-Nxxx-xxxxxxxxxxxx

[RFC 4122]: https://tools.ietf.org/html/rfc4122

Here, `N` takes the value of `8`, `9`, `a` or `b` to indicate the RFC 4122
variant. Then, we look at `M`; versions 1 through 5 are currently in use (we'd
rather not use a version reserved for future use). So, at least theoretically,
all of the `x`'s are up for grabs, no?

Digging further, versions 1 and 2 already encode the timestamp, but there is
nothing human readable about that:

    089b6606-37db-11ea-ae33-0221860e9b7e
    186b9fd8-37db-11ea-ae33-0221860e9b7e
    21628d54-37db-11ea-9e87-0221860e9b7e

These can be actually decoded, using software (or pen and paper, for the
patient), in which case the `x`'s have a defined meaning: a timestamp, and a MAC
address. We don't want our abuse to clash with someone else's (like our future
selves) intended use, somewhere down the line.

Versions 3 and 5 are explicitly defined to take the result of a hash function as
an input while being generated; theoretically, this could allow us to make up
the concrete values of our `x`'s as we wish. This usage however, would again,
clash with the intended purpose: UUID 3/5 can be used to derive IDs, which
sounds like a very useful property: things such as domain names, URLs, or other
kinds of "foreign" identifiers can be brought to the "UUID space" in a
deterministic manner.

This leaves us with version 4, which is defined as: just random data. It sounds
like the perfect target.

```python
import datetime, random, uuid

def randch(n: int, a: str = "0123456789abcdef") -> str:
    return "".join(random.choice(a) for _i in range(n))

def invoice_uuid(now=None) -> uuid.UUID:
    now = now or datetime.datetime.now()
    return uuid.UUID(
        "{date}-{time}-4{r1}-{r2}{r3}-{r4}".format(
            date=now.strftime("%Y%m%d"),
            time=now.strftime("%H%M"),
            r1=randch(3),
            r2=randch(1, "89ab"),
            r3=randch(3),
            r4=randch(12),
        )
    )
```

This generates a UUID that looks like this:
`20200115-1245-473a-85ca-8845cc45ac6c`.

Can we cram in more metadata? Say we have a customer, with number
`ae0e9a50-d2f5-593c-81a7-0931bb11a33a` (in case you were wondering, this is the
UUID5 derived from `intio.tv` in the DNS namespace). Maybe we can take one of
the sections, and copy them directly into our invoice UUID? This way, if we see
several invoice numbers (without any other context), all ending in
`0931bb11a33a`, we know they were issued to a particular customer. It takes one
copy-paste into the search box in the invoicing software to filter and find the
customer. We can still have 281474976710656 customers before there's a
collision.

## Performance considerations

Since we're considering using these UUIDs as primary keys, we must consider the
indexing performance of this scheme. When you look at UUID1, this seems like the
exact reason behind the timestamp encoding; run this Python code and take note
of the output:

```python
import time, uuid
for _i in range(10):
    print(uuid.uuid1())
    time.sleep(1)
```

The prefix is definitely related to current time - it changes regularly, and
more often than the rest of the identifier. I don't have a source to quote here,
but whoever came up with this idea, must've had indexes in mind.

<details markdown="1">
<summary>(But what are indexes (or indices)?)</summary>

It's the secret sauce that allows a search query to be faster. Imagine a library
full of books; they may lay on the shelves, physically divided e.g. into
categories, But what if you needed to find the 10 oldest books in the
collection?

If the librarian kept an index of all books, sorted by their publication date,
now it becomes a matter of looking up the titles there, instead of looking at
each and every single book in succession. By building an index, we traded a tiny
bit of extra up-front work, and got much faster search in return.

This makes a lot of sense, whenever you have a collection that is read/searched
way more often than it is updated. Books are read much more often than they're
written.

</details>

There is approximately an infinity of methods to implement an index, and many of
these will rely on looking at a prefix of some value.

For example, to implement revision history, [git][] keeps a hash of the
historical contents of each file it tracks; these hashes are then used as file
names; these files then store the concrete contents (this is called
[content-addressable storage][cas]).

[git]: https://www.git-scm.com/
[cas]: https://en.wikipedia.org/wiki/Content-addressable_storage

Since looking at a very long list of file names in a directory can be slow (it
has to be scanned one-by-one), git groups them into subdirectories, using a
two-digit prefix. Looking at that short list of prefixes is a very fast
operation; then we need one more fast lookup to find the actual object. Now what
would happen, if all of these hashes had an identical two-digit prefix? The
entire scheme would be defeated, and the performance of many operations would
suck horribly.

Many other systems can be implemented this way; for example, in a SQL database,
an index on a character string field can be made to take into account only the
first N characters - which makes sense in some cases: the index is smaller,
faster to build, still achieves acceptable performance, etc.

It is imaginable, that relying on the prefix of a typical UUID value being
"sufficiently random", someone might come to a conclusion, that it might be
beneficial to keep a more compact index.

My gut feeling though is, your biggest user-visible performance bottleneck is
still going to be the loading time of that 6MB blob of JavaScript.
