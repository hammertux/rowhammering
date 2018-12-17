# How is DRAM Organised?

## Intro
Usually, DRAM is connected to the CPU through a **channel**. Modern Organisations have more channels (e.g., dual-channel). A DRAM has two "sides" known as **ranks**. The front of the DRAM is _rank-0_ and the back is _rank-1_. A DRAM contains various chips in an organisation like the following:

![DRAM Chip](https://github.com/andreadidio98/rowhammering/blob/master/DRAM%20chip.png?raw=true)

A chip is subdivided in multiple banks and each bank is subdivided in various rows. Usually each row is 8KB and stores data in bits in physical memory.

## <a name="reading"></a>Reading from DRAM
When the CPU wants to access a row in memory (e.g., row 1), we have to **activate the row** and the activated row is **copied to the row buffer**, then the value _from the row buffer_ is returned to the CPU. If the CPU wants to access a different row, the process starts again, _evicting_ the previous row from the row buffer. I.e., the row buffer acts like a cache for rows. If the CPU wants to access a row that is in the row buffer, we have a **row hit**, if the row is NOT in the row buffer, there is a **row conflict**.

## Refreshing the DRAM
Constraint from the physical world:
- The cells leak charge over time.
- Content of the cells has to be **refreshed** repetitively to keep data integrity.

To refresh the DRAM, the process is: The data from the DRAM cells is read **into the row buffer**, and then the same data is **written back to the cells**. DDR3 and DDR4 have some standards that specify the maximum interval between refreshes to guarantee data integrity.

### Problem:

**Cells leak faster upon proximate accesses.** This means that if we access two neighboring cells, the surrounding cells leak charge faster, meaning that the next refresh might not be fast enough to refresh the cells and keep data integrity.

# Rowhammer

> It's like breaking into an apartment by repeatedly slamming the neighbor's door until the vibrations open the door you were after. - Motherboard Vice

Let's say for example, we want to flip bits in _row 2_, what we can do is activating intermittently _rows 1 & 3_. The whole process explained [above](#reading) is repeated for every activation of the two rows. By doing this long enough, we could have bit flips in row 2. (given that the memory module on the machine is vulnerable!)

## How can we flip bits?

In order to exploit the rowhammer vulnerability, the memory accesses _MUST_ be:
- **uncached** (i.e., every access must physically reach the DRAM)
- **fast** (we want to have as many accesses between row refreshes, i.e., we are _racing_ against the next row refresh)
- **targeted** (we need to reach two specific rows to have the bit flips in the middle row)

The CPU cache lies _between_ the CPU core and the DRAM, therefore only **non-cached accesses** actually reach the DRAM. There are two choices we can make:
1. _Flush_ the cache after having put the data in the cache.
2. Don't put the data in the cache in the first place.

There are four _access techniques_ to achieve the goal of having the next access being served directly from DRAM:
1. **CLFLUSH** instruction (x86) (Kim et al. 2014)
2. **Cache eviction** (Aweke et al. 2016)
3. **Non-temporal accesses** (Qiao et al. 2016)
4. **Uncached memory** (Veen et al. 2016)

In the _first_ access technique we start by accessing the data, which is loaded in the cache and then we flush it from cache (clflush), and then we loop indefinitely in a **reload-flush** sequence until we get bit flips.

```assembly
hammer:
  mov (X), %eax //Read from address X
  mov (Y), %ebx //Read from address Y
  clflush (X) //Flush cache for address X
  clflush (Y) //Flush cache for address Y
  jmp hammer //Loop indefinitely
```

For the "hammer" routine to work (i.e., cause bit flips), addresses X and Y _MUST_ map to **different rows** of DRAM in the **same bank**.

In the _second_ access technique, Aweke et al. showed that an attacker can force the cache to invalidate its content by accessing memory addresses belonging to the **same cache eviction set**. On modern processors, last-level caches have **very high associativity** (8- or 16-way). Therefore, many memory accesses are required to evict a cache block, and this slows down the rowhammering process (NOT ideal). In this approach also the replacement policy of the cache matters (e.g., on modern caches replacement policy is usually NOT LRU).


In the _third_ access technique we use non-temporal accesses, i.e., when we access data once and NOT in the immediate future, and therefore the data is not put into the cache (low temporal-locality data) because it would be evicted anyways. Hence, we can use **"Non-Temporal Access (NTA) instructions"** in order to **bypass the cache**, which are there to minimise _cache pollution_. All the non-temporal stores to a single address are combined in one **Write-Combining (WC) buffer**, and _ONLY_ the LAST write goes straight to DRAM (no matter how many stores) meaning that the rate would not be sufficient to "hammer". The trick (Qiao et al.) is to follow the non-temporal store by a cache access to the same address.

```assembly
hammer:
  movnti %eax, (X)
  movnti %eax, (Y)
  mov %eax, (X)
  mov %eax (Y)
  jmp hammer
```


The _fourth_ access technique is good especially on mobile devices:
- On ARMv7 the flush instruction is **privileged**
- Cache eviction seems to be too slow
- On ARMv8 non-temporal stores are still cached in practice.

Since v4.0, Android has been using ION memory management. Apps can use the interface _/dev/ion_ for **uncached**, physically contiguous memory, and **no privilege and permissions** are needed (Veen et al.).















# Sources

1. [Google Project Zero Rowhammer](https://googleprojectzero.blogspot.com/2015/03/exploiting-dram-rowhammer-bug-to-gain.html)
2. [RuhrSec 2017: "Rowhammer Attacks: A Walkthrough Guide", Dr. Clémentine Maurice & Daniel Gruss](https://www.youtube.com/watch?v=-33gCDrSl_Q)
