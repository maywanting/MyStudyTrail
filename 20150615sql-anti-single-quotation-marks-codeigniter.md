title: "sql语句的反单引号问题--CodeIgniter框架编程"
date: 2015-06-15 15:57:44
tags:
- PHP
- CodeIgniter
- sql
categories:
- experience
---

> ## 问题描述

众所周知，mysql语句编写时，尤其是框架开发，最好加上反单引号，以防止出现保留字符而报错，CI框架也不例外。以下面一段程序为例。

```php
$this->db->select ("id, lamp_num as num, lamp_name as name");
$this->db->from("lamp");
$this->db->where("problem_state_time > lamp_state_setting");
$new_data = $this->db->get()->result();
echo $this->db->last_query();
```

输出的sql语句就是

```sql
SELECT `id`, `lamp_num` as num, `lamp_name` as name
FROM (`sl_lamp`)
WHERE `problem_state_time` > lamp_state_setting
```

也就是说框架本身已经帮你把反单引号加上去了，非常的智能啊
但是呢，问题来了，我突然间想搜索这样的语句

```sql
select id, lamp_num as num, lamp_name as name, 'lamp' as problem_type from sl_lamp where problem_state_time > lamp_state_setting;
```

在mysql终端输入后则是下面的效果

```bash
+-----+------+-----------------+--------------+
| id  | num  | name            | problem_type |
+-----+------+-----------------+--------------+
|  54 | 001  | 人民北路001       | lamp         |
|  55 | 002  | 人民北路002       | lamp         |
|  59 | 006  | 人民北路006       | lamp         |
| 128 | 1004 | sklcc1004       | lamp         |
+-----+------+-----------------+--------------+
4 rows in set (0.01 sec)
```

若在框架里面写，一开始我是这么写的

```php
$this->db->select ("id, lamp_num as num, lamp_name as name， 'lamp' as problem_type");
$this->db->from("lamp");
$this->db->where("problem_state_time > lamp_state_setting");
$new_data = $this->db->get()->result();
echo $this->db->last_query();
```

结果报错了，报错信息为`Error Number: 1054; Unknown column ''lamp'' in 'field list'`
输出的sql语句则是

```sql
SELECT `id`, `lamp_num` as num, `lamp_name` as name, `'lamp'` as problem_type FROM (`sl_lamp`) WHERE `problem_state_time` > lamp_state_setting;
```

> ## 解决方法

也就是说如果反单引号扩起来的话，就会直接默认里面的内容作为属性名，这明显不是我想要的啊。后来查了[官方手册](https://ellislab.com/codeigniter/user-guide/database/active_record.html#select)，明白了原来select函数总共有两个参数，前一个不用多说就是查找内容，后一个则是关于反单引号的开关，默认是打开，关闭的话只要再添一个参数`FALSE`即可。

```php
$this->db->select ("`id`, `lamp_num` as num, `lamp_name`as name， 'lamp' as problem_type", FALSE);
$this->db->from("lamp");
$this->db->where("problem_state_time > lamp_state_setting");
$new_data = $this->db->get()->result();
echo $this->db->last_query();
```

则sql语句为

```sql
SELECT `id`, `lamp_num` as num, `lamp_name`as name, 'lamp' as problem_type
FROM (`sl_lamp`)
WHERE `problem_state_time` > lamp_state_setting;
```

这样就对了嘛～

The End~
