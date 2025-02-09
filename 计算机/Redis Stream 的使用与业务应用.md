# Redis Stream 的使用与业务应用



## 基本使用

### XADD 队列添加消息

~~~ shell
XADD key ID field value [field value ...]
~~~

- **key** ：队列名称，如果不存在就创建
- **ID** ：消息 id，我们使用 * 表示由 redis 生成，可以自定义，但是要自己保证递增性。
- **field value** ： 记录。

~~~shell
# 向队列当中推送一条稳态建模消息
XADD javaToPython * modelId 25 type steady path f:/data/csv/build_test.csv

# 返回一个 redis 自动生成的消息 id
"1734486699344-0"

# 再次推送一条瞬态建模的消息
XADD javaToPython * modelId 26 type transient path f:/data/csv/build_test02.csv pointNames AMD10001.AD

#返回一条消息 id
"1734486930841-0"
~~~



### XLEN 获取队列当中存在的消息长度

~~~ shell
XLEN key
~~~

- **key**：队列名称

~~~ shell
# 获取当前现存的消息长度
XLEN javaToPython

# 现有之前推送的两条消息
(integer) 2
~~~



### XDEL 删除队列当中的消息

```
XDEL key ID [ID ...]
```

- **key**：队列名称
- **ID** ：消息 ID

~~~ shell
#向队列当中再次添加一条新的消息
XADD javaToPython * modelId 27 type steady path f:/data/csv/build_test03.csv
"1734487532701-0"

#查看当前队列的长度
XLEN javaToPython
(integer) 3

#删除消息成功返回 1
XDEL javaToPython 1734487532701-0
(integer) 1

#再次查看当前队列的长度
XLEN javaToPython
(integer) 2
~~~

![image-20241218100903926](./assets/image-20241218100903926.png) 

![image-20241218100831546](./assets/image-20241218100831546.png) 



### XRANGE 顺序获取队列消息

```shell
#该命令会自动的过滤到被删除的消息
XRANGE key start end [COUNT count]
```

- **key** ：队列名
- **start** ：开始值， **-** 表示最小值
- **end** ：结束值， **+** 表示最大值
- **count** ：数量

~~~ shell
# 从队列当中从降序获取两条消息 
XRANGE javaToPython - + COUNT 2 

#当前队列当中存在的消息
1) 1) "1734486699344-0"
   2) 1) "modelId"
      2) "25"
      3) "type"
      4) "steady"
      5) "path"
      6) "f:/data/csv/build_test.csv"
2) 1) "1734486930841-0"
   2) 1) "modelId"
      2) "26"
      3) "type"
      4) "transient"
      5) "path"
      6) "f:/data/csv/build_test02.csv"
      7) "pointNames"
      8) "AMD10001.AD"
~~~



### XREVRANGE 逆序获取队列消息

```shell
#该命令会自动的过滤到被删除的消息
XREVRANGE key end start [COUNT count]
```

- **key** ：队列名
- **end** ：结束值， **+** 表示最大值
- **start** ：开始值， **-** 表示最小值
- **count** ：数量

~~~ shell
# 从队列当中从升序获取两条消息 
XRANGE javaToPython - + COUNT 2

#当前队列当中存在的消息
1) 1) "1734486930841-0"
   2) 1) "modelId"
      2) "26"
      3) "type"
      4) "transient"
      5) "path"
      6) "f:/data/csv/build_test02.csv"
      7) "pointNames"
      8) "AMD10001.AD"
2) 1) "1734486699344-0"
   2) 1) "modelId"
      2) "25"
      3) "type"
      4) "steady"
      5) "path"
      6) "f:/data/csv/build_test.csv"
~~~



### XREAD 非阻塞方式获取消息队列

```
XREAD [COUNT count] [BLOCK milliseconds] STREAMS key [key ...] id [id ...]
```

- **count** ：数量
- **milliseconds** ：可选，阻塞毫秒数，没有设置就是非阻塞模式
- **key** ：队列名
- **id** ：消息 ID

~~~ shell
# 向添加两组消息分别到队列 javaToPython_one 与 javaToPython_two 当中
XADD javaToPython_one * modelId 25 name steady_model_one path f:/data/csv/build_test_001.csv
"1734488861940-0"

XADD javaToPython_one * modelId 26 name steady_model_two path f:/data/csv/build_test_002.csv
"1734488866997-0"

XADD javaToPython_two * modelId 27 name transient_model_one path f:/data/csv/build_test.csv
"1734488872164-0"

XADD javaToPython_two * modelId 28 name transient_model_two path f:/data/csv/build_test.csv
"1734488876470-0"

# 从 Stream 头部读取两条消息 这里的 0-0 表示的意思是不指定 id，不指定 id 默认就用 0 或者 0-0 表示
XREAD COUNT 2 STREAMS javaToPython_one javaToPython_two 0-0 0-0 

1) 1) "javaToPython_one"
   2) 1) 1) "1734488861940-0"
         2) 1) "modelId"
            2) "25"
            3) "name"
            4) "steady_model_one"
            5) "path"
            6) "f:/data/csv/build_test_001.csv"
      2) 1) "1734488866997-0"
         2) 1) "modelId"
            2) "26"
            3) "name"
            4) "steady_model_two"
            5) "path"
            6) "f:/data/csv/build_test_002.csv"
2) 1) "javaToPython_two"
   2) 1) 1) "1734488872164-0"
         2) 1) "modelId"
            2) "27"
            3) "name"
            4) "transient_model_one"
            5) "path"
            6) "f:/data/csv/build_test.csv"
      2) 1) "1734488876470-0"
         2) 1) "modelId"
            2) "28"
            3) "name"
            4) "transient_model_two"
            5) "path"
            6) "f:/data/csv/build_test.csv"
~~~



### XGROUP CREATE 创建一个消费群组

```shell
XGROUP [CREATE key groupname id-or-$] [SETID key groupname id-or-$] [DESTROY key groupname] [DELCONSUMER key groupname consumername]
```

- **key** ：队列名称，如果不存在就创建
- **groupname** ：组名。
- **$** ： 表示从尾部开始消费，只接受新消息，当前 Stream 消息会全部忽略。

#### 从头开始消费:

```shell
#创建一个python 端从头开始的消费组
XGROUP CREATE javaToPython_one java-to-pytohn-group 0-0  
```

#### 从尾部开始消费:

```shell
#创建一个python 端从尾开始的消费组
XGROUP CREATE javaToPython_two java-to-pytohn-group $
```



### XREADGROUP GROUP 读取消费组的消息

```shell
XREADGROUP GROUP group consumer [COUNT count] [BLOCK milliseconds] [NOACK] STREAMS key [key ...] ID [ID ...]
```

- **group** ：消费组名
- **consumer** ：消费者名。
- **count** ： 读取数量。
- **milliseconds** ： 阻塞毫秒数。
- **key** ： 队列名。
- **ID** ： 消息 ID。

```shell
# 查看 javaToPython_one 队列的当前长度
XLEN javaToPython_one

# 当前队列当中有两条消息
(integer) 2

# 查看一下当前队列当中的消息
XRANGE javaToPython_one - + COUNT 2
# 当前队列当中存在的消息
1) 1) "1734488861940-0"
   2) 1) "modelId"
      2) "25"
      3) "name"
      4) "steady_model_one"
      5) "path"
      6) "f:/data/csv/build_test_001.csv"
2) 1) "1734488866997-0"
   2) 1) "modelId"
      2) "26"
      3) "name"
      4) "steady_model_two"
      5) "path"
      6) "f:/data/csv/build_test_002.csv"


# 从 python 端的消费组 java-to-pytohn-group 开始进行消费，java 端发向队列 javaToPython_one 推送的消息
XREADGROUP GROUP java-to-pytohn-group python-one-consumer COUNT 1 STREAMS javaToPython_one >
# 消费了一条消息
1) 1) "javaToPython_one"
   2) 1) 1) "1734488861940-0"
         2) 1) "modelId"
            2) "25"
            3) "name"
            4) "steady_model_one"
            5) "path"
            6) "f:/data/csv/build_test_001.csv"        
# 再次消费一条消息
1) 1) "javaToPython_one"
   2) 1) 1) "1734488866997-0"
         2) 1) "modelId"
            2) "26"
            3) "name"
            4) "steady_model_two"
            5) "path"
            6) "f:/data/csv/build_test_002.csv"
```

![image-20241218104120619](./assets/image-20241218104120619.png) 



### XACK 确认消息消费

~~~ shell
XACK <stream> <group> <ID>
~~~

- **stream**: 队列名
- **group**: 消费者组的名称
- **ID**: 消息的唯一 id

~~~ shell
# 检查当前的消费状态
XPENDING javaToPython_one java-to-pytohn-group 

# 可以看到java-to-pytohn-group 组下的 javaToPython_one 队列有两条消息还未被消费
1) "2" 					 	  
2) "1734488861940-0" 			
3) "1734488866997-0" 			
4) 1) 1) "python-one-consumer"  
      2) "2"

# python 端 java-to-pytohn-group 组下的 javaToPython_one 消费者确认 id 为 1734488866997-0 的消息被消费
XACK javaToPython_one java-to-pytohn-group  1734488866997-0
#确认成功返回 1
(integer) 1

# 可以看到 id 为 1734488866997-0 的消息已被消费，目前只剩一条消息
XPENDING javaToPython_one java-to-pytohn-group 
1) "1"
2) "1734488861940-0"
3) "1734488861940-0"
4) 1) 1) "python-one-consumer"
      2) "1"
~~~





### 检查消息是否消费

```
XPENDING <stream> <group>
```

- **stream**: 队列名
- **group**: 消费者组的名称

~~~ shell
# 检查 java-to-pytohn-group 组是否将 javaToPython_one 队列当中的消息消费
XPENDING javaToPython java-python-service

# 返回结果
1) "2" 					 	   # 未确认消息数量 2 表示有两条消息未确认
2) "1734488861940-0" 			# 消息 ID 范围 1734488861940-0 到 1734488866997-0 这个时间端内未确认的消息
3) "1734488866997-0" 			# 消息 ID 范围 1734488861940-0 到 1734488866997-0 这个时间端内未确认的消息
4) 1) 1) "python-one-consumer"   # 表示 python-one-consumer 消费者有两条消息未被消费
      2) "2"
~~~





