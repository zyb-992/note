# API

1. **LIMIT OFFSET**

   ```mysql
   # 跳过y条数据 读取x条数据
   select * from table_name limit x offset y;
   ```





# Gorm

1.

```mysql
# 查找
# First/Last:只有在目标struct是指针或者通过 db.Model() 指定 model 时，该方法才有效。
First
Last
Take
Find 
Scan
```

