# Part 2

In part 2, you will implement compaction mechanism. You will implement: 

* `CompactionJob`. This component receives an iterator and persists its output as a list of SSTables. It is utilized in both `DBImpl::FlushThread` and `DBImpl::CompactionThread`.

* `LeveledCompactionPicker`, which is derived from `CompactionPicker`. It generates a new compaction task. The parameters for the compaction task are stored in `Compaction`. It returns nullptr if no compaction is required for the LSM-tree.


* `DBImpl::CompactionThread`. This thread manages the compaction process and updates the superversion. You may refer to the implementation of `DBImpl::FlushThread` for guidance.

You may need to use `ulimit` to increase the number of open file descriptors. (by default it is 1024, which is not enough).

## DBImpl::CompactionJob

You will implement CompactionJob::Run. It receives an iterator and writes the iterator's output to disk. You should merge the records with the same key. The output is divided into SSTables, with the size of data blocks in SSTable not exceeding the target SSTable size (refer to `sst_file_size` in `storage/lsm/options.hpp`). The actual SSTable size may be larger than the target SSTable size since we have index data, bloom filter and metadata. 

The code is in `compaction_job.hpp`.

### Test

You can test it through `test/test_lsm --gtest_filter=LSMTest.CompactionBasicTest`

## Leveled compaction strategy

You will implement the leveled compaction strategy. This strategy is the default strategy of RocksDB.

The leveled compaction strategy in Wing is as follows: Suppose the size ratio is $T$, and the size limit of Level 0 is $B$. The size limit of Level 1, 2, 3 $\cdots$ is $BT,BT^2,BT^3\cdots$ Level 0 consists of multiple sorted runs flushed by `DBImpl::FlushThread`, each sorted run has 1 or 2 SSTables. If the number of sorted runs is larger than 4, all the sorted runs are compacted to the next level. If the number of sorted runs reaches 20, then `DBImpl::FlushThread` stops to flush until they are compacted to the next level. Level 1, 2, 3 $\cdots$ consists of only one sorted run. If one of them reaches its capacity, it picks an SSTable with the minimum overlapping with the next level, and compact the SSTable with the next level.

In LeveledCompactionPicker::Get, you must check for the existence of a compaction task. If one exists, you must create and return a `std::unique_ptr<Compaction>`. Otherwise, return a nullptr.

The code is in `compaction_picker.hpp`, `compaction_picker.cpp`, `compaction.hpp`.


## DBImpl::CompactionThread

### Put/Del operation

The `Put/Del` operation inserts a record into the LSM-tree. The key is converted to an internal key by adding the current sequence number and record type (`RecordType::Value` or `RecordType::Deletion`). Initially, it is inserted into the memtable. If the memtable reaches its capacity, it is appended to the list of immutable memtables. Then it nofities the flush thread. Then it installs the superversion (refer to `DBImpl::SwitchMemtable`).

The flush thread waits for the signal and fetches the list of immutable memtables. It flushes them (through `CompactionJob`) to sorted runs. Once the sorted runs are created, it creates a new superversion, installs the superversion, and notifies the compaction thread (refer to `DBImpl::FlushThread`). 

The compaction thread waits for the signal from the flush thread. Upon awakening, it checks the levels and attempts to find a compaction task (refer to `CompactionPicker::Get` and `Compaction` in `lsm/compaction_pick.hpp` and `lsm/compaction.hpp`). For example, it may find that Level 0 is too large, so it creates a compaction task: compacting some SSTable files from Level 0 to Level 1. After new SSTable files are created, it creates a new superversion, installs the superversion and looks for compaction tasks again. It is possible that multiple compactions occur when one immutable memtable is flushed. It also add removal tags to useless SSTables, so that they will be removed while destruction. 

### Locks in LSM-tree

There are 3 locks in LSM-tree. We use concurrency primitives in C++, including `std::mutex`, `std::unique_lock`, `std::shared_mutex`(C++17), `std::shared_lock`(C++17), `std::condition_variable`.

The first is `db_mutex_`. This mutex is used to protect the process of operating metadata (e.g. switching memtables, creating new superversion and installing new superversions).

The second is `write_mutex_`. This mutex is used to protect `Put` operations. 

The third is `sv_mutex_`. This mutex is used to protect the reference and installation of the superversion pointer.

### Multiversion Concurrency Control in LSM-tree

In LSM-tree, `Put` operations do not block `Get`/`Scan` (implemented by `DBIterator`) operations. This is achieved by supporting multiple superversions simultaneously. Each superversion has a reference count which is maintained by `std::shared_ptr`, the SSTables and the sorted runs in one superversion will not be removed if the reference count does not decrease to 0. For `Get` and Scan operations, a superversion is referenced, and the data accessed within this superversion remains consistent regardless of new superversions created by the `Put`/`FlushThread`/`CompactionThread`.

### Task

You will implement `DBImpl::CompactionThread`.

The compaction thread awaits signals from the flush thread or the deconstructor via the condition variable `compact_cv_`. Upon being awakened, it first checks `stop_signal_`. If `stop_signal_` is true, it stops immediately. Otherwise, it calls `CompactionPicker::Get` to obtain a compaction task. If a task is present, it executes the compaction outside of the DB mutex. After compaction, it reacquires the DB mutex, creates a new superversion, and updates the DBImpl superversion. You can assume that the number of SSTables is small, allowing you to iterate through each SSTable and copy their pointers to the new superversion. You also need to set `compaction_flag_` to true if you are not waiting for `compact_cv_`, like `flush_flag_` in `DBImpl::FlushThread`. 

More specifically, you need to write something like:

```c++
void DBImpl::CompactionThread() {
  while (!stop_signal_) {
    std::unique_lock lck(db_mutex_);
    // Check if it has to stop. 
    // It has to stop when the LSM-tree shutdowns.
    if (stop_signal_) {
      compact_flag_ = false;
      return;
    }
    std::unique_ptr<Compaction> compaction = /* A new compaction task */
    if (!compaction) {
      compact_flag_ = false;
      compact_cv_.wait(lck);
      continue;
    }
    compact_flag_ = true;
    // Do some other things
    db_mutex_.unlock();
    // Do compaction
    db_mutex_.lock();
    // Create a new superversion and install it
  }
}
```

The DB mutex should only be used for metadata operations. You must avoid performing the compaction process under the DB mutex.

The code is in `lsm.cpp` and `lsm.hpp`.

### Remove old SSTables

Utilize SetRemoveTag to set the `remove_tag_` to true. When the destructor of SSTable is invoked, it checks the tag and determines whether to remove the file. This is safe as the destructor is called only when the SSTable is not in use.

### Test

You can test all the components through `test/test_lsm --gtest_filter=LSMTest.LSMBasicTest:LSMTest.LSMSmallGetTest:LSMTest.LSMSmallScanTest:LSMTest.LSMSmallMultithreadGetPutTest:LSMTest.LSMSaveTest:LSMTest.LSMBigScanTest:LSMDuplicateKeyTest:LSMTest.LeveledCompactionTest`

or, you can just use `test/test_lsm --gtest_filter=LSMTest.LSM*:LSMTest.LeveledCompactionTest`