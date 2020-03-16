



1.  参考： https://www.cnblogs.com/haippy/archive/2011/12/04/2276064.html  （好的博客。）







# 一些警示：

## （1） 明明那个别人的网上的博客说leveldb这个东西看1 week就可以非常懂了。自己居然花费了那么多的时间，这和自己总是想一些没用的东西以及自己总是去关注各种没有什么意义的时事新闻有关。。自己没有非常好地珍惜自己的时间🚪】。（其实因为自己只要） ==(自己可能需要非常激动地看代码  以及时刻想着自己正在面临的压力了。==       

==:123==: 

==高亮 显示==：== +字体 +==



## （2）：自己在前期看带代码的时候明显感觉到自己走了很多的弯路，比如直接看代码的时候没有注意里面的release 和erase对于lru 的使用，以及自己没有从顶层去注意那些东西。 还有就是自己配置docker  链接clion 都弄了那么久。而看了那么多的东西自己也没有怎样动手。

















#  疑问：

## 1.



# 笔记要点：

## 1.  C++中构造函数或析构函数定义为private

参考： https://blog.csdn.net/vivian187/article/details/93043070 







``` C++


class LEVELDB_EXPORT Table {
 private:
  friend class TableCache;
  //TODO??  应该怎么去理解这种没有在类内定义的struct？
  struct Rep;

  static Iterator* BlockReader(void*, const ReadOptions&, const Slice&);

  //TODO??? 为什么外面当可以调用 new Table ？ 这个地方不是说private里面的函数吗？
  explicit Table(Rep* rep) : rep_(rep) {}

}
Status Table::Open(const Options& options, RandomAccessFile* file,
                   uint64_t size, Table** table) {

		*table = new Table(rep);//这个Table的构造函数 是private 的，但是只要不是实例化就可以调用，并且返回一个指针*
    (*table)->ReadMeta(footer);
}
```





## 2 。参考：https://www.cnblogs.com/liuhao/archive/2012/11/29/2795455.html 

   [LevelDB](http://code.google.com/p/leveldb/)是Google开源的持久化KV单机存储引擎，据称是HBase的鼻祖[Bigtable](http://wenku.baidu.com/view/176e3f0d4a7302768e99399c.html)的重要组件tablet的开源实现。针对存储面对的普遍随机IO问题，LevelDB采用merge-dump的方式，将逻辑场景的随机写请求转换成顺序写log和写memtable的操作，由后台线程根据策略将memtable持久化成分层的sstable。针对读请求，LevelDB会首先查找内存中的memtable和imm（不可变的memtable），然后逐层查找sstable。

   为了加快查找速度，LevelDB在内存中采用Cache的方式，在sstable中采用bloom filter的方式，尽最大可能减少随机读操作。

 



   LevelDB的Cache分为两种，分别是table cache和block cache。table cache缓存的是sstable的索引数据，类似于文件系统中对inode的缓存；block cache是缓存的block数据，block是sstable文件内组织数据的单位，也是从持久化存储中读取和写入的单位；由于sstable是按照key有序分布的，因此一个block内的数据也是按照key紧邻排布的（有序依照使用者传入的比较函数，默认按照字典序），类似于Linux中的page cache







 table cache默认大小是1000，注意此处缓存的是1000个sstable文件的索引信息，而不是1000个字节。table cache的大小由options.max_open_files确定，其最小值为20-10，最大值为50000－10。

 







   1) 首先从block cache中查找block，如果找不到，直接从持久化存储中读取，获取到一个新的block，插入block cache;

   2) 对于查到的block，返回对应的迭代器（LevelDB中，所有的get\merge操作均是抽象成iterator实现的）；

   3）如果仔细读代码，iter->RegisterCleanup函数实现会有点绕，这个函数在iter析构时被调用，执行注册的ReleaseBlock，ReleaseBlock调用cache_handle的Unref方法，对cache中缓存的block减少一个引用计数；cache执行insert函数时，会给所有的LRUHandle的引用计数设成2，其中1用于LRUCache自身，在执行cache的Release操作时被Unref，从而真正释放。











## 三、Table Cache  

基于LRUCache实现。key-value如下：

![img](https://pic1.zhimg.com/80/v2-a15e0f8d4de042656d1814f0b997fddc_1440w.jpg)

## ** LevelDb日知录之九** **levelDb中的****Cache**

　　书接前文，前面讲过对于levelDb来说，读取操作如果没有在内存的memtable中找到记录，要多次进行磁盘访问操作。假设最优情况，即第一次就在level 0中最新的文件中找到了这个key，那么也需要读取2次磁盘，一次是将SSTable的文件中的index部分读入内存，这样根据这个index可以确定key是在哪个block中存储；第二次是读入这个block的内容，然后在内存中查找key对应的value。

　　levelDb中引入了两个不同的Cache:Table Cache和Block Cache。其中Block Cache是配置可选的，即在配置文件中指定是否打开这个功能。

![img](Block-cache%20&&%20Table-cache.assets/2011121116391556.png)

图9.1 table cache

 　图9.1是table cache的结构。在Cache中，key值是SSTable的文件名称，Value部分包含两部分，一个是指向磁盘打开的SSTable文件的文件指针，这是为了方便读取内容；另外一个是指向内存中这个SSTable文件对应的Table结构指针，table结构在内存中，保存了SSTable的index内容以及用来指示block cache用的cache_id ,当然除此外还有其它一些内容。

 

- key值是SSTable的文件名称

- Value部分包含两部分

- - RandomAccessFile*: 指向磁盘打开的SSTable文件的文件指针。
  - Table*: 是指向内存中这个SSTable文件对应的Table结构指针，table结构在内存中，保存了SSTable的index内容以及用来指示block cache用的cache_id。

- 

```C++
从这个可以看出来去除一
void TableCache::Evict(uint64_t file_number) {
  char buf[sizeof(file_number)];
  EncodeFixed64(buf, file_number);
  cache_->Erase(Slice(buf, sizeof(buf)));
}
```



## 四、Block Cache

![img](https://pic4.zhimg.com/80/v2-7735891ae5754f6e7efb2e1e0ea2355b_1440w.jpg)

结构如下：

- key：文件的cache_id + 这个block在文件中的offset。
- value：Block的内容。

