# Config & Common

## Config

### 结构定义

``` c++
class Configuration {
private:
	ptree pt;
	unordered_map<uint16_t, string> id2ip;
	unordered_map<string, uint16_t> ip2id;
	int ServerCount;
};
```

- ptree 结构来源于 boost 库，可将对系统的配置信息转换为具有层级信息的树状结构
- id2ip/ip2id 建立集群机器 ip 与 node id 之间的映射关系
- ServerCount 集群服务器数

### Config 构造

``` c++
Configuration::Configuration() {
	ServerCount = 0;
	read_xml("../conf.xml", pt);
	ptree child = pt.get_child("address");
	for(BOOST_AUTO(pos,child.begin()); pos != child.end(); ++pos) 
    {  
        id2ip[(uint16_t)(pos->second.get<int>("id"))] = pos->second.get<string>("ip");
        ip2id[pos->second.get<string>("ip")] = pos->second.get<int>("id");
        ServerCount += 1;
    }
}
```

调用 property_tree 的相关方法解析 XML 配置文件，获取 address tag 中各项 id 与 ip 的配置信息

## Common

### 基本结构定义

#### 文件位置相关

- 通过 `file_pos_tuple` 描述一个文件的物理位置，(node_id, offset, size)

- `file_pos_info` 应该是用于节点之间使用的一种 message 结构，保存了一组 `file_pos_tuple`

- `nrfsfileattr` 有点类似传统 FS 中的 inode，记录文件 mode (文件或目录)、size、修改/创建时间 time

#### 文件元数据相关

``` c++
typedef struct 
{
	NodeHash hashNode; /* Node hash array of extent. */
    uint32_t indexExtentStartBlock; /* Index array of start block in an extent. */
    uint32_t countExtentBlock; /* Count array of blocks in an extent. */
} FileMetaTuple;

typedef struct                          /* File meta structure. */
{
    time_t timeLastModified;        /* Last modified time. */
    uint64_t count;                 /* Count of extents. (not required and might have consistency problem with size) */
    uint64_t size;                  /* Size of extents. */
    FileMetaTuple tuple[MAX_FILE_EXTENT_COUNT];
} FileMeta;
```

#### 文件夹元数据相关

``` c++
typedef struct {
	char names[MAX_FILE_NAME_LENGTH];
	bool isDirectories;
} DirectoryMetaTuple;

typedef struct                          /* Directory meta structure. */
{
    uint64_t count;                 /* Count of names. */
    DirectoryMetaTuple tuple[MAX_DIRECTORY_COUNT];
} DirectoryMeta;
```

### 常量

- 文件路径最大长度 `MAX_PATH_LENGTH` = 255
- 文件名最大长度 `MAX_FILE_NAME_LENGTH` = 50
- 文件夹最大层数(?) `MAX_DIRECTORY_COUNT` = 60
- ??? `MAX_FILE_EXTENT_COUNT` = 20
- 文件系统分块大小 `BLOCK_SIZE` = 1MB

