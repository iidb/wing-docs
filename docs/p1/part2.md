# Part 2

In part 2, you will implement compaction mechanism. You will implement: 

* `LeveledCompactionPicker`, which is derived from `CompactionPicker`. It generates a new compaction task. The parameters for the compaction task are stored in `Compaction`. It returns nullptr if no compaction is required for the LSM-tree.

* `CompactionJob`. This component receives an iterator and persists its output as a list of SSTables. It is utilized in both `DBImpl::FlushThread` and `DBImpl::CompactionThread`.

* `DBImpl::CompactionThread`. This thread manages the compaction process and updates the superversion. You may refer to the implementation of `DBImpl::FlushThread` for guidance.

## Leveled compaction strategy

You will implement the leveled compaction strategy. This strategy is the default strategy of RocksDB.

The leveled compaction strategy in Wing is as follows: Suppose the size ratio is $T$, and the size limit of Level 0 is $B$. The size limit of Level 1, 2, 3 $\cdots$ is $BT,BT^2,BT^3\cdots$ Level 0 consists of multiple sorted runs flushed by `DBImpl::FlushThread`, each sorted run has 1 or 2 SSTables. If the number of sorted runs is larger than 4, all the sorted runs are compacted to the next level. If the number of sorted runs reaches 20, then `DBImpl::FlushThread` stops to flush until they are compacted to the next level. Level 1, 2, 3 $\cdots$ consists of only one sorted run. If one of them reaches its capacity, it picks an SSTable with the minimum overlapping with the next level, and compact the SSTable with the next level.

In LeveledCompactionPicker::Get, you must check for the existence of a compaction task. If one exists, you must create and return a `std::unique_ptr<Compaction>`. Otherwise, return a nullptr.

## DBImpl::CompactionJob

You will implement CompactionJob::Run. It receives an iterator and writes the iterator's output to disk. You should merge the records with the same key. The output is divided into SSTables, with the size of records in each SSTable not exceeding the target SSTable size (refer to sst_file_size in lsm/options.hpp). The actual SSTable size may be larger than the target SSTable size since we have index data and metadata. 

## DBImpl::CompactionThread

The compaction thread awaits signals from the flush thread or the deconstructor via the condition variable compact_cv_. Upon being awakened, it first checks `stop_signal_`. If `stop_signal_` is true, it stops immediately. Otherwise, it calls `CompactionPicker::Get` to obtain a compaction task. If a task is present, it executes the compaction outside of the DB mutex. After compaction, it reacquires the DB mutex, creates a new superversion, and updates the DBImpl superversion. You can assume that the number of SSTables is small, allowing you to iterate through each SSTable and copy their pointers to the new superversion.

The DB mutex should only be used for metadata operations. You must avoid performing the compaction process under the DB mutex.

### Remove old SSTables

Utilize SetRemoveTag to set the `remove_tag_` to true. When the destructor of SSTable is invoked, it checks the tag and determines whether to remove the file. This is safe as the destructor is called only when the SSTable is not in use.

## Test

TODO