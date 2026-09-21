---
title: "Why Facet Counts Break When You Add Vector Search"
date: 2026-09-21 10:00:00 +0530
categories: [Search, Faceting]
tags: [vector-search, hybrid-search, facets, ecommerce]
description: "Hybrid search quietly turns your facet counts into a function of topK. Here's why it happens, and the three ways out."
---

You add a vector leg to your keyword search. Relevance goes up, the demo looks great, it ships. A few weeks later someone files a bug: the facet says **Adidas (22)**, they click it, and the page shows around 105 results.

No exception, no failed test. The code is doing exactly what you asked. The problem is that facet counts and top-K vector retrieval make incompatible assumptions, and hybrid search puts them in the same request.

## The short version

A facet count is a headcount. "Adidas (22)" means there are 22 Adidas products in your results. Counting only works if you know the **full list** of matching documents.

Lexical search gives you a full list. Vector search doesn't. It gives you a top-N ranking, cut off wherever you set `topK`. When you count facets over a mix of the two, the number you show depends more on your `topK` setting than on what's actually in the catalogue.

## The problem, with numbers

The numbers below are illustrative, but the mechanics are exactly what you'll see in production.

Setup: a shoe catalogue with 3,000 Adidas products. The search is hybrid:

- **Lexical leg:** normal keyword matching (BM25)
- **Vector leg:** the 100 documents whose embeddings are closest to the query embedding (`topK=100`)

Query: *"cushioned trail runners for flat feet"*

- Keyword matching finds 30 documents
- Vector search returns its 100 closest documents
- Combined (assume no overlap for simplicity): 130 documents
- Of those 130, 22 are Adidas
- The page displays: **Adidas (22)**

Now the user clicks the Adidas facet.

The search runs again with a brand filter. The vector search is now restricted to Adidas only, so instead of 100 documents spread across all brands, it returns 100 documents that are *all* Adidas. Add the handful of keyword matches that are Adidas, and the page shows roughly 105 products.

The facet said 22. The user got 105.

## Why this happens: a complete list vs a shortlist

The 30 keyword matches are complete. If a document contains those terms, it's in the list.

The 100 vector matches are a shortlist. They're the closest 100 *because you asked for 100*. There could easily be another 800 documents almost exactly as close, sitting just outside the cutoff. Counting facets over a complete list plus a shortlist gives a number that isn't a true count of either.

Clicking the facet makes it worse, because the shortlist gets rebuilt. The vector search doesn't narrow its previous 100 results down to the Adidas ones. Most engines **pre-filter** vector search by default, so it searches again, looking only at Adidas documents, and finds a fresh 100. Most of what the user lands on was never in the list that was counted.

Post-filtering (keeping only the Adidas ones from the original 100) would make the count match, but then a user who explicitly asked for Adidas sees 22 results out of 3,000 Adidas products. You trade a lying count for a starved results page.

With keyword-only search none of this happens. Filtering a complete list always gives a subset of that list, so the count always holds.

How bad it gets depends on how big the keyword list is compared to `topK`. If keyword finds 40,000 and vector adds 100, nobody notices. If keyword finds 30 and vector adds 100, the counts are almost entirely a product of `topK`, and that happens on rare terms, misspellings and long conversational queries, which are exactly the queries vector search was added to fix.

## The three ways out

**A. Use vectors to reorder, never to find.** The matching list comes only from keyword matching plus filters, and vector similarity just re-sorts it. Counts stay accurate, but a document worded differently from the query never shows up.

**B. Ask for "everything similar enough," not "the closest 100."** A similarity threshold makes the vector leg a complete list defined by the data, not by `topK`. The catch: similarity scores aren't on the same scale across queries, so one threshold can return 4 documents for one query and 9,000 for another. It needs calibration against real query logs.

**C. Run two queries and accept the mismatch.** Counts come from a keyword-only query, results from the hybrid query. The counts won't match the page, so only do this if you drop the numbers and show facets as plain filters.

## Which businesses actually need facet counts?

Not every business needs them, and the ones that do are usually the ones that need vector search the least.

Facet counts matter when users come with precise, structured queries and use facets to narrow down a large set of near-identical products. Think industrial and parametric catalogues like Digi-Key, Grainger and Mouser, where a query like "0.1% 10kΩ 0805" is matched on exact terms and specs, and narrowing 40,000 resistors down to 12 is the whole point of the interface. Here lexical precision is what matters and semantic recall adds little, because the vocabulary is closed. Counts tell the user how much each filter will narrow things down, so they have to be exact.

Large consumer marketplaces are the opposite. Queries are messy natural language, so semantic recall is valuable, and counts are not. At the time of writing, both Amazon and Walmart show facets as filters with no counts at all, and they dropped them long before vector search, for reasons like personalized ranking and the cost of exact counts at scale.

| Business type | Query style | Needs exact counts? | Value of semantic recall |
|---|---|---|---|
| Parametric / industrial catalogue | Structured ("0.1% 10kΩ 0805") | Yes, it's the core workflow | Low, the vocabulary is closed |
| Large consumer marketplace | Messy natural language | No, already dropped | High |
| Mid-size consumer retail | Mixed | Often yes | Moderate to high |

**The middle is where it bites:** a mid-size retailer that shows counts and also wants semantic recall. That's where you should pick Option A deliberately, as a product decision, instead of discovering the inconsistency in production.

## The rule to remember

**Using vectors to *find* documents breaks faceted navigation. Using vectors to *rank* documents doesn't.**

And whichever option you pick, apply your filters to **both** the keyword leg and the vector leg. Filter only one, and the two legs disagree about which documents are even eligible. That's a second, subtler version of the same bug.
