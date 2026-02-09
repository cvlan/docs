# IP Allocation

How CloudVLAN assigns IP addresses to nodes within a CVLAN's CIDR range.

## Algorithm

![IP Allocation Flow](../images/ip-allocation-flow.svg)

The allocator uses a **dual-pointer** approach with two O(1) fast paths and an O(n) fallback:

### Fast Path 1: Reuse Gap (O(1))

When a node is deleted, its IP becomes a "gap." The `first_free` pointer tracks the lowest available gap. On the next allocation, this gap is reused immediately.

```
Allocation: Check first_free → skip if reserved → allocate → clear first_free
```

### Fast Path 2: Watermark (O(1))

If no gaps exist, increment the `next_offset` high-water mark. This is a simple counter that advances through the CIDR range.

```
Allocation: Increment next_offset → skip if reserved → allocate
```

### Slow Path: Gap Scan (O(n))

When the watermark reaches the end of the CIDR range and no `first_free` is set, scan all allocated IPs to find gaps. This only happens when the range is nearly full.

If no gaps found: return `409 Conflict` (CIDR exhausted).

## Data Model

Each CVLAN tracks its IP allocation state:

| Field | Type | Purpose |
|-------|------|---------|
| `next_offset` | u32 | High-water mark for new allocations |
| `first_free` | Option\<u32\> | Lowest free offset from deletions |
| `allocated_count` | i32 | Current number of allocated IPs |
| `reserved_ips` | TEXT | JSON array of reserved ranges |

## Reserved IPs

The first and last usable IPs in each CIDR are reserved (gateway + broadcast convention):

| CIDR Size | Reserved | Usable |
|-----------|----------|--------|
| /24 (256) | .1, .254 | 252 |
| /16 (65536) | .0.1, .255.254 | 65532 |
| /8 (16M) | .0.0.1, .255.255.254 | ~16M - 4 |

Reserved ranges can also be configured explicitly (e.g., `"10.0.0.3-7"` to reserve a block).

## Deletion Flow

When a node is deleted:
1. Release its IP offset
2. Decrement `allocated_count`
3. Set `first_free = min(current_first_free, released_offset)`

The next allocation will reuse the lowest available gap.

## CIDR Auto-Allocation

When creating a CVLAN without specifying a CIDR, the system auto-allocates from RFC 1918 ranges:

| Priority | Range | Available /24 blocks |
|----------|-------|---------------------|
| 1 | 10.0.0.0/8 | 256 |
| 2 | 172.16.0.0/12 | 16 |
| 3 | 192.168.0.0/16 | 1 |

Overlap detection uses bitwise range comparison. CIDRs are tenant-isolated — two tenants can use the same CIDR in different CVLANs.

## Concurrency

Each CVLAN has its own **per-CVLAN mutex** for IP allocation. This replaced an earlier retry-based approach:

| Approach | Throughput | Issues |
|----------|-----------|--------|
| Retry-based (SELECT FOR UPDATE) | ~5K ops/s | Deadlocks under contention |
| Per-CVLAN mutex | **21K ops/s** | Clean, no retries needed |

The mutex ensures that only one allocation happens per CVLAN at a time, but different CVLANs can allocate concurrently.

## Performance

Tested at 200 concurrent connections:
- **21,000 allocations/sec** sustained
- **p99 latency < 10ms**
- Zero allocation conflicts
