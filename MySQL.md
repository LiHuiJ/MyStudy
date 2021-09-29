# mysql时间格式化

**DATE_FORMAT(date,format) ** 函数用于以不同的格式显示日期/时间数据。 date是日期列,format是格式
**STR_TO_DATE(str,format)**   将字符串转成日期

```sql
SELECT STR_TO_DATE('2018-06-01','%Y-%m-%d');
SELECT DATE_FORMAT(NOW(),'%Y-%m-%d');
```

## 格式化的各种格式：

```sql
%W 星期名字(Sunday……Saturday)  
%D 有英语前缀的月份的日期(1st, 2nd, 3rd, 等等。）  
%Y 年, 数字, 4 位  
%y 年, 数字, 2 位  
%a 缩写的星期名字(Sun……Sat)  
%d 月份中的天数, 数字(00……31)  
%e 月份中的天数, 数字(0……31)  
%m 月, 数字(01……12)  
%c 月, 数字(1……12)  
%b 缩写的月份名字(Jan……Dec)  
%j 一年中的天数(001……366)  
%H 小时(00……23)  
%k 小时(0……23)  
%h 小时(01……12)  
%I 小时(01……12)  
%l 小时(1……12)  
%i 分钟, 数字(00……59)  
%r 时间,12 小时(hh:mm:ss [AP]M)  
%T 时间,24 小时(hh:mm:ss)  
%S 秒(00……59)  
%s 秒(00……59)  
%p AM或PM  
%w 一个星期中的天数(0=Sunday ……6=Saturday ）  
%U 星期(0……52), 这里星期天是星期的第一天  
%u 星期(0……52), 这里星期一是星期的第一天  
%% 一个文字“%”
```



# mysql自动创建表

## 1、创建函数

```sql
CREATE PROCEDURE create_table()
BEGIN
declare str_date varchar(16);
SET str_date = date_format(now(),"%Y%i");
SET @sqlcmd1 = CONCAT('CREATE TABLE t_org_',str_date,"(
  `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT '主键ID',
  `org_code` varchar(11) NOT NULL COMMENT '信用代码',
  `org_name` varchar(64) NOT NULL COMMENT '名称',
  `create_time` datetime NOT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=11 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;");
PREPARE p1 FROM @sqlcmd1;
EXECUTE p1;
DEALLOCATE PREPARE p1;
END
```

## 2、创建事件

```sql
-- 每分钟执行
CREATE EVENT IF NOT EXISTS e_create
ON SCHEDULE EVERY 1 MINUTE
ON COMPLETION PRESERVE
DO CALL create_table();
-- '设计计划'中可修改计划执行
-- 查看计划
SHOW EVENTS;
```

## 3、开启与关闭事件

```sql
-- 开启
alter event e_create ON COMPLETION PRESERVE ENABLE;

-- 关闭
alter event e_create ON COMPLETION PRESERVE DISABLE;
```

