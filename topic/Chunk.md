# Chunk

- Chunk 是列数据的集合，用于在内存中连续存储同一列的数据，以降低内存分配开销、降低内存占用以及实现内存使用量的统计/控制

  ```go
  /// util/chunk/chunk.go
  type Chunk struct {
      // 指明选择的行，如果为空，则表示全选
  	sel []int
  	// 列数据的集合
  	columns []*Column
      // ...............................
  }
  
  /// util/chunk/column.go
  type Column struct {
  	length     int // 用来表示有多少行数据
  	nullBitmap []byte // bit 0 is null, 1 is not null，存储列中每个元素是否为 null
  	offsets    []int64 // 变长数据在 data 中的偏移量
  	data       []byte // 数据存储，包括了定长和变长的数据
  	elemBuf    []byte // 定长数据在读写时用来 encode 和 decode
  }
  
  /// util/chunk/row.go
  type Row struct {
  	c   *Chunk
  	idx int // rowID
  }
  ```

  - 支持 read-only，只支持追加写
  - 用 Chunk 格式表示内存中的数据，行是抽象概念，数据是以列为单位来存储，通过 rowID 来关联到对应的行
  - 列中的数据使用字节数组来存储，避免了序列化和反序列化的开销

- 待存储的数据类型可以分为三类：定长类型、变长类型和空值；

  - 定长类型：`ETInt`（整形）、`ETReal`（浮点数）、`ETDecimal`（Decimal）、`ETDatetime`（时间）、`ETTimestamp`（时间戳）、`ETDuration`（周期）
  - 变长类型：`ETString`（字符串）、`ETJson`（json格式）

- 写入定长类型：

  - 对于定长类型值的写入，需要用到 `Column.data`、`Column.elemBuf` 和 `Column.nullBitmap` 三个字段

  - 定长列的创建：

    ```go
    // elemLen：表示数据类型的长度
    // cap：表示列中存了多少数据（即行数）
    func newFixedLenColumn(elemLen, cap int) *Column {
    	return &Column{
    		elemBuf:    make([]byte, elemLen),
    		data:       make([]byte, 0, cap*elemLen),
    		nullBitmap: make([]byte, 0, (cap+7)>>3),
    	}
    }
    ```

    - `elemBuf` 是一个提前分配好的数组，其大小等于对应列的数据类型的大小，通过复用 `elemBuf` 来避免重复分配内存

  - 写入步骤：以 `AppendInt64` 为例

    ```go
    func (c *Column) AppendInt64(i int64) {
    	*(*int64)(unsafe.Pointer(&c.elemBuf[0])) = i
    	c.finishAppendFixed()
    }
    
    func (c *Column) finishAppendFixed() {
    	c.data = append(c.data, c.elemBuf...)
    	c.appendNullBitmap(true)
    	c.length++
    }
    
    // 更新位图
    func (c *Column) appendNullBitmap(notNull bool) {
    	idx := c.length >> 3
    	if idx >= len(c.nullBitmap) {
    		c.nullBitmap = append(c.nullBitmap, 0)
    	}
    	if notNull {
    		pos := uint(c.length) & 7
    		c.nullBitmap[idx] |= byte(1 << pos)
    	}
    }
    ```

    - 将待写入的数据复制到 `elemBuf` 中
    - 将 `elemBuf` 中的数据追加到 `data` 中
    - 更新位图

- 写入变长类型：

  - 对于变长类型的写入，需要用到`Column.data`、`Column.offset` 和 `Column.nullBitmap` 三个字段

  - 变长列的创建：

    ```go
    func newVarLenColumn(cap int, old *Column) *Column {
        // 由于长度未知，在首次初始化时使用一个经验值 8 来作为长度
    	estimatedElemLen := 8
        // 后续再次调用时，使用之前数据长度 * 1.125 来作为估计值
    	if old != nil && old.length != 0 {
    		estimatedElemLen = (len(old.data) + len(old.data)/8) / old.length
    	}
    	return &Column{
    		offsets:    make([]int64, 1, cap+1),
    		data:       make([]byte, 0, cap*estimatedElemLen),
    		nullBitmap: make([]byte, 0, (cap+7)>>3),
    	}
    }
    ```

    - `offsets` 表示偏移量，用来记录下一条记录的写入位置

  - 写入步骤：以 `AppendString` 为例：

    ```go
    func (c *Column) AppendString(str string) {
    	c.data = append(c.data, str...)
    	c.finishAppendVar()
    }
    
    func (c *Column) finishAppendVar() {
    	c.appendNullBitmap(true)
    	c.offsets = append(c.offsets, int64(len(c.data)))
    	c.length++
    }
    ```

    - 将数据复制到 `data` 中
    - 更新位图
    - 将变长数据的长度追加到 `offsets` 中，作为下一条数据的起始位置

- 写入 NULL 值：

  ```go
  func (c *Column) AppendNull() {
  	c.appendNullBitmap(false)
  	if c.isFixed() {
  		c.data = append(c.data, c.elemBuf...)
  	} else {
  		c.offsets = append(c.offsets, c.offsets[c.length])
  	}
  	c.length++
  }
  ```

  - 在位图中写入一个 0，表示当前位置对应的数据为 NULL
  - 对于定长数据，在 `data` 中填入一个当前数据类型的长度来占位
  - 对于变长数据，使用当前数据长度对应的偏移量（即之前数据占用的偏移量）来更新 `offsets`

- 引入 Chunk 格式的优点：

  - 每次函数调用都会返回一批数据，数据量由 `tidb_max_chunk_size` 变量来控制，默认为 1024
    - OLAP 友好
    - 对于 OLTP 由于不需要返回那么多的数据，需要改进
  - 内存使用高效，减轻 gc 压力
  - 缓存友好，充分利用 CPU 的 pipeline
  - 内存监控和控制更加方便

## 参考

- https://pingcap.com/blog-cn/tidb-source-code-reading-10/



