





# 疑问:

## 1.   

```C++
class LEVELDB_EXPORT Cache {
 public:
  Cache() = default;  
// Insert a mapping from key->value into the cache and assign it
  // the specified charge against the total cache capacity.
  //
  // Returns a handle that corresponds to the mapping.  The caller
  // must call this->Release(handle) when the returned mapping is no
  // longer needed.
  //
  // When the inserted entry is no longer needed, the key and
  // value will be passed to "deleter".
  virtual Handle* Insert(const Slice& key, void* value, size_t charge,
                         void (*deleter)(const Slice& key, void* value)) = 0;


```

？在这些地方的为什么要是虚函数



## 2.  lru 和in_use 在什么用到了， 比如说lru里面的











# 一些要点：







## 1. 

参考： https://izualzhy.cn/leveldb-cache 



缓存里的一个`LRUHandle`对象可能经历这么几种状态：

1. 被手动从缓存删除/相同 key 新的 value 替换后原有的`LRUHandle*`/容量满时淘汰，此时缓存不应该持有该对象，随着外部也不再持有后应当彻底删除该对象。
2. 不满足1的条件，因此对象正常存在于缓存，同时被外部持有，此时即使容量满时，也不会淘汰。
3. 不满足1的条件，因此对象正常存在于缓存，当时已经没有外部持有，此时当容量满，会被淘汰。

这3个状态，就分别对应了以下条件：

1. `in_cache = false`
2. `in_cache = true && refs >= 2`
3. `in_cache = true && refs = 1`

其中满足条件3的节点存在于`lru_`，按照 least recently used 顺序存储，`lru_.next`是最老的节点，当容量满时会被淘汰。满足条件2的节点存在于`in_use_`，表示该 LRUHandle 节点还有外部调用方在使用。
*注：关于状态2，可以理解为只要当外部不再使用该缓存后，才可能被执行缓存淘汰策略*

## 2.  cache_test 里面看到每次insert之后都有release：。



为了保证`refs`值准确，无论`Insert`还是`Lookup`，都需要及时调用`Release`释放，使得节点能够进入`lru_`或者释放内存。


LRUHandle可能处于以下3种状态之一：

- 状态A: 在cache中(被`table_`引用)，并且有外部引用。即：`in_cache=true && refs>1`
- 状态B: 在cache中(被`table_`引用)，但没有外部引用。即：`in_cache=true && refs=1`
- 状态C: 没有在cache中，即`in_cache=false`
- 

猜测：

 （1） Erase 和 release是不同的， erase 应该就是真的要把一个key从table中删除。





```C++
  // Release a mapping returned by a previous Lookup().
  // REQUIRES: handle must not have been released yet.
  // REQUIRES: handle must have been returned by a method on *this.

void LRUCache::Release(Cache::Handle* handle) {
  MutexLock l(&mutex_);
  Unref(reinterpret_cast<LRUHandle*>(handle));
}



// If e != nullptr, finish removing *e from the cache; it has already been
// removed from the hash table.  Return whether e != nullptr.
bool LRUCache::FinishErase(LRUHandle* e) {
  if (e != nullptr) {
    assert(e->in_cache);
    LRU_Remove(e);
    e->in_cache = false;//TODO？？ 如果原本这个e是重复的key（），被缓存了， 现在变成不缓存，那么下面的Unref(e),真的有把这个e删除吗？，什么时候才是真正的删除呢？
    usage_ -= e->charge;
    Unref(e);
  }
  return e != nullptr;
}

  // If the cache contains entry for key, erase it.  Note that the
  // underlying entry will be kept around until all existing handles
  // to it have been released.

void LRUCache::Erase(const Slice& key, uint32_t hash) {
  MutexLock l(&mutex_);
  FinishErase(table_.Remove(key, hash));
}
```





(2)

```C++

//正常一个Insert的时候，一个Handle 在in_use, 当调用release的时候可以变成在lru,如果用了erase之后就会是移除了
Cache::Handle* LRUCache::Insert(const Slice& key, uint32_t hash, void* value,
                                size_t charge,
                                void (*deleter)(const Slice& key,
                                                void* value)) {

```







## 3. 感悟： 其实自己对于这个Refs 的引用计数的一直纠结以及不懂得原因就是因为自己没有从顶向下看， 比如没有弄清楚里面的Release  和 Erase的区别， 里面为什么Insert 里面要配套用Release。 （感觉这些东西自己早点拿别人的博客看来加深自己的想法就好了。

（1） https://www.yuanguohuo.com/2019/07/19/leveldb-cache/ 

（2） https://izualzhy.cn/leveldb-cache

（3）https://blog.csdn.net/u012658346/article/details/45486051





## 4. 



- 查找Table Cache，获取Index信息。

- - 代码：TableCache::Get-> TableCache::FindTable
  - 如果找到了，直接返回value，并获取到Table*指针。
  - 如果没有在缓存中找到文件，那么打开SSTable文件，将其index部分读入内存。



- 根据Table*查找Block Cache，获取具体block记录。

- - 代码：TableCache::Get-> Table::InternalGet -> Table::BlockReader
  - 通过index_block判断记录是否存在，否直接返回。
  - 如果存在，优先从Block Cache查找；找不到，则需要从文件获取到。





### 2.2. 使用

一个[sstable](https://izualzhy.cn/leveldb-sstable)的读取，通过[class Table](https://izualzhy.cn/leveldb-table)实现。

而[Open一个Table](https://izualzhy.cn/leveldb-table#31-open)，主要就是传入文件返回`Table*`，因此缓存的 key 就是文件的 file_number

```C++
  char buf[sizeof(file_number)];
  EncodeFixed64(buf, file_number);
  Slice key(buf, sizeof(buf));
```

value 就是文件和`Table*`

```C++
struct TableAndFile {
  RandomAccessFile* file;
  Table* table;
};
```

当需要读取一个 sstable 时，查看是否在 cache 中，如果在，则直接从 cache 中返回`TableAndFile*`，否则构造一个`TableAndFile*`插入到 cache 并返回。

```C++
      TableAndFile* tf = new TableAndFile;
      tf->file = file;
      tf->table = table;
      *handle = cache_->Insert(key, tf, 1, &DeleteEntry);
```





当数据过期淘汰时，关闭文件句柄，清理内存。

```C++
static void DeleteEntry(const Slice& key, void* value) {
  TableAndFile* tf = reinterpret_cast<TableAndFile*>(value);
  delete tf->table;
  delete tf->file;
  delete tf;
}
```

`Insert`时`charge`为1，代表一个文件，总的数目大小在初始化时通过`TableCacheSize(options_)`指定，因此就可以起到控制整个进程打开的文件句柄数目的作用。

注意`TableCache`有手动逐出`Evict`的操作，对应删除文件后删除对应缓存的场景。

```C++
void TableCache::Evict(uint64_t file_number) {
  char buf[sizeof(file_number)];
  EncodeFixed64(buf, file_number);
  cache_->Erase(Slice(buf, sizeof(buf)));
}
```









##  3. 总结

可以看到 leveldb 里是二级缓存，第一级存放`TableAndFile`，第二级存放`Block`，默认都使用 LRUCache，当然也可以自定义。