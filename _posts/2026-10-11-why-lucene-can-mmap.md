---
title: "Why Lucene Can Use mmap and Most Databases Can't"
date: 2026-10-11 10:00:00 +0530
categories: [Search, Lucene]
tags: [lucene, mmap, page-cache, storage-engine, search-infrastructure]
description: "Lucene serves reads through mmap, but Postgres and MongoDB can't. Let's find out why."
---

<style>
.svg-fig{margin:1.5rem 0;padding:16px;border:1px solid #e5e7eb;border-radius:10px;background:#fcfcfd;}
.svg-fig svg{width:100%;height:auto;display:block;font-family:-apple-system,Segoe UI,Helvetica,Arial,sans-serif;}
.svg-fig figcaption{color:#6b7280;font-size:14px;margin-top:10px;font-family:-apple-system,Segoe UI,Helvetica,Arial,sans-serif;}
</style>

We know Lucene reads its index through `mmap`. The file is mapped into memory and reads are served straight from the OS page cache. No system call per read, no copy, and a page is loaded only when someone touches it. It's a big reason Lucene is fast.

So the natural question is: why don't databases like Postgres or MongoDB do the same? They also jump to specific pages through indexes instead of reading whole files. The read pattern is basically the same.

But mmap is not a magic solution that fits every storage engine. The difference is not in how they read. It's in **whether their data ever changes after it's written.**

## 1. Write-once vs in-place updates

- **Most databases** edit existing pages **in place**. If a balance changes, the same bytes on the same page are overwritten.
- **Lucene never modifies a file after writing it.** Even its writes don't go through mmap. New files are written with ordinary file writes and only then mapped *read-only*.

Everything else in this post follows from this one difference.

## 2. Write-once by design: Lucene's packed format

Lucene's files are packed so tightly that editing them in place is not practical.

Let's take the word `laptop`. Its posting list is the sorted list of documents that contain it: `3, 7, 42, 108`. Lucene doesn't store these numbers as they are. It stores the **gaps** between them, `3, 4, 35, 66`, because small numbers take fewer bytes. The lists for other words sit right behind it in the same file.

Now let's say doc 42 is edited and no longer mentions "laptop":

<figure class="svg-fig">
<svg viewBox="0 0 720 250" role="img" aria-label="Removing one doc ID shifts every byte after it">
  <defs><marker id="a1" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0,0 L10,5 L0,10 z" fill="#6b7280"/></marker></defs>
  <text x="20" y="86" font-size="14" font-weight="700" fill="#1f2937">Before</text>
  <text x="212" y="50" font-size="13" text-anchor="middle" fill="#1e40af" font-weight="600">laptop</text>
  <text x="350" y="50" font-size="13" text-anchor="middle" fill="#92400e" font-weight="600">lemon</text>
  <text x="465" y="50" font-size="13" text-anchor="middle" fill="#166534" font-weight="600">light</text>
  <g font-size="15" text-anchor="middle">
    <rect x="120" y="60" width="46" height="40" fill="#dbeafe" stroke="#3b82f6"/><text x="143" y="86">3</text>
    <rect x="166" y="60" width="46" height="40" fill="#dbeafe" stroke="#3b82f6"/><text x="189" y="86">4</text>
    <rect x="212" y="60" width="46" height="40" fill="#dbeafe" stroke="#3b82f6"/><text x="235" y="86">35</text>
    <rect x="258" y="60" width="46" height="40" fill="#dbeafe" stroke="#3b82f6"/><text x="281" y="86">66</text>
    <rect x="304" y="60" width="46" height="40" fill="#fef3c7" stroke="#d97706"/><text x="327" y="86">5</text>
    <rect x="350" y="60" width="46" height="40" fill="#fef3c7" stroke="#d97706"/><text x="373" y="86">9</text>
    <rect x="396" y="60" width="46" height="40" fill="#dcfce7" stroke="#16a34a"/><text x="419" y="86">2</text>
    <rect x="442" y="60" width="46" height="40" fill="#dcfce7" stroke="#16a34a"/><text x="465" y="86">6</text>
    <rect x="488" y="60" width="46" height="40" fill="#dcfce7" stroke="#16a34a"/><text x="511" y="86">11</text>
  </g>
  <path d="M327 102 C 327 135, 281 135, 281 166" fill="none" stroke="#6b7280" stroke-dasharray="4 3" marker-end="url(#a1)"/>
  <path d="M419 102 C 419 135, 373 135, 373 166" fill="none" stroke="#6b7280" stroke-dasharray="4 3" marker-end="url(#a1)"/>
  <text x="20" y="196" font-size="14" font-weight="700" fill="#1f2937">After</text>
  <g font-size="15" text-anchor="middle">
    <rect x="120" y="170" width="46" height="40" fill="#dbeafe" stroke="#3b82f6"/><text x="143" y="196">3</text>
    <rect x="166" y="170" width="46" height="40" fill="#dbeafe" stroke="#3b82f6"/><text x="189" y="196">4</text>
    <rect x="212" y="170" width="46" height="40" fill="#fee2e2" stroke="#dc2626" stroke-width="2"/><text x="235" y="196" fill="#b91c1c" font-weight="700">101</text>
    <rect x="258" y="170" width="46" height="40" fill="#fef3c7" stroke="#d97706"/><text x="281" y="196">5</text>
    <rect x="304" y="170" width="46" height="40" fill="#fef3c7" stroke="#d97706"/><text x="327" y="196">9</text>
    <rect x="350" y="170" width="46" height="40" fill="#dcfce7" stroke="#16a34a"/><text x="373" y="196">2</text>
    <rect x="396" y="170" width="46" height="40" fill="#dcfce7" stroke="#16a34a"/><text x="419" y="196">6</text>
    <rect x="442" y="170" width="46" height="40" fill="#dcfce7" stroke="#16a34a"/><text x="465" y="196">11</text>
    <rect x="488" y="170" width="46" height="40" fill="none" stroke="#9ca3af" stroke-dasharray="4 3"/>
  </g>
  <text x="212" y="234" font-size="13" text-anchor="middle" fill="#1e40af" font-weight="600">laptop</text>
  <text x="304" y="234" font-size="13" text-anchor="middle" fill="#92400e" font-weight="600">lemon</text>
  <text x="419" y="234" font-size="13" text-anchor="middle" fill="#166534" font-weight="600">light</text>
  <g font-size="13" fill="#374151">
    <text x="560" y="72">laptop: docs 3, 7, 42, 108</text>
    <text x="560" y="90">stored as gaps</text>
    <text x="560" y="176" fill="#b91c1c" font-weight="600">Doc 42 removed:</text>
    <text x="560" y="194">gap before 108 = 101</text>
    <text x="560" y="212">list is 1 byte shorter</text>
    <text x="560" y="230">every list after it moves</text>
  </g>
</svg>
<figcaption>One word, one document. "lemon" and "light" were never touched but both of them moved. (Each box here is one byte. Real Lucene packs gaps in blocks of 128, which makes it worse, not better.)</figcaption>
</figure>

One gap changed and every byte after it shifted. A real document has hundreds of words, so one edit would mean rewriting hundreds of these lists.

So Lucene doesn't edit. An update is a **delete + add**:

<figure class="svg-fig">
<svg viewBox="0 0 720 280" role="img" aria-label="Update as delete plus add across segments">
  <defs><marker id="a2" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0,0 L10,5 L0,10 z" fill="#374151"/></marker></defs>
  <g font-size="13">
    <rect x="20" y="30" width="210" height="150" rx="8" fill="#f8fafc" stroke="#94a3b8"/>
    <text x="125" y="54" text-anchor="middle" font-weight="700">Segment 1 · last week</text>
    <rect x="34" y="66" width="182" height="28" rx="4" fill="#fff" stroke="#cbd5e1"/><text x="46" y="85">P17 · usb cable</text>
    <rect x="34" y="102" width="182" height="28" rx="4" fill="#fee2e2" stroke="#dc2626"/><text x="46" y="121" text-decoration="line-through" fill="#991b1b">P42 · gaming laptop</text>
    <rect x="34" y="138" width="182" height="28" rx="4" fill="#fff" stroke="#cbd5e1"/><text x="46" y="157">P88 · laptop bag</text>
    <text x="125" y="200" text-anchor="middle" fill="#6b7280" font-style="italic">file never edited</text>
    <rect x="40" y="214" width="170" height="48" rx="6" fill="#fef3c7" stroke="#d97706"/>
    <text x="125" y="234" text-anchor="middle" font-weight="600">new tiny "deletes" file</text>
    <text x="125" y="252" text-anchor="middle">marks P42 as deleted</text>

    <rect x="270" y="30" width="190" height="80" rx="8" fill="#f0fdf4" stroke="#16a34a"/>
    <text x="365" y="54" text-anchor="middle" font-weight="700">Segment 5 · new</text>
    <rect x="284" y="68" width="162" height="28" rx="4" fill="#dcfce7" stroke="#16a34a"/><text x="296" y="87">P42 · gaming phone</text>
    <text x="365" y="132" text-anchor="middle" font-weight="600">update P42 =</text>
    <text x="365" y="150" text-anchor="middle">flag old copy + write new copy</text>

    <path d="M230 168 L496 168" stroke="#374151" fill="none" marker-end="url(#a2)"/>
    <path d="M460 82 L496 82" stroke="#374151" fill="none" marker-end="url(#a2)"/>
    <text x="365" y="186" text-anchor="middle" fill="#6b7280">later: merge</text>

    <rect x="500" y="30" width="200" height="150" rx="8" fill="#eff6ff" stroke="#3b82f6"/>
    <text x="600" y="54" text-anchor="middle" font-weight="700">Segment 6 · merged</text>
    <rect x="514" y="66" width="172" height="28" rx="4" fill="#fff" stroke="#cbd5e1"/><text x="526" y="85">P17 · usb cable</text>
    <rect x="514" y="102" width="172" height="28" rx="4" fill="#fff" stroke="#cbd5e1"/><text x="526" y="121">P88 · laptop bag</text>
    <rect x="514" y="138" width="172" height="28" rx="4" fill="#fff" stroke="#cbd5e1"/><text x="526" y="157">P42 · gaming phone</text>
    <text x="600" y="200" text-anchor="middle" fill="#6b7280" font-style="italic">deleted copy is gone;</text>
    <text x="600" y="218" text-anchor="middle" fill="#6b7280" font-style="italic">segments 1 and 5 are removed</text>
  </g>
</svg>
<figcaption>Product P42 changes from "gaming laptop" to "gaming phone". Nothing is edited. The old copy is flagged in a small new file, the new copy goes into a new segment, and a merge later writes a clean segment. Even the delete marker is a new file.</figcaption>
</figure>

In other words, every file Lucene has ever written is only read from then on.

## 3. In-place updates under mmap

Reading through mmap is not the issue. The issue is writing. With mmap, the OS decides when a changed page is saved to disk and which pages stay in memory. A database that edits pages in place needs to make those decisions itself. If it hands them to the OS, it runs into two problems. Let's see them one by one.

### 3.1 Half-saved transactions

A transaction usually changes more than one thing, and the database promises **all or nothing**. After a crash you either see every change or none of them. To keep that promise, the database has to decide when changed pages reach the disk. With mmap, the kernel decides that. Below are two examples of the same problem, one across pages and one inside a single page.

#### Example A: across pages

Let's say we transfer ₹500 from account A (on page 1) to account B (on page 7).

Every database keeps a **write-ahead log (WAL)** so it can finish or undo a transaction after a crash. It works only because of one rule: **the log reaches the disk before the changed pages do.** A database with its own buffer pool writes the pages itself, so it simply flushes the log first.

On mmap the database still flushes the log to the disk. But it can't stop the kernel from writing a changed page *earlier*. If the kernel does that and the machine crashes, the log on disk has never heard of the change. So recovery has nothing to undo, and the ₹500 is gone.

<figure class="svg-fig">
<svg viewBox="0 0 720 236" role="img" aria-label="Buffer pool flushes the log before the page; on mmap the kernel writes the page first">
  <defs><marker id="a3" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0,0 L10,5 L0,10 z" fill="#374151"/></marker></defs>
  <g font-size="13" text-anchor="middle">
    <text x="20" y="22" text-anchor="start" font-weight="700" font-size="14">Database with its own buffer pool</text>
    <rect x="20" y="32" width="165" height="72" rx="6" fill="#f8fafc" stroke="#94a3b8"/>
    <text x="102" y="63">1. Edit A and B</text><text x="102" y="81">in its own RAM</text>
    <rect x="205" y="32" width="185" height="72" rx="6" fill="#eff6ff" stroke="#3b82f6"/>
    <text x="297" y="54">2. Database writes page 1:</text><text x="297" y="72" font-weight="600">log to disk first,</text><text x="297" y="90" font-weight="600">then the page</text>
    <rect x="410" y="32" width="90" height="72" rx="6" fill="#fee2e2" stroke="#dc2626"/><text x="455" y="73" font-weight="700" fill="#b91c1c">CRASH</text>
    <rect x="520" y="32" width="180" height="72" rx="6" fill="#dcfce7" stroke="#16a34a"/>
    <text x="610" y="54">Log: transfer started,</text><text x="610" y="72">never committed</text><text x="610" y="90" font-weight="700">undo A → ₹500 safe ✓</text>

    <text x="20" y="140" text-anchor="start" font-weight="700" font-size="14">Same database on mmap</text>
    <rect x="20" y="150" width="165" height="72" rx="6" fill="#f8fafc" stroke="#94a3b8"/>
    <text x="102" y="181">1. Edit A and B</text><text x="102" y="199">in mapped memory</text>
    <rect x="205" y="150" width="185" height="72" rx="6" fill="#fff7ed" stroke="#ea580c"/>
    <text x="297" y="172">2. Kernel writes page 1</text><text x="297" y="190">(A = 500) on its own,</text><text x="297" y="208" font-weight="600">log not flushed yet</text>
    <rect x="410" y="150" width="90" height="72" rx="6" fill="#fee2e2" stroke="#dc2626"/><text x="455" y="191" font-weight="700" fill="#b91c1c">CRASH</text>
    <rect x="520" y="150" width="180" height="72" rx="6" fill="#fee2e2" stroke="#dc2626"/>
    <text x="610" y="172">Disk: A debited, B not</text><text x="610" y="190">Log on disk: nothing</text><text x="610" y="208" font-weight="700" fill="#b91c1c">₹500 gone ✗</text>
  </g>
  <g stroke="#374151" fill="none">
    <path d="M185 68 L201 68" marker-end="url(#a3)"/><path d="M390 68 L406 68" marker-end="url(#a3)"/><path d="M500 68 L516 68" marker-end="url(#a3)"/>
    <path d="M185 186 L201 186" marker-end="url(#a3)"/><path d="M390 186 L406 186" marker-end="url(#a3)"/><path d="M500 186 L516 186" marker-end="url(#a3)"/>
  </g>
</svg>
<figcaption>Both databases have a log. The difference is who writes page 1 and whether the log gets there first.</figcaption>
</figure>

The database can *force* a write to the disk any time it wants. What it can't do is *stop* the kernel from writing a page early. The only way around it is to edit a private copy and move it into the mapped file once the log is safe. But that is the same as building a buffer pool again.

#### Example B: inside one page

Now zoom in. Even a single edit takes more than one step. Let's say B's row on page 7 holds a balance and a version number. The transaction has updated the balance but not yet the version.

In a database with its own buffer pool, the transaction holds a lock on the page while editing it. The background process that writes pages to disk has to take the same lock first, so it waits until the edit is done. The kernel takes no lock. It can copy the page in the middle of the edit, and if the machine crashes before the next write, that half-edited page is what stays on disk.

<figure class="svg-fig">
<svg viewBox="0 0 720 300" role="img" aria-label="Background writer waits for the page lock; kernel writeback copies mid-edit">
  <defs><marker id="a4" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0,0 L10,5 L0,10 z" fill="#374151"/></marker></defs>
  <g font-size="13" text-anchor="middle">
    <text x="185" y="26" font-weight="700" font-size="15">Database with its own buffer pool</text>
    <text x="535" y="26" font-weight="700" font-size="15">Database on mmap</text>

    <rect x="20" y="40" width="330" height="80" rx="8" fill="#f8fafc" stroke="#94a3b8"/>
    <text x="185" y="62" font-weight="700">Page 7 in memory · mid-edit · locked</text>
    <text x="185" y="84">balance: 1000 → 1500 <tspan fill="#16a34a">done</tspan></text>
    <text x="185" y="104">version: v1 → v2 <tspan fill="#d97706">not yet</tspan></text>

    <rect x="370" y="40" width="330" height="80" rx="8" fill="#f8fafc" stroke="#94a3b8"/>
    <text x="535" y="62" font-weight="700">Page 7 in memory · mid-edit</text>
    <text x="535" y="84">balance: 1000 → 1500 <tspan fill="#16a34a">done</tspan></text>
    <text x="535" y="104">version: v1 → v2 <tspan fill="#d97706">not yet</tspan></text>

    <rect x="60" y="146" width="250" height="52" rx="6" fill="#eff6ff" stroke="#3b82f6"/>
    <text x="185" y="168" font-weight="600">Background writer</text>
    <text x="185" y="187">needs the page lock → waits</text>

    <rect x="410" y="146" width="250" height="52" rx="6" fill="#fff7ed" stroke="#ea580c"/>
    <text x="535" y="168" font-weight="600">Kernel writeback</text>
    <text x="535" y="187">no lock to ask for → copies now</text>

    <rect x="40" y="224" width="290" height="58" rx="6" fill="#dcfce7" stroke="#16a34a"/>
    <text x="185" y="247">Crash at any point: disk only</text>
    <text x="185" y="267" font-weight="700">ever holds a finished edit ✓</text>

    <rect x="390" y="224" width="290" height="58" rx="6" fill="#fee2e2" stroke="#dc2626"/>
    <text x="535" y="247">Crash now: disk keeps 1500 + v1</text>
    <text x="535" y="267" font-weight="700" fill="#b91c1c">a state no transaction created ✗</text>
  </g>
  <g stroke="#374151" fill="none">
    <path d="M185 120 L185 142" marker-end="url(#a4)"/><path d="M185 198 L185 220" marker-end="url(#a4)"/>
    <path d="M535 120 L535 142" marker-end="url(#a4)"/><path d="M535 198 L535 220" marker-end="url(#a4)"/>
  </g>
</svg>
<figcaption>Same edit, same moment. The buffer pool gives the database a "not now" button. mmap doesn't.</figcaption>
</figure>

Lucene has neither of these problems. It never edits a page, so there is never a half-finished change for the OS to save. A new segment that's only partly on disk is not referenced by any commit yet, so a crash just leaves an unused file behind.

### 3.2 Cache eviction without query context

This one is not about correctness and it doesn't come from editing in place. It's about how well the cache gets used.

At the application level we have control over what is stored in application buffers. We can customize what goes into those buffers based on query patterns. But at the OS cache level we lose that control. Let's see one example.

The OS evicts pages based only on how recently they were touched. Assume that the cache holds the hot index pages `I1–I6` and someone runs a full scan of a 100 GB table. The database knows those scan pages will never be read again, so it pushes them through a small ring of reused slots. The OS has no way of knowing that. To the OS the scan pages are just the most recently used ones, so the index pages go out. Classic cache pollution problem.

<figure class="svg-fig">
<svg viewBox="0 0 720 290" role="img" aria-label="Scan evicting hot pages versus ring buffer">
  <defs><marker id="a5" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0,0 L10,5 L0,10 z" fill="#374151"/></marker></defs>
  <g font-size="13" text-anchor="middle">
    <text x="20" y="28" text-anchor="start" font-weight="700" font-size="14">OS page cache after a full table scan</text>
    <rect x="20" y="44" width="110" height="44" rx="6" fill="#f3f4f6" stroke="#9ca3af"/><text x="75" y="64">scan:</text><text x="75" y="80">T1 … T100</text>
    <rect x="150" y="44" width="60" height="44" fill="#fee2e2" stroke="#dc2626"/><text x="180" y="71">T93</text>
    <rect x="216" y="44" width="60" height="44" fill="#fee2e2" stroke="#dc2626"/><text x="246" y="71">T94</text>
    <rect x="282" y="44" width="60" height="44" fill="#fee2e2" stroke="#dc2626"/><text x="312" y="71">T95</text>
    <rect x="348" y="44" width="60" height="44" fill="#fee2e2" stroke="#dc2626"/><text x="378" y="71">T96</text>
    <rect x="414" y="44" width="60" height="44" fill="#fee2e2" stroke="#dc2626"/><text x="444" y="71">T97</text>
    <rect x="480" y="44" width="60" height="44" fill="#fee2e2" stroke="#dc2626"/><text x="510" y="71">T98</text>
    <rect x="546" y="44" width="60" height="44" fill="#fee2e2" stroke="#dc2626"/><text x="576" y="71">T99</text>
    <rect x="612" y="44" width="60" height="44" fill="#fee2e2" stroke="#dc2626"/><text x="642" y="71">T100</text>
    <text x="411" y="114" fill="#b91c1c">Hot index pages I1–I6 pushed out → next lookup reads from disk ✗</text>

    <text x="20" y="160" text-anchor="start" font-weight="700" font-size="14">Database cache: scan goes through a small ring</text>
    <rect x="20" y="176" width="110" height="44" rx="6" fill="#f3f4f6" stroke="#9ca3af"/><text x="75" y="196">scan:</text><text x="75" y="212">T1 … T100</text>
    <rect x="150" y="176" width="60" height="44" fill="#dbeafe" stroke="#3b82f6"/><text x="180" y="203">I1</text>
    <rect x="216" y="176" width="60" height="44" fill="#dbeafe" stroke="#3b82f6"/><text x="246" y="203">I2</text>
    <rect x="282" y="176" width="60" height="44" fill="#dbeafe" stroke="#3b82f6"/><text x="312" y="203">I3</text>
    <rect x="348" y="176" width="60" height="44" fill="#dbeafe" stroke="#3b82f6"/><text x="378" y="203">I4</text>
    <rect x="414" y="176" width="60" height="44" fill="#dbeafe" stroke="#3b82f6"/><text x="444" y="203">I5</text>
    <rect x="480" y="176" width="60" height="44" fill="#dbeafe" stroke="#3b82f6"/><text x="510" y="203">I6</text>
    <rect x="546" y="176" width="60" height="44" fill="#fef3c7" stroke="#d97706"/><text x="576" y="203">T99</text>
    <rect x="612" y="176" width="60" height="44" fill="#fef3c7" stroke="#d97706"/><text x="642" y="203">T100</text>
    <text x="609" y="152" fill="#92400e">ring: reused</text>
    <text x="411" y="276" fill="#166534">Scan pages only cycle through the ring; I1–I6 stay cached ✓</text>
  </g>
  <g stroke="#374151" fill="none">
    <path d="M130 66 L146 66" marker-end="url(#a5)"/>
    <path d="M672 176 C 672 160, 546 160, 546 172" marker-end="url(#a5)"/>
    <path d="M75 220 L75 248 L609 248 L609 224" marker-end="url(#a5)"/>
  </g>
</svg>
<figcaption>The database knows the scan is a one-time read. The OS can only guess from recency.</figcaption>
</figure>

Lucene has the same limitation. It just accepts it because its workload is mostly index lookups. Databases serve different use-cases with mixed workloads, so they want the control.

## 4. Write-once implications for Lucene

Write-once is not free either. Until a merge runs, deleted documents still sit inside the segment files. Let's say a 10 GB segment has 4 GB of deleted docs. Please find the breakdown below.

| Cost | What happens |
|---|---|
| Wasted page cache | The OS caches pages, not documents. 40% of the cached bytes for that segment hold dead docs, so less live data fits in RAM. |
| Wasted CPU | Lucene still decodes the deleted doc IDs from the posting lists and then skips them. |
| Skewed scoring | BM25's IDF counts how many docs contain a word. If 1,000 docs have "laptop" and 300 of them are deleted, IDF still uses 1,000 until the merge. So scores can shift slightly after it. |

That's why merge policies prioritize segments with a lot of deletes, and why an index that stops receiving writes is often force-merged.

## 5. Summary

mmap is safe only when the data never changes after it's written. Lucene designed its file format around that: tightly packed, write-once segments where an update is a delete plus an add. Databases that edit pages in place can't hand that control to the OS without giving up the guarantees their applications depend on. So they keep their own cache.

- **Lucene** is a library of printed books. The librarian (the OS) can shelve, fetch and remove copies however she likes. The books never change, so nothing can go wrong.
- **A database** is a whiteboard people are actively writing on. You can't let the janitor photograph and archive it whenever he walks by. He might capture half a sentence.

So before reaching for mmap in your own storage layer, knowing your application is important.

> Ask one question: does any byte change after it's written? If no, let the OS do the caching. If yes, keep your own buffer pool.

## 6. References

- **MongoDB MMAPv1.** MongoDB's original storage engine was built on mmap. It was later replaced by WiredTiger, which manages its own cache, and removed in version 4.2.
- **Crotty, Leis, Pavlo.** *"Are You Sure You Want to Use MMAP in Your DBMS?"* CIDR 2022. Covers these problems in much more detail.
- **LMDB.** A database that does use mmap successfully. It never edits a page in place. A changed page is written as a new copy and one pointer flips at commit. By default it maps the file read-only and writes with normal file writes.
