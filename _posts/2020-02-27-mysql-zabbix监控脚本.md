## mysql-zabbix监控脚本

- check_mysql.py，主从延迟也是这个脚本。
``` bash
#!/usr/bin/env python
# encoding: utf-8



import pymysql
import sys


# 获取连接
g_port = 0
def get_conn():
conn = None
try:
conn = pymysql.connect(
host="127.0.0.1",
port=int(g_port),
user="zabbix",

#需要修改密码
passwd="xxxxxxx",
charset="utf8",
)
except Exception as err:
print(err)
return conn


# 查询语句执行
def get_data(sql):
conn = get_conn()
cur = conn.cursor()
cur.execute(sql)
data = cur.fetchall()
conn.close()
return data


def get_value_by_varname(var_name):

sql = '''
show global status where variable_name='%s';
'''%(var_name)
data = get_data(sql)
if(len(data)==1) :
return data[0][1]
else:
return 0

def get_value_by_slave_Seconds_Behind_Master():
sql = '''
show slave status
'''
data = get_data(sql)
if (len(data) == 1):
return data[0][32]
else:
return 0


if __name__ == '__main__':
g_port = sys.argv[2]
if(sys.argv[1]=='Seconds_Behind_Master') :
valu =get_value_by_slave_Seconds_Behind_Master()
else:
valu = get_value_by_varname(sys.argv[1])

print(valu)
