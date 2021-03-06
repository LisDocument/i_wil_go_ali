# MySQL的自增主键

## 1. 自增值的保存位置

表的结构定义存放在后缀为.frm的文件中，但是并不会保存自增值。

不同引擎对自增值的保存策略不同。

- MyISAM引擎的自增值保存在数据文件中
- InnoDB引擎的自增值，保存在内存里，并且在MySQL8.0版本以后，才有了“自增值持久化”的能力，也就是才实现“如果发生重启，表的自增值可以恢复MySQL重启前的值”
  - MySQL5.7及以前的版本，自增值保存在内存里，并没有持久化，每次重启后，第一次打开表都会去找自增值的最大值max(id)，然后这个值+1作为这个表的当前自增值，即MySQL重启可能会修改一个表的AUTO_INCREMENT的值
  - MySQL8.0版本，将自增值的变更记录在了redo log中，重启的时候依靠redolog恢复重启之前的值。

## 2. 自增值的修改机制

如果id被定义为AUTO_INCREMENT，在插入一行数据的时候，自增值的行为如下

1. 如果插入数据的时候id字段指定为0，null，未指定值，那么就这个表当前的AUTO_INCREMENT值填到自增字段；
2. 如果插入数据时id字段制定了具体的值，那么就使用这个值
   1. 如果这个值大于当前AUTO_INCREMENT，那么 AUTO_INCREMENT会修改为新的自增值，从初始值**auto_increment_offset**开始，以**auto_increment_increment**为步长，直到找到第一个大于这个值的值，作为新的自增值。
   2. 如果小于AUTO_INCREMENT，没有变化

> 有一些场景下，可能会不使用默认值，双M的主备结构，要求双写的时候，可以设置步长和初始值，使两台数据库一个使用单数的自增值，一个使用双数的自增值，保证两个库生成的主键不会冲突。

## 3. 自增值的修改时机

出现冲突回滚的时候，主键自增值派发的不会进行回滚。

可以从这里考虑，如果自增值可以回滚的话，会出现一系列的情况 

- 如果允许自增值回滚，那么同时两个事务同时执行的时候，第一个事务申请到了id=2， 第二个事务申请到了id=3，那么第一个事务发生了回滚，自增值回滚，会导致自增值变为2，但是此时第二个事务已经入库，第三个事务申请自增值获取id=2+1，与第二个事务直接冲突。
- 为了解决上面出现的冲突，那么需要先去判断表里是否存在这个id，这个成本太高了。或者将自增id的锁扩大，那么会导致系统并发能力大大降低。

因此InnoDB放弃了这个设置，语句执行失败也不回退自增id。因此自增id只能保证递增，但不保证连续。

## 4. 自增锁的优化

从3已经可以看到，自增锁不是一个事务锁，每次申请完就会释放，以便允许其他事务申请。这个功能在MySQL5.1版本之后才有效。

5.1.22引入了一个新的策略，新增了参数**innodb_autoinc_lock_mode**。默认是1

- 0，采用MySQL5.0的策略，语句执行结束时放锁
- 1，普通insert，申请完立刻释放；insert...select...这样的语句，等语句结束后才释放
- 2， 所有申请自增都是申请后释放

策略1对insert...select语句加了自增锁，是对于binlog格式设置为statement的情况，binlog这种格式下是以session记的，即sessionA执行多条插入语句的时候，session B执行了一句，如果策略是2的时候，可能sessionA的语句执行完生成的id是2，3，4，6，sessionB的语句是5，但是statememt写入的是sessionA执行完然后执行sessionB，那么就可能存在sessionA在备库执行获得的id是2，3，4，5，sessionB获得的是6，会发生事务不一致。

**当然推荐binlog的格式是row**，如果是row的情况下，就不用考虑，直接使用策略2。

批量插入数据的语句，MySQL是不知道预先要申请多少个id的，这也是策略1批量更新加锁的原因之一，insert values这种语句除外，这个语句可以知道具体是多少个数据要插入的。

MySQL有一个批量申请自增id的策略，对于上面说的这种批量插入数据的语句，第一次申请的时候会分配一个自增id，一个用完申请分配两个，两个用完申请会有4个，依此类推。因此可能会获取到多个自增id没有用完，但是不会返回，因此这里也有可能出现主键不连续。

## 5. 自增值用完了怎么办

如果自增值已经到了最大，那么下一行数据插入后不会改变表的自增值，再次插入的时候会出现重复key的情况。

如果创建的InnoDB表没有设置主键，引擎会自动创建一个不可见，长度为6个字节的row_id，InnoDB维护了一个全局的dict_sys.row_id作为要插入数据的row_id，然后把dict_sys.row_id+1。虽然字节长度为6，但是在实现上其实row_id是一个无符号长整形（8个字节）。因为InnoDB在设计的时候给row_id留了6个字节的长度，因此写回的时候其实会丢失精度。

1. row_id写入表的值范围是是从 0 到 2^48-1
2. 当 dict_sys.row_id=2^48时，如果再有插入数据的行为要来申请 row_id，拿到以后再取最后 6 个字节的话就是 0。

在InnoDB逻辑里，申请到row_id=N后，就将这条数据写入数据库，如果数据库已经存在这条row_id=N，那么就更新这条记录。

