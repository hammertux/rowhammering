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
