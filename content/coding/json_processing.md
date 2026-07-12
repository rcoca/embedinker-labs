---
title: "Processing Gigabytes of JSON: Why Streaming is the Only Way to Scale"
date: 2026-07-14T15:00:00+00:00
draft: false
categories:
  - Engineering
  - Coding
tags:
  - C++
  - JSON
  - Performance Optimization
  - Systems Programming
  - Streaming I/O
description: Avoid the 'DOM Trap' when parsing massive JSON files. Learn how to use streaming pipelines and on-the-fly decompression to process 10GB+ files with minimal RAM.
series: High Performance Systems
slug: processing-massive-json-streaming-vs-dom
---


# Processing Gigabytes of JSON: Why Streaming is the Only Way to Scale
### The Massive Config Audit

In large-scale cloud or storage environments, the "state of the world" is often exported as a giant JSON dump (e.g., an inventory of every virtual disk, volume, and node in a cluster). These files can be tens of gigabytes when uncompressed. If we try to open these with a standard JSON library, our application will crash because those libraries typically build a **DOM (Document Object Model)**—a tree structure in RAM that is often 5-10x larger than the file itself.

**The Production Goal:** Build a tool that can calculate statistics (like model distribution) from a compressed inventory dump without ever loading more than a few kilobytes into memory at once.
### Streaming vs. DOM Parsing
#### The Challenge

Sometimes, developers use `json::parse(file)`, which reads the entire file into a tree structure. If the file is 10GB, we might need 50GB of RAM. In production, this is an immediate "Out of Memory" (OOM) failure.

#### Strategy 1: The Decompression Pipeline

We don't want to decompress the `.bz2` file to disk first—that wastes I/O and disk space. Instead, we create a **streaming pipeline**.

- **The Tool:** `libarchive` or `bzlib`.
- **The Logic:** Feed the compressed bytes into a buffer, and pass only the decompressed chunks to the parser.
```c
/* 
 * The Pipeline: Compressed File -> Decompression Buffer -> Object Accumulator -> Parser
 */
while (archive_read_next_header(a, &entry) == ARCHIVE_OK) {
    char chunk[65536]; // Read in 64KB blocks for high throughput
    la_ssize_t bytes_read;

    while ((bytes_read = archive_read_data(a, chunk, sizeof(chunk))) > 0) {
        for (la_ssize_t i = 0; i < bytes_read; i++) {
            // 1. Dynamic Buffer Expansion
            // Ensure the accumulator can hold the current object
            if (work_idx + 1 >= capacity) {
                capacity *= 2;
                work_buf = realloc(work_buf, capacity);
            }
            work_buf[work_idx++] = chunk[i];

            // 2. Object Boundary Detection
            // When we hit '}', we have a potentially complete JSON object
            if (chunk[i] == '}') {
                work_buf[work_idx] = '\0'; // Null-terminate for the parser
                
                char *obj_start = strchr(work_buf, '{');
                if (obj_start) {
                    // Trigger the stream parser on the isolated object
                    nanojsonc_parse_object(obj_start, NULL, &total_tags, on_json_pair);
                }
                
                // Reset accumulator for the next object
                work_idx = 0;

                // Progress reporting to avoid "silent" processing of GBs of data
                if (total_tags % 50000 == 0) {
                    fprintf(stderr, "\rUnique models: %d | Total tags: %d", 
                            unique_models_count, total_tags);
                    fflush(stderr);
                }
            }
        }
    }
    break; // Process the first file in the archive and exit
}
```
#### Strategy 2: Stream Parsing (SAX style)

Instead of building a tree, we use a **Stream Parser** (like `nanojsonc` or a custom state machine).

- **The Logic:** The parser emits "events" (e.g., `StartObject`, `Key("model")`, `StringValue("SSD_X100")`).
- **The Benefit:** We only care about the current token. Once we've counted the model, we throw that token away and move to the next one.

**The "Interesting Part": The Token Loop (Conceptual Snippet)**

```cpp
// Instead of: auto data = json::parse(file); 
// We do this (or equivalent, in C):
while (stream_parser.next_token(token)) {
    if (token.type == TOKEN_KEY && token.value == "model") {
        std::string model_name = stream_parser.get_next_value();
        inventory_counts[model_name]++; // Only store the counts, not the whole file
    }
}
```
While the logic is conceptually simple (as seen in this pseudo-code), implementing it in C requires a custom hash table to avoid the overhead of high-level containers.
#### Strategy 3: The Hash Table (C Implementation)
In C, we don't have the luxury of built-in maps. To keep the tool fast and memory-stable, we implemented a **Fixed-Size Hash Table with Linear Probing**.
```C
// Using a prime number for TABLE_SIZE reduces collisions
#define TABLE_SIZE 10007 

unsigned int hash(const char *str) {
    unsigned int h = 5381; // DJB2 starting constant
    int c;
    while ((c = *str++))
        h = ((h << 5) + h) + c; // h * 33 + c
    return h % TABLE_SIZE;
}

void update_count(const char *model_name) {
    unsigned int h = hash(model_name);
    for (int i = 0; i < TABLE_SIZE; i++) {
        int idx = (h + i) % TABLE_SIZE; // Linear Probing
        if (!table[idx].occupied) {
            strncpy(table[idx].model_name, model_name, 63);
            table[idx].count = 1;
            table[idx].occupied = 1;
            unique_models_count++;
            return;
        }
        if (strcmp(table[idx].model_name, model_name) == 0) {
            table[idx].count++;
            return;
        }
    }
}
```

#### Engineering Takeaways

- **Avoiding the "DOM Trap":** Standard JSON libraries build a Document Object Model (DOM) tree in RAM, which often balloons to 5–10x the size of the raw file. For gigabyte-scale files, **streaming is not an optimization—it is a requirement**. By using a SAX-style stream parser, we processed data as a flow of tokens rather than a static structure.
- **Deterministic Memory Management in C:** By implementing a custom hash table with linear probing and a fixed prime size, we eliminated the need for dynamic heap allocations during the "hot path" of the loop. This not only prevents memory fragmentation but also ensures that the tool's performance is deterministic and cache-friendly.
- **Solving the Split-Object Problem:** Streaming data from compressed archives introduces the risk of objects being split across buffer boundaries. Implementing a dynamic accumulation buffer ensured that we only triggered the parser on complete, null-terminated JSON objects, maintaining data integrity without sacrificing speed.
- **Pipeline Efficiency:** The architecture follows a strict linear pipeline: `Compressed File`  `Decompression Buffer`  `Object Accumulator`  `Parser`. This decoupling allows each stage to be optimized independently and ensures that the memory footprint is limited to the size of a single JSON object, regardless of the total file size.