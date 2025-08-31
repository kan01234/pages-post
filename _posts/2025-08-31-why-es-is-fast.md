---
layout: post
title: "Why Elasticsearch Is Fast and How It Stores Data"
date: 2025-08-31
tags: es, performance, data-structures
categories: datastore
---

Elasticsearch (ES) has become the go-to engine for powering search, analytics, and log processing. From e-commerce platforms handling millions of product searches per day to observability pipelines crunching terabytes of logs, ES consistently delivers sub-second responses. But what makes it so fast—and how exactly are documents stored inside it?  

This post explores the internals of Elasticsearch: the inverted index, document storage, segments, and retrieval paths.

---

## Inverted Index: Foundation of Speed

Traditional databases rely on row-based indexes such as B-Trees. Elasticsearch, built on Apache Lucene, uses a different model: the **inverted index**. Instead of mapping primary keys to rows, it maps **terms to the documents that contain them**.

For example:

``
Document 1: "Elasticsearch is fast"
Document 2: "Fast search engines use inverted indexes"
``

The inverted index looks like:

``
elasticsearch → [1]
fast          → [1, 2]
search        → [2]
engines       → [2]
use           → [2]
inverted      → [2]
indexes       → [2]
``

With this structure, finding all documents that contain "fast" is just a matter of looking up the term in the dictionary—no need to scan rows.

✅ **Why it’s fast**: ES avoids full scans by jumping directly to the matching document IDs.

---

## How a Document Is Stored

When you index a JSON document into ES:

``
{
  "user": "kan",
  "message": "Elasticsearch is fast",
  "likes": 3
}
``

Elasticsearch transforms it into several storage layers:

1. **_source (optional)**  
   - The original JSON is stored as a compressed binary blob.  
   - This allows you to fetch the full document later via `GET`.  
   - You can disable `_source` to save space, but you lose the ability to retrieve the raw document.

2. **Inverted Index**  
   - Text fields go through an **analyzer** (tokenizer + filters).  
   - Example: "Elasticsearch is fast" → `elasticsearch → [doc1]`, `fast → [doc1]`.  
   - Stored in postings lists for efficient term-to-document lookups.

3. **Doc Values (Column Store)**  
   - For fields that need **sorting, aggregations, or grouping** (e.g., `likes`, `user`).  
   - Stored in a column-oriented format on disk.  
   - Optimized for scanning and aggregations.

4. **Stored Fields**  
   - Metadata like `_id`, routing information, and field-level storage.

So after indexing, ES effectively has:

``
DocID: 1
--------------------------
_source: {user:"kan", message:"Elasticsearch is fast", likes:3}

Inverted Index:
  "elasticsearch" → [1]
  "fast"          → [1]

Doc Values (columnar):
  user: [ "kan", ... ]
  likes: [ 3, ... ]
``

---

## Segments and Immutability

Internally, Lucene (which ES is built on) writes documents into **segments**. Each segment is like a mini-index containing:  
- A postings list (inverted index)  
- Stored fields  
- Doc values  
- A term dictionary  

Segments are immutable. New documents create new segments. Deletes are handled by a delete marker bitmap until a background merge compacts segments.  

✅ **Why it’s fast**: Immutable segments allow lock-free, concurrent reads even while indexing is happening.

---

## Query and Retrieval

When you do:

``
GET my_index/_doc/1
``

Elasticsearch looks up the doc ID in the stored fields, retrieves the `_source` JSON (if enabled), and returns it.  

When you run a search query:  
- ES scans the inverted index for matching terms.  
- It fetches candidate doc IDs.  
- It uses doc values if sorting/aggregating is required.  
- Finally, it fetches the `_source` only for the top hits (not for every candidate doc).

---

## Why It’s Fast Altogether

Elasticsearch is fast not because of a single trick, but because of a carefully layered design:  

- Inverted index for blazing-fast term lookups  
- Immutable segments for lock-free reads  
- Columnar storage for analytics  
- Distributed sharding and replication  
- Smart caching and memory mapping  

The result: a system that can handle both search and analytics at scale with sub-second performance.
