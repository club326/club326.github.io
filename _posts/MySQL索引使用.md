# MySQL的索引使用
## 1.MySQL索引原理
## 2.explain含义解析
id: 1//SELECT标识符
  select_type: SIMPLE//查询类型
        table: room_title//表名
         type: range
possible_keys: idx_cs_fc_ctime
          key: idx_cs_fc_ctime
      key_len: 5//这个长度说明联合索引实际使用的长度（单位字节）
          ref: NULL
         rows: 1
        Extra: Using index condition; Using where; Using filesort

## 3.索引优化
* 单列索引效率不高时应使用联合索引（很多情况下联合索引效率更高）
* 创建联合索引时应将等值查询放在前面（范围查询列放在后面）
* 多个sql使用的索引大致相同时，有时没必要设创建多个索引，可以共用一个索引
 idx_status_uid_time(`check_status`,`checking_uid`,`create_time`)，idx_cs_ctime_sou(check_status,create_time,source_type)，idx_cs_fc_ctime(check_status,fc_finish_time,create_time)
 这三个索引在使用过程中会发现都使用了（check_status,create_time）这两个列，其实在增加一列性能无明显提升时，可以考虑将这3个索引合并成只创建一个索引idx_cstatus_ctime(check_status,create_time),这样兼顾了读写性能；
## 4.注意事项
