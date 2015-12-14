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
more people and I get feedback about the errors I may have made.

<!-- more -->


