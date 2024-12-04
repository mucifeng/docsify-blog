|  名称   | 命令表达式  |
|  ----  | ----  |
| 创建表 | create '表名', '列族名1','列族名2','列族名N'| 
| 查看所有表 | list| 
| 描述表 | describe ‘表名’| 
| 判断表存在 | exists  '表名'| 
| 判断是否禁用启用表|  is_enabled '表名'is_disabled ‘表名’| 
| 添加记录|  put ‘表名’, ‘rowKey’, ‘列族 : 列‘ , '值'| 
| 查看记录rowkey下的所有数据|  get '表名' , 'rowKey'| 
| 查看表中的记录总数|  count '表名'| 
| 获取某个列族 | get '表名','rowkey','列族'| 
| 获取某个列族的某个列|  get '表名','rowkey','列族：列’| 
| 删除记录 | delete ‘表名’ ,‘行名’ , ‘列族：列'| 
| 删除整行|  deleteall '表名','rowkey'| 
| 删除一张表|  先要屏蔽该表 才能对该表进行删除第一步 disable ‘表名’ ，第二步 drop '表名'| 
| 清空表 | truncate '表名'| 
| 查看所有记录|  scan "表名"| 
| 查看某个表某个列中所有数据 | scan "表名" , {COLUMNS=>'列族名:列名'}| 
| 更新记录| hbase没有修改，都是追加|


### 操作
```
进入hbase命令行  
./hbase shell

显示hbase中的表  
list

创建user表，包含info、data两个列族
create 'user', 'info1', 'data1'
create 'user', {NAME => 'info', VERSIONS => '3'}

向user表中插入信息，row key为rk0001，列族info中添加name列标示符，值为zhangsan
put 'user', 'rk0001', 'info:name', 'zhangsan'

向user表中插入信息，row key为rk0001，列族info中添加gender列标示符，值为female
put 'user', 'rk0001', 'info:gender', 'female'

向user表中插入信息，row key为rk0001，列族info中添加age列标示符，值为20
put 'user', 'rk0001', 'info:age', 20

向user表中插入信息，row key为rk0001，列族data中添加pic列标示符，值为picture
put 'user', 'rk0001', 'data:pic', 'picture'

获取user表中row key为rk0001的所有信息
get 'user', 'rk0001'

获取user表中row key为rk0001，info列族的所有信息
get 'user', 'rk0001', 'info'

获取user表中row key为rk0001，info列族的name、age列标示符的信息
get 'user', 'rk0001', 'info:name', 'info:age'

获取user表中row key为rk0001，info、data列族的信息
get 'user', 'rk0001', 'info', 'data'
get 'user', 'rk0001', {COLUMN => ['info', 'data']}

get 'user', 'rk0001', {COLUMN => ['info:name', 'data:pic']}

获取user表中row key为rk0001，列族为info，版本号最新5个的信息
get 'user', 'rk0001', {COLUMN => 'info', VERSIONS => 2}
get 'user', 'rk0001', {COLUMN => 'info:name', VERSIONS => 5}
get 'user', 'rk0001', {COLUMN => 'info:name', VERSIONS => 5, TIMERANGE => [1392368783980, 1392380169184]}

获取user表中row key为rk0001，cell的值为zhangsan的信息
get 'people', 'rk0001', {FILTER => "ValueFilter(=, 'binary:图片')"}

获取user表中row key为rk0001，列标示符中含有a的信息
get 'people', 'rk0001', {FILTER => "(QualifierFilter(=,'substring:a'))"}

put 'user', 'rk0002', 'info:name', 'fanbingbing'
put 'user', 'rk0002', 'info:gender', 'female'
put 'user', 'rk0002', 'info:nationality', '中国'
get 'user', 'rk0002', {FILTER => "ValueFilter(=, 'binary:中国')"}

查询user表中的所有信息
scan 'user'

查询user表中列族为info的信息
scan 'user', {COLUMNS => 'info'}
scan 'user', {COLUMNS => 'info', RAW => true, VERSIONS => 5}
scan 'persion', {COLUMNS => 'info', RAW => true, VERSIONS => 3}
查询user表中列族为info和data的信息
scan 'user', {COLUMNS => ['info', 'data']}
scan 'user', {COLUMNS => ['info:name', 'data:pic']}


查询user表中列族为info、列标示符为name的信息
scan 'user', {COLUMNS => 'info:name'}

查询user表中列族为info、列标示符为name的信息,并且版本最新的5个
scan 'user', {COLUMNS => 'info:name', VERSIONS => 5}

查询user表中列族为info和data且列标示符中含有a字符的信息
scan 'user', {COLUMNS => ['info', 'data'], FILTER => "(QualifierFilter(=,'substring:a'))"}

查询user表中列族为info，rk范围是[rk0001, rk0003)的数据
scan 'people', {COLUMNS => 'info', STARTROW => 'rk0001', ENDROW => 'rk0003'}

查询user表中row key以rk字符开头的
scan 'user',{FILTER=>"PrefixFilter('rk')"}

查询user表中指定范围的数据
scan 'user', {TIMERANGE => [1392368783980, 1392380169184]}

删除数据
删除user表row key为rk0001，列标示符为info:name的数据
delete 'people', 'rk0001', 'info:name'
删除user表row key为rk0001，列标示符为info:name，timestamp为1392383705316的数据
delete 'user', 'rk0001', 'info:name', 1392383705316

清空user表中的数据
truncate 'people'

修改表结构
首先停用user表（新版本不用）
disable 'user'

添加两个列族f1和f2
alter 'people', NAME => 'f1'
alter 'user', NAME => 'f2'
启用表
enable 'user'


disable 'user'(新版本不用)
删除一个列族：
alter 'user', NAME => 'f1', METHOD => 'delete' 或 alter 'user', 'delete' => 'f1'

添加列族f1同时删除列族f2
alter 'user', {NAME => 'f1'}, {NAME => 'f2', METHOD => 'delete'}

将user表的f1列族版本号改为5
alter 'people', NAME => 'info', VERSIONS => 5
启用表
enable 'user'

删除表
disable 'user'
drop 'user'
```