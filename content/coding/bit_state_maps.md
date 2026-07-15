---
title: "Processing Billions of Integers: A Lesson in Memory Efficiency and Bit Manipulation"
date: 2026-07-07T10:45:00+00:00
draft: false
categories:
  - Engineering
  - Coding
tags:
  - C++
  - Systems Programming
  - Performance Optimization
  - Linux
  - Bit Manipulation
description: How to analyze 4GB of raw binary metadata using only 1GB of RAM by leveraging mmap and a 2-bit state machine.
series: High Performance Systems
slug: processing-billions-of-integers-memory-efficiency
distribution:
  linkedin:
    status: posted
    payload_snippet: analyze 4GB of raw binary metadata 
    link_posted: ""
  reddit:
    status: pending
    target_subreddits:
      - cpp
      - systems
    link_posted: ""
  grimm_network:
    status: pending
    thread_id: ""
---

# Processing Billions of Integers: A Lesson in Memory Efficiency and Bit Manipulation

### The Challenge
In storage systems, we often deal with raw binary dumps of metadata tables. When debugging consistency issues, you may need to scan billions of 32-bit identifiers to find duplicates or unique entries. Because these files can be massive, loading them into a standard hash map is impossible.

The task seems simple on the surface: given a binary file containing 1 billion 32-bit integers, count how many are unique and how many appear exactly once.

However, at this scale (4GB of raw data), the "naive" approach fails immediately. If you were to use a `std::unordered_set<uint32_t>` or a `std::map`, the memory overhead per element (pointers, buckets, and node metadata) would balloon from 4 bytes to roughly 24–32 bytes per integer. You would need over 30GB of RAM just to store the set—an unacceptable requirement for most production environments.
### Real-world Context
In a real production environment, these integers aren't just random numbers—they are typically **Logical Block Addresses (LBAs)** or **Hash Fingerprints**. For example, in a deduplication engine, you might scan billions of fingerprints to calculate storage efficiency. If a fingerprint appears only once, it's a unique block; if it appears multiple times, the system can save space by storing only one copy.

In systems like S3 or Ceph, data is stored as objects. There is often a mapping layer that translates an `ObjectID` to a physical location on a server. During a migration or a "garbage collection" (GC) cycle, you might dump the current mapping table to analyze fragmentation.

### Strategy 1: Zero-Copy I/O with `mmap`

To avoid the overhead of repeated system calls and copying data from kernel space to user space, the best approach is **Memory Mapping (`mmap`)**.

By mapping the file directly into the process's address space, we treat the binary file as if it were a massive array in memory. The OS handles the paging and read-ahead optimizations, which is significantly faster than using `std::ifstream` or `fread`.

**The "Interesting Part": Mapping the Binary**

```cpp
int fd = open(filename.c_str(), O_RDONLY);
struct stat st;
fstat(fd, &st);

// Map the file as a read-only byte array
void* mapped = mmap(nullptr, st.st_size, PROT_READ, MAP_PRIVATE, fd, 0);
const uint32_t* data = static_cast<const uint32_t*>(mapped);

// Now we can iterate through the file as a simple C-array
for (size_t i = 0; i < st.st_size / sizeof(uint32_t); ++i) {
    process(data[i]);
}
```

### Strategy 2: The 2-Bit State Machine

Since we are dealing with `uint32_t`, there is a finite universe of possible values (). Instead of storing the numbers themselves, we can store the **state** of each number in a bitmap.

To distinguish between "not seen," "seen once," and "seen multiple times," 1 bit isn't enough. We need **2 bits per possible value**:

- `00`: Not seen
- `01`: Seen exactly once
- `10`: Seen more than once

This reduces the memory footprint to a constant , regardless of whether the input file is 4GB or 400GB.

**The "Interesting Part": Bit-Packing Logic**  
The challenge here is calculating exactly which bit in which byte corresponds to a specific integer. We use bit-shifting and masking to update the state without affecting neighboring values.

```cpp
void mark_seen(uint32_t val) {
    size_t byte_idx = static_cast<size_t>(val) / 4; // 4 numbers per byte (2 bits each)
    int bit_offset = (val % 4) * 2;                // Offset: 0, 2, 4, or 6
    
    // Extract the current 2-bit state using a mask (0x03 is binary 11)
    uint8_t current_state = (bitmap[byte_idx] >> bit_offset) & 0x03;

    if (current_state == 0) {
        // Transition: Not seen -> Seen once (set to 01)
        bitmap[byte_idx] |= (1 << bit_offset);
    } else if (current_state == 1) {
        // Transition: Seen once -> Multiple (set to 10)
        bitmap[byte_idx] &= ~(3 << bit_offset);    // Clear the bits first
        bitmap[byte_idx] |= (2 << bit_offset);     // Set state to 10
    }
}
```

### Strategy 3: The Final Tally

Once the file is processed, we perform a single linear pass over our 1GB bitmap. We check every 2-bit pair and increment our counters based on whether the state is non-zero (unique) or exactly one (seen once).

**The "Interesting Part": Efficient Counting**

```cpp
for (uint8_t byte : bitmap) {
    for (int shift = 0; shift < 8; shift += 2) {
        uint8_t state = (byte >> shift) & 0x03;
        if (state != 0) unique_count++;
        if (state == 1) once_count++;
    }
}
```

### Engineering Takeaways

By shifting the perspective from "storing data" to "tracking state," we achieved:

- **Time Complexity:**  where  is the number of integers. We only read the file once.
- **Space Complexity:**  relative to input size. The memory usage is fixed at 1GB, ensuring the application won't crash due to OOM (Out of Memory) errors on larger datasets.
- **I/O Efficiency:** By using `mmap`, we minimized context switching and leveraged the OS page cache for maximum throughput.