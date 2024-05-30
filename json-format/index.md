# 将Mysql查询内容转换为JSON


# 安装命令
```bash
pip install csvkit mycli
```

# 打印JSON格式

## 在Mycli终端中打印

```bash
 $ mycli -P 3306 -proot mysql
MySQL root@localhost:mysql> \T csv
MySQL root@localhost:mysql> pager csvjson | jq '.'

```

## 在Linux命令行打印

```bash
mycli -P 3306 -proot mysql --csv -e "select * from user;" | csvjson | jq .

```

# 参考连接
https://github.com/dbcli/mycli/issues/753#issuecomment-504279817
