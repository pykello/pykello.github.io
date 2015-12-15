---
layout: post
title: "cstore_fdw and 'Files are Hard'"
category: programming
---

I recently came accross the "[Files are hard][files-are-hard]" article, and it
made me wonder how reliable is cstore_fdw's design and implementation.

[cstore_fdw][cstore-fdw] is a columnar store for PostgreSQL that I designed and
developed in my previous job at Citus Data.

I am writing this post so my decisions for cstore_fdw's design get reviewed by
more people, and I get some feedback and improve the design.

<!-- more -->

## File Structure

Structure of cstore\_fdw is similar to [Hive's ORC][orc], but with some differences.
One of the biggest differences is that cstore\_fdw stores metadata in a separate
file, while ORC stores it at the end of data file.

The reason for this difference? It made avoiding data corruption much easier, as
we will se later.

Each cstore\_fdw table consists of two files:

* Data file
* Metadata file


### Data File

cstore\_fdw divides rows into stripes. Each stripe is 150K rows by default. For
each stripe data is stored in column oriented manner. 

   ![cstore-layout](/images/cstore-layout.svg)


### Metadata File

Metadata file is a small file which contains some version infomration, plus
outline of the data file. This includes, location and size of each stripe.


## Operations

Currently the only modification operation that cstore\_fdw supports is batch
insert, which can be triggered using 

{% highlight sql %}
COPY cstore_table FROM '/path/to/file.csv' WITH CSV;
{% endhighlight %}

or 

{% highlight sql %}
INSERT INTO cstore_table FROM SELECT * FROM some_other_table WHERE some_condition
{% endhighlight %}

Our plans were to first get batch inserts right, as this on its own was very
useful to our customers who used it to store archive data. Then add support
for single row inserts, deletes, and updates.

What does happen when we do a batch load?

- For every 150K rows construct a new stripe and append it to the data file.
  While doing this, never touch the metadata file.
- Sync and close the data file.
- Create a temporary metadata file, and write the metadata which represents the
  newly inserted data to it.
- Sync and close the temporary metadata file.
- Rename the temporary metadata file to the main metadata file.

If the system crashes during the first four steps, the table is still in a
consistent state and we won't have data corruption.

This is because:

- We don't change the main metadata file in these steps.
- We don't overwrite or move the old data, in the data fil.

What about 5th step? The nice thing here is that the [specification][rename]
of the rename system call requires it to be automic. So this step either is done
in full (in which case we'll see the new data) or isn't done at all (in which
case we'll see the old data, but won't have data corruption).

And that is the reason for using a separate file for metadata. I needed an
atomic operation to rely on, and rename is atomic. Rewriting the whole data
in a new file and doing the rename for data file wasn't an option, since it is
very expensive. So instead, we rewrite the small metadata file and do the rename
for that.

Of course there were some other designs to avoid file corruption, but I found
this design very simple compared to other designs.

Reading the [discussion in hacker news][hn] for "Files are hard", I see that
I've missed a step, which is doing a sync on the parent directory to make the
rename durable. That's something we should fix, but I'm happy that our design
seem to avoid most of the issues discussed in "Files are hard".

[files-are-hard]: http://danluu.com/file-consistency/
[cstore-fdw]: https://github.com/citusdata/cstore_fdw/
[orc]: https://cwiki.apache.org/confluence/display/Hive/LanguageManual+ORC
[rename]: http://pubs.opengroup.org/onlinepubs/009695399/functions/rename.html
[hn]: https://news.ycombinator.com/item?id=10729132