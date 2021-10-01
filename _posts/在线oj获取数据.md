tags:

- 小工具

categories:

- 杂

title: 在线oj获取数据

---



# 在线oj获取数据

python3:

```python
import socket
import sys    

ip = '39.108.164.219'
port = 60000

def send_raw(raw):

    try:
        with socket.create_connection((ip, port), timeout=10) as conn:
            conn.send(bytes(raw,encoding="ascii"))
            conn.close()
    except:
        return False  
    return True

data=""
for line in sys.stdin:
    data+=line
send_raw(data)

```



vps上运行

```shell
#!/bin/bash
i=1
while [ $i -eq 1 ]
do 
	 nc -FNlp 60000  >> /tmp/data
	echo -e  "\n---------------------------------\n" >> /tmp/data
done

```

