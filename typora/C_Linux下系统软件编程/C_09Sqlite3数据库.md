# 数据库

[TOC]

## 1.数据存储的方式

- 变量
- 数组/队列/栈
- 链表
- 文件
- 数据库
  - RAM ->掉电/程序运行结束丢失
  - ROM ->掉电/程序运行结束不丢失

## 2.不同数据的特点`二维表格/键值对`

- 关系型数据库：将复杂的数据结构简化为二维表格形式
  - 大型:`Oracle、DB2`
  - 中型:`MySql、SQLServer`		
  - 小型:`Sqlite`

- 非关系:以键值对存储，且结构不固定
  - `JSON`
  - `Redis`
  - `MongoDB`

## 3.`sqlite3`数据库

### 1.为什么用`sqlite3`?`开源、轻量级、文件型、可移植、c语言开发`

- 开源免费、`c语言`开发
- 轻量级，代码量少约1万行，总大小`10MB`以内
- 有`sqlitebrowser`可视化工具，便于管理
- 文件型数据库，可以移动，跨平台移植性好

### 2.`sqlite3`相关命令（`sqlite3`  【数据库名.db】）

- .help 				查看数据库相关命令

- .tables                              查看数据库已有的表

- .headers on/off               打开/关闭标头

- .mode column                设置左对齐

- .quit                                  退出

**注意：命令前加点且后面无分号**

安装：`**sudo apt-get install sqlitebrowser`**
启动：`**sqlitebrowser  【数据库名.db】 打开linux下的可视化工具`**

### 2.`sqlite3-SQL`语句

1. #### `SQL`数据类型：`空/interger整形/real浮点型/text字符型`

   - NULL空单元格，

   - integer整形，

   - real浮点型，

   - text字符型，

   - blob根据输入确定

     > [!IMPORTANT]
     >
     > - `SQL`语句对大小写不敏感，大小写均可
     >
     > - `SQL`语句后有分号
     > - 数据库命令 `.tables`不加；

2. #### `SQLite3-API`表的基础操作（创建，增删改查）

   ```c
   1. 创建
      create table 【表名】（【列名】数据类型 ，【列名】数据类型，【列名】数据类型）；
   2. 插入
      insert into【表名】values（值1，值2，值3...）；
   3. 删除
      a.删除数据： delete from 【表名】 条件 ；
   			delete from  【表名】where 【列名】= xxx；
      b.删除表：	  delete table 【表名】；
   4. 修改
      update 【表名】 set 【列名】= new  where【列名】 =old ；
   5. 查找
      1. 查看表中所有数据： select * from 【表名】；*表示所有数据,
      2. 查看指定列：select 【列1】，【列2】 from 【表名】;
      3. 条件查找：select * from【表名】 where【列名】<关系运算符>xxx;
      										"= > < ! and(&&) or(||)"
      4.模糊查找：select *from 【表名】where【列名】 like"%xxx"/"_xxx"；
   										"%通配多个字符/_通配一个字符"
      5.有序查找（ASC升序/DESC降序）：select *from 【表名】 order by asc/desc;
   *. '主动键值增长列'：
   	必须是INTEGER整型， ID integer primary key autoincrement
   	 insert into 【表名】values（NULL,XXX）；
   *. 时间相关函数
   	datetime（"now","+8 hours"）;
   
   '语句一定以分号结尾！！！！！！！！
    事务机制？插入大量数据-降低时间开销
   ```

### 3.`sqlite3-C`函数接口

 [SQLite参考手册.CHM](..\Y_辅助文档\SQLite参考手册.CHM)  

[SQL入门经典(第五版)中文版.pdf](..\Y_辅助文档\SQL入门经典(第五版)中文版.pdf) 

```c
核心接口就3个
打开/创建数据库	
	sqlite3_open(const char *filename, sqlite3 **ppDb)
执行SQL语句	
	sqlite3_exec(sqlite3*, const char *sql, sqlite_callback, void *data, char **errmsg)
关闭数据库
	sqlite3_close(sqlite3*)
错误处理
	sqlite3_errmsg(sqlite3*)
```

