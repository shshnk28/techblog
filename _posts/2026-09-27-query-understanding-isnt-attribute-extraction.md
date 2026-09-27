  ---
title: "Query Understanding Isn't Attribute Extraction: The 6 Jobs It Does in E-commerce"
date: 2026-09-27 10:00:00 +0530
categories: [Search, Query Understanding]
tags: [query-understanding, ecommerce, search-relevance, intent, entity-extraction]
description: "Query understanding is more than pulling out brand, color and price. Here are the six jobs it actually does in e-commerce search, and why the hard part is deciding when to filter."
---

Consider a shopper on an e-commerce site who needs a new phone cover. She types **"iphone 15 cover"** into the search box and hits enter. The first page is full of iPhones. The phones themselves, not covers.

Now consider a second shopper doing her weekly grocery order. She types **"apple"** and gets zero results. The system decided "apple" meant the brand, filtered down to Apple products, and found none in the grocery section.

In both cases the right products were sitting in the catalog. Ranking wasn't the problem. Search simply misunderstood what the user wanted, and preventing that is the job of **Query Understanding (QU)**.

A lot of people think of QU as attribute extraction: pull out the brand, the color, the price, and you're done. QU is not just attribute extraction. It does a lot more than that, so let's break it down, with e-commerce examples for each part.

## The core idea

QU converts a raw user query into structured search intent. Understanding the words is only the starting point. What retrieval and ranking actually need from QU is the answer to one question:

**What is the user trying to buy, and how should search behave for this query?**

It helps to picture QU as a series of steps, each one adding a bit more structure to the raw query:

> Raw query → intent → entities / attributes → rewrites → filters / boosts → use-case signals → confidence

Let's go through them one by one.

## 1. Intent understanding

Intent understanding figures out what *kind* of search the user is doing.

| Query | Intent |
|---|---|
| iphone 15 | Specific product / model search |
| running shoes | Category browse |
| best phone under 30000 | Recommendation / comparison |
| iphone 15 cover | Accessory search |

Different intents need different search behavior. The user who types "iphone 15" knows exactly what she wants, so search should behave like a model search and an exact-match path makes sense. "running shoes" is a browse, so show the range and let her narrow it down with facets. "best phone under 30000" is closer to a recommendation. She wants help choosing.

So intent ends up driving a lot of decisions downstream:

- the retrieval strategy (exact matching vs broad recall)
- which index, vertical, or category to search
- whether to return products, accessories, brands, or buying guides
- which ranking model or features to use
- which UI modules and facets to show

## 2. Entity and attribute extraction

This is the part most people already know. You pull structured meaning out of the query.

Let's say the query is **black nike running shoes for men under 5000**. QU should extract:

- brand = Nike
- category = running shoes
- color = black
- audience = men
- price ≤ 5000

The usual suspects are brand, category, product/model, color, size, material, price, gender, occasion, and compatibility (for example, which phone a cover fits).

| Query | Extracted meaning |
|---|---|
| red cotton saree under 2000 | category = saree, color = red, material = cotton, price < 2000 |
| iphone 15 cover | product = iPhone 15, category = cover |
| 32 inch smart tv | size = 32 inch, category = smart TV |
| puma shoes for men | brand = Puma, category = shoes, audience = men |

Extraction is one of the most important parts of e-commerce QU. But pulling an attribute out of the query is only half the job. What you do with it is the other half, and that comes in section 4.

## 3. Query rewriting

Query rewriting turns what the user typed into a better query for retrieval.

| User query | Rewritten query |
|---|---|
| fridge double door | double door refrigerator |
| mobile cover iphone 15 | iphone 15 case cover |
| office chair back support | ergonomic office chair |
| shoes for marathon | running shoes |

The reason is simple. Users and catalogs speak different languages. The user says "fridge", the catalog says "refrigerator", and a plain keyword match misses the connection.

Take "mobile cover iphone 15". Searched as-is, the word "iphone" can pull actual iPhones into the results. Rewritten as "iphone 15 case cover", it's clear the thing being searched for is a cover.

Rewrites are usually learned from search logs, click logs, add-to-cart and purchase logs, and catalog terms, plus some hand-curated rules on top.

## 4. Filter or boost?

Once an attribute is extracted, QU has to decide what to do with it. There are three options: a **hard filter** (only show matching products), a **soft boost** (prefer matching products but keep the others), or a **ranking hint**.

- **"iphone under 50000"**: price can safely be a hard filter. The user gave a clear limit.
- **"cheap iphone"**: "cheap" should not become a hard filter. There's no number to filter on, so use it as a ranking signal instead.
- **"red dress"**: "red" could be a filter or a boost. It depends on how confident QU is and how well the catalog is tagged.

That last point matters more than it looks. For our "black nike running shoes for men under 5000" query, a sensible output is:

```json
{
  "filters": [
    { "field": "category", "op": "=",  "value": "running shoes" },
    { "field": "brand",    "op": "=",  "value": "Nike" },
    { "field": "price",    "op": "<=", "value": 5000 }
  ],
  "boosts": [
    { "field": "color",    "value": "black", "weight": 0.4 },
    { "field": "audience", "value": "men",   "weight": 0.3 }
  ]
}
```

Why is brand a filter but color only a boost? Because of **catalog data quality**. Brand data is usually clean. A Nike shoe is tagged "Nike". Color data is messy. The same shoe might be tagged "black", "charcoal", "jet black", or nothing at all, and a hard filter on color would quietly drop good products. Audience has the same problem, since a lot of running shoes are tagged "unisex" and a hard filter on "men" would hide all of them.

So not every extracted attribute should become a hard filter. A wrong boost pushes a good product down a few places. A wrong filter removes it completely.

## 5. Use-case understanding

Intent decides *how search behaves*. Use-case understanding decides *which attributes matter when the user didn't name any*.

Many e-commerce queries aren't literal product searches. Users often don't know the exact specs they need, and they depend on the store to surface the right product.

| Query | System should infer |
|---|---|
| laptop for coding | RAM, CPU, SSD, keyboard, display matter |
| phone for gaming | processor, battery, refresh rate, cooling matter |
| shoes for marathon | running shoes; cushioning and durability matter |
| gift for 2 year old | age-appropriate toys and gifts |
| dress for wedding guest | occasion- and style-based products |

"laptop for coding" has no RAM, CPU, or SSD info in it. With "phone for gaming", the user hasn't said which brand or model she wants either. But we know that for gaming, a high refresh rate and a powerful processor matter, and a great camera is not the priority. Basically, with a spec-less query like this, it becomes QU's job to work out the specs and pass them to retrieval and ranking as boosts.

## 6. Handling ambiguous queries

Some queries can mean very different things, and QU needs to know when it's guessing.

"iphone 15 cover" has only one reasonable meaning: covers that fit an iPhone 15. QU can guide retrieval strongly here.

Ambiguous queries are more common than you'd expect, especially on stores that sell across many categories:

| Query | Possible meanings |
|---|---|
| drumsticks | chicken drumsticks (grocery), drum sticks (musical instruments), drumstick vegetable / moringa (grocery) |
| apple | Apple phones and laptops, apple fruit, apple juice or apple cider vinegar |
| glasses | spectacles and sunglasses, drinking glasses (kitchen) |
| keyboard | computer keyboard, musical keyboard |
| mouse | computer mouse, mouse trap (home and pest control) |
| tablet | Android tablet or iPad, medicine tablets (pharmacy) |

In every one of these, the meanings live in completely different parts of the catalog. If QU commits to one meaning with a hard category filter, everyone who meant something else gets bad results or no results at all. That's exactly what happened to our grocery shopper at the start.

How does QU know a query is ambiguous? For queries it has seen before, past behavior tells it. If clicks and purchases for "drumsticks" are split between grocery and musical instruments, the query is ambiguous. For new queries, it goes by how confident its models are about the category.

QU attaches this as a **confidence** score to its interpretation. When confidence is low, the safe strategy is to keep retrieval broad, use soft boosts, and let ranking and facets sort out what the user meant. The shopper who wanted chicken drumsticks can tap "Grocery" in the facets and she's there. The shopper who got zero results usually just leaves.

## The segment signal: head, torso, and tail

The six jobs above all *interpret* the query. QU also produces one more signal that works differently. It tells you how familiar the system is with the query, based on how often it's been searched before.

- **Head**: searched very often, with strong behavioral data. Example: "iphone".
- **Torso**: searched often enough to have decent signals. Example: "nike running shoes".
- **Tail**: rarely or never seen before. Example: "laptop for java spring boot development under 70000".

Please note that this signal says nothing about what the query *means*. In other words, it measures how much history you have with the query, and that decides **how much to trust the other six outputs**.

For head and torso queries, it's worth precomputing and curating things: query-specific rewrites, learned rankings, even hand-tuned merchandising. You have the data to build them and the traffic to justify the effort. Tail queries have no history to learn from, so the system has to generalize from entity extraction, taxonomy, use-case understanding, and semantic matching.

## Putting it together

For **black nike running shoes for men under 5000**, the complete QU output looks like this:

```json
{
  "intent": "category_search_with_constraints",
  "entities": [
    { "type": "brand",    "value": "Nike" },
    { "type": "category", "value": "running shoes" },
    { "type": "color",    "value": "black" },
    { "type": "audience", "value": "men" },
    { "type": "price",    "operator": "<=", "value": 5000 }
  ],
  "rewrites": [],
  "filters": [
    "category = running shoes",
    "brand = Nike",
    "price <= 5000"
  ],
  "boosts": [
    "color = black",
    "audience = men"
  ],
  "use_case_signals": [],
  "confidence": "high",
  "segment": "torso"
}
```

Two fields are empty, and that's correct. There's nothing to rewrite because every word was already understood. There are no use-case signals because the user spelled out exactly what she wants. With "laptop for coding" it would be the other way round: very few entities, and most of the useful information sitting in `use_case_signals`.

## How QU is built

QU is not one model. Each job above is usually handled by its own component, and sometimes by several:

- **Intent**: a text classifier that sorts queries into search types.
- **Entities and attributes**: dictionaries of known brands and categories, plus a tagging model that labels each word in the query ("nike" → brand, "black" → color).
- **Rewrites**: mappings learned from logs. For example, users who searched "fridge" bought products titled "refrigerator". These usually sit alongside hand-written rules.
- **Use-cases**: mappings from phrases like "for gaming" to the attributes that matter, either curated by hand or learned from what users bought.
- **Segment**: a lookup table built offline from query frequency.

The QU service itself is a **wrapper over these components**. It calls each one, merges the results into the single structured output you saw above, and applies the filter-vs-boost rules based on how confident each component is.

This split pays off in practice. Each model can be retrained or replaced on its own schedule, while retrieval and ranking only ever see one stable output format. Most teams start with dictionaries and rules, move to trained models once they've collected enough data, and are now using LLMs more and more for long, unusual queries where the other components don't have much to work with.

## Key takeaway

QU in e-commerce does six jobs:

1. Understand intent
2. Extract entities and attributes
3. Rewrite the query
4. Decide filter vs boost
5. Understand use-cases
6. Handle ambiguous queries safely

On top of that, the segment signal (head, torso, or tail) decides how much to trust the other six.

Extracting an attribute is the easy part. Deciding whether you're sure enough to filter on it is the hard part. So when in doubt, boost, don't filter.
