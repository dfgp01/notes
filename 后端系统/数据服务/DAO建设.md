
代码语言：golang
依赖框架：gorm
目标：从定义一个User实体开始，实现dao的CRUD操作，要加上service层的接口，达到安全稳定的操作

# 适用数据库

- RDB类，含mysql、postgre等
	- 数据以行为单位，需原子化关联的，需事务一致的
- nosql类，含mongodb等
	- 适用于不需要事务、不需要强关联的表，例如：
		  - 全局字典表：`label:"goldmind", value:"1", category:"game_type"`
		  - 多语言对照表：`key: "tips", zh-cn:"提示", en:"message tip"`
		  - 公共修改日志表：记录系统中的每一次增删改操作。
	- 特点：这些表通常不需要 ID 关联，没有外部关联，也不需要复杂的查询或统计。

### RDB(Mysql)建表规范

	- 采用gorm的规范来建表，每个实体默认继承 `gorm.Model`，`deleted_at` 造成的列和索引浪费可以忽略不计
	- 每个表都有一个自增 ID（实体继承`gorm.Model`即可），不与业务挂钩
	- 每个主表有自己的业务ID，`string`类型
		- 例如 users 表的 account（可以是 UUID、A12345，甚至是数字字符串 50001 等）
	- 有些表可能包含 `status` 或 `type` 字段，建议用enum，业务层通过全局字典管理。
	- 考虑到主从同步问题，可以新增字段 version，正整数递增，从0开始
	- 若表中含有其他表的外键，则使用 `{table}_id` 格式，例如 `user_id` 表示 user 表的 ID。
		- 建议为外键字段创建 B-tree 索引，由 Service 层检查合法性，避免数据库负担过多。
	- 先从一个User实体试试

# 公共表

- 数据库信息表：db_infos
	- **需要手动添加，或框架中加入检测**
```json
{
	"_id" : "bson.id",
  "name": "db-test1",
  "dsn": "database-addr:port",
  "addr": "ip:port",
  "type": "mysql",
  "last_time" : unix(),
  "extra": {  }	//未決定結構
}
```

- 修改信息日志表：change_logs
	- 最好能有tx-id，因爲涉及多表修改事務，有關聯的ID和是否使用了事務

```json
{
	"db_info_id": "bson.id"
	"table_name": "users"
	"data_id": 1234			//users表的主鍵ID
	"action": "create update delete"
	"contents": [{"field": "account", field_type:"string", "before": any, "after": any}]
	"create_time", unix()
}
```

//建立索引（TODO：db_info_id和table_name做联合索引，没有单独查的场景 ）
db.change_log.createIndex({ db_info_id: 1 });
db.change_log.createIndex({ table_name: 1 });
db.change_log.createIndex({ data_id: 1 });

# 查询

通过输入条件参数和分页参数，构成一个查询请求

	- 分页参数参考：
```json
{
	"page_no" : 1,
	"size" : 20,
	"total" : 100,
	"page_count" : 5
}
```
	- 查询参数参考：
		- 暂无

	- 查询返回的列表数据：
	```json
	{
		"data" : {},
		"list" : [],
		"page" : {
		{
			"page_no" : 1,
			"size" : 20,
			"total" : 100,
			"page_count" : 5
		}
	}
	```
### dao模块设计

- 数据库连接建立后，`gorm.DB` 必须保持为单例。
- 通过 `NewDAO(db)` 创建 DAO，`db` 来自原始 `db` 的克隆。
- gorm插件解决主从同步问题，乐观锁和版本机制都属于这里，用强制的接口 Sync() 来保证原子性
- 提供 Close()，在程序结束时释放数据库连接
### service模块设计

	举例UserService，service的职责是协调dao和cache，如防止穿透查询；解析查询参数，先从cache处获取，如没有则向db获取然后put cache，期间要使用同步锁来保证原子性；另外涉及修改数据，还要记录修改日志
	
- 提供基础接口（注意使用范型）：
	- **单条查询**：`Get(id, options...) [T]`
	- - **创建**：`Create(T, options...)` 新增记录
	- **更新**：`Update(T, options...)` 修改记录，需实现 `Updatable()`
	- **删除**：`Delete(id, options...)` 软删除，对于非 `Updatable()` 的记录，执行物理删除
	- **条件查询**：`List(condition, options...) Resp[T]` 结构参考上面的 查询返回的列表数据
- options机制，在所有的接口中，都预留了 options... 参数，可以扩展很多行为：
	- WithDeleted 查询时包括已软删除的数据
	- WithForceDelete 物理删除数据
	- WithNoCache 不走缓存，直接查库
	- WithPage 分页查询


# 增删改

# 查询

orm结构
field-tag: `query:unique;exp=between;order=asc;`
exp: gt, ge, lt, le, eq, ne, nil, nnil, between([2]slice), in([n]slice)

`QueryParam`，DAO層的查詢結構，内嵌Pager，不要將其他層的request混進來，request的查詢結構應該貼合業務，QueryParam偏向通用設計，甚至以ORM方式驅動，將request轉爲QueryParam是service的邏輯
interface QueryParam{
	Pager() Pager
	Unique() []string
	Order() []string
}
實現QueryParam的好處是確保查詢參數是struct，且具有一些規範參數

`Result`，查詢結果結構，内嵌PagerResp、List<T>分別對應分頁結果，全查結果，單個結果List(0)
Result{
	Pager
	List<T>
}

`Error`
	RecordNotFound，無論列表還是單個，不受具體數據庫約束，接口可決定是否嚴格返回錯誤，用Option形式，
		例如：userDao.Get(1, ...option) => RequiredOption -> if (notFound){ return RecordNotFound }

注意区分API层和DAO层的error，以下是数据平台API层的Error示例，目前先忽略
Error{
	Code: 200
	Msg: "my err msg"	//非必須
	Data: error-ref		//内部Log，不到API層
}
200:		ok，正常，不是錯誤，只是缺省設置
400:		record not found
500:		internal error
600:		外部組件錯誤，如mysql, redis等
700:		外部請求錯誤，通常是第三方API
701:		外部請求無回應
702:		外部請求回應數據解析錯誤