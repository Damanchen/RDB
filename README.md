## RDB

&emsp;&emsp;rdb是一个用于解析Redis的RDB文件的Go包。解析RDB格式参照[Redis RDB File Format](https://github.com/sripathikrishnan/redis-rdb-tools/blob/master/docs/RDB_File_Format.textile)   
&emsp;&emsp;该包是在[BrotherGao/RDB](https://github.com/BrotherGao/RDB)基础上修改。增加对RDB Version 9的支持，以及bug的修复。

RDB Version 8 改动部分：
*  Lua脚本可以持久化到RDB文件中，类型为RDB_OPCODE_AUX，以key-value的形式持久化。其中，key为"lua"，value为对应的脚本内容
*  增加RDB_TYPE_ZSET_2类型，浮点类型不在以字符串的形式保存，而是以binary形式保存到RDB中去
*  增加数据的长度增加RDB_64BITLEN类型
*  增加RDB_TYPE_MODULE类型，Redis 4.0引入Module模块。(该包不支持对该部分的解析)

RDB Version 9 改动部分：
* RDB可以包含keys的LRU或LFU (已实现 lru/lfu 字段的解析)
* 新建流数据类型Stream (暂时还未实现)
* RDB可以包含模块AUX字段 (该包不准备支持对该部分的解析)

## 使用
如下是示例程序部分代码
```go
type decoder struct {
	db int
	i  int
	nopdecoder.NopDecoder
}

func (p *decoder) StartDatabase(n int) {
	p.db = n
	fmt.Printf("Start parsing DB%d\n", p.db)
}

func (p *decoder) EndDatabase(n int) {
	fmt.Printf("Finish parsing DB%d\n", p.db)
}

func (p *decoder) Aux(key, value []byte) {
	fmt.Printf("aux_key=%q\n", key)
	fmt.Printf("aux_value=%q\n", value)
}

func (p *decoder) Set(key, value []byte, expiry int64) {
	fmt.Printf("db=%d %q -> %q ttl=%d\n", p.db, key, value, expiry)
}

func (p *decoder) Hset(key, field, value []byte) {
	fmt.Printf("db=%d %q . %q -> %q\n", p.db, key, field, value)
}

func (p *decoder) Sadd(key, member []byte) {
	fmt.Printf("db=%d %q { %q }\n", p.db, key, member)
}

func (p *decoder) StartList(key []byte, length, expiry int64) {
	p.i = 0
}

func (p *decoder) Rpush(key, value []byte) {
	fmt.Printf("db=%d %q[%d] -> %q\n", p.db, key, p.i, value)
	p.i++
}

func (p *decoder) StartZSet(key []byte, cardinality, expiry int64) {
	p.i = 0
}

func (p *decoder) Zadd(key []byte, score float64, member []byte) {
	fmt.Printf("db=%d %q[%d] -> {%q, score=%g}\n", p.db, key, p.i, member, score)
	p.i++
}

func (p *decoder) StartRDB() {
	fmt.Println("Start parsing RDB")
}

func (p *decoder) EndRDB() {
	fmt.Println("Finish parsing RDB")
}

func (p *decoder) GetKeys() int {
	decodedKeys := p.i
	return decodedKeys
}

func maybeFatal(err error) {
	if err != nil {
		fmt.Printf("Fatal error: %s\n", err)
		os.Exit(1)
	}
}

func main() {
	f, err := os.Open(os.Args[1])
	maybeFatal(err)
	err = rdb.Decode(f, &decoder{})
	maybeFatal(err)
}
```

运行结果如下：
```powershell
Start parsing RDB
aux_key="redis-ver"
aux_value="5.0.10"
aux_key="redis-bits"
aux_value="64"
aux_key="ctime"
aux_value="1610349359"
aux_key="used-mem"
aux_value="1902424"
aux_key="repl-stream-db"
aux_value="0"
aux_key="repl-id"
aux_value="e31f63605ddde5b2cbf8459f46a464c5b0fbc333"
aux_key="repl-offset"
aux_value="776"
aux_key="aof-preamble"
aux_value="0"
db: [0]
Start parsing DB0
dbSize: [4], expiresSize: [2]
lruIdle: [7574825]
db=0 "cfq3" -> "cfq3" ttl=0
lruIdle: [7574825]
db=0 "cfq2" -> "cfq2" ttl=0
rdbFlagExpiryMS: [1610435732823]
lruIdle: [22]
db=0 "cfq4" -> "cfq4" ttl=1610435732823
rdbFlagExpiryMS: [1610435330609]
lruIdle: [7573292]
db=0 "cfq" -> "cfq" ttl=1610435330609
Finish parsing DB0
Finish parsing RDB
rdbFlagEOF

```
具体参见example目录下的test.go文件。

v8rdb 目录下为用于测试的 v8 版本的rdb文件；v9rdb 目录下为用于测试的 v8 版本的rdb文件

## 安装

```
go get github.com/Damanchen/RDB
```

