Answer these questions after reading the code
=============================================


## What is the file structure of leveldb?

## How to write a key into a LevelDB instance?

1. Put `DBImpl::Writer` into a wait queue `writers_`
2. After the Writer is in front, call `MakeRoomForWrite` to allocate space for the log. This may fail.
3. If (2) Success, create the new batch group by calling `DBImpl::BuildBatchGroup` which combines the pending write batch data into a whole batch.
4. After the batch is ready by (3), write the log with `log::Writer`
5. The log is then synced if `sync` option is enabled, and then the batch is also written to `memtable` by calling `WriteBatchInternal::InsertInto`
6. Handle errors if `sync_error`. Update the `last_sequence` number (**XXX(void001):** which seems to be the LSN?)
7. Iterate through the waiting queue, mark the previous items as __finished__ and notify the waiter


**Note that step 4 - 5 do not need to stay in the Global `mutex_` of `DBImpl`, since the IO is already guaranteed to be serialized by the current `DBImpl::Writer` and the wait queue**

__The step 7 is the final step of the Write call, Let me highlight one of the elegant design of the DBImpl::Write. It uses a writer queue and every thread waiting on this queue can help other writers write the data as well. See the following paragraph for an example__
Suppose we have the following writers_ queue
```TEX
(head)
[W0] <- [W1] <- [W2] <- [W3] <- [W4] <- [W5]
----------------------------
(Covered by BuildBatchGroup)
```
The current thread is processing [W0]. And when processing [W0], it found that the following 3 Writers can also be merged into this batch group (by calling `BuildBatchGroup`)
So now the last_writer is [W3], and all of W0-W4 is written to the log and memtable, so we definitely want to avoid writing them again. Therefore, when we are finishing the current thread's work, we iterate through the `writers_.front()` to `last_writer` and mark [W1], [W2], [W3], as finished and pop them out of the queue as well, note that we call `cv.Signal()` to wake up the corresponding writer thread for [W1], [W2], [W3] and notify them they can return.
Why don't we notify [W0]? Because we are the writer for [W0], we don't need to notify anyone. After the notify and dequeue, the queue becomes this

```TEX
(head)
[W4] <- [W5]
```

Finally, the current thread notify the waiter of [W4], which means it can now take its job.

### Sub-Questions

- [x] What is `DBImpl::BuildBatchGroup`'s job?
  + `BuildBatchGroup` will iterate through all the writers in the `writers_` deque, and merge the multiple `WriteBatch` into a big write batch. And return the aggregated batch as well as the last writer it touches to the caller

- [x] What is the benefit of having a `writer_` queue and allow batch group processing?
  + It reduces the number of IOs needed and thread context switching. Each thread can do more things (within the limit). And what's important, it doesn't violate the WriteBatch semantic.

- [x] What is `WriteBatchInternal::InsertInto`'s job?
 + It inserts the write batch into the `memtable`.

## How to read a key from the LevelDB instance?



## **Concurrency Control of LevelDB R/W operations (How does leveldb support concurrent read/write operations)**

## What's the thread model of LevelDB?

* [ ] What are the Condition Variables?
  - `DBImpl::background_work_finished_signal_` : Used to wait for some background work to be finished.

## How does LevelDB perform the compaction?

## What IO model does LevelDB use?

## How to recover a LevelDB database after restart the process?

## What are the major components (classes) of LevelDB?

* Storage-Related
  * MemTable

## What is the format of major data structures?

- [x] What is the format of a `WriteBatch`
  + A WriteBatch stores its data in `rep_` which is a string, and the `rep_` follows the format below:
  ```C
  WriteBatch
  --------------------------------------------------------------
  | Sequence (64bit) | Count (32bit) | Data (Array of Records) |
  --------------------------------------------------------------
  ```
  + Each `Record` can be either a `kTypeValue` or a `kTypeDeletion`

  ```C
  kTypeValue:
      key           value
  ------------------------------
  | varstr --- | varstr ---    |
  ------------------------------

  kTypeDeletion:
  --------------
  | varstr --- | 
  --------------
  ```

  + The format of `varstr` is shown below:

  ```C
  ----------------------------------------------------/
  | len varint32 (32bit) | 8bit | 8 bit | 8 bit | ... 
  ----------------------------------------------------/

  ```
  + `varint32` is a varaiable length `int32`, the logic of encoding the varint32 is in `leveldb/util/coding.cc` `EncodeVarint32`.




## Please detail each of the utility function which are used to encode the data

### EncodeVarint32

The function EncodeVarint32 is used to **compress** (mostly) the `uint32_t` to a at most 5 Bytes varint. The process is as follows:
The function encodes the `uint32` into another format, in this format, in each Byte of the `varint32`, the low 7 bits represents the actual data, the high 1 bit indicates whether there are more data in the next Byte, `1` means Yes, `0` means No.

Take Hex numer `0x0000460E`(32bit) as an example, The encoding process is as follows:

```C
raw binary number: 0100 0110 0000 1110 (omit the 16 leading zero bits)
                              ^0001110| the number is larger than 128 (1000 0000), encode 7 bits and mark the 8th bit as 1
encoded:           XXXX XXXX 1000 1110
raw binary number: 0100 0110 0--- ---- 
                     ^0001100|        | the number is larger than 2 << 14 (100 0000 0000 0000), encode next 7 bits and mark
encoded:      XXXX 1000 1100 1000 1110
raw binary number: 01-- ---- ---- ---- 
                                        the number is smaller than 2 << 21, directly store it and end the process
              0001 1000 1100 1000 1110

The result is 24 bit, which is smaller than original 32 bit
```

You may already notice that, the encoded data is not always smaller than the original data, since we use 1 bit in each byte to represent if there are more data or not. When the data exceeds 1 << 28, you need more space to hold the whole data, but we expect the varint should mostly be smaller than 1 << 28, and from the staticstic point of view, a uint32 data larger than 1 << 28 is rare, and the total size of the compressed varint32 should be smaller than using uint32 fixed length.

