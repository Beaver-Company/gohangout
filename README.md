## RUN

gohangout --config config.yml

## 一个简单的配置

```
inputs:
    - Kafka:
        topic:
            weblog: 1
        codec: json
        consumer_settings:
            bootstrap.servers: "10.0.0.100:9092"
            group.id: gohangout.weblog
filters:
    - Grok:
        src: message
        match:
            - '^(?P<logtime>\S+) (?P<name>\w+) (?P<status>\d+)$'
            - '^(?P<logtime>\S+) (?P<status>\d+) (?P<loglevel>\w+)$'
        remove_fields: ['message']
    - Date:
        location: 'Asia/Shanghai'
        src: logtime
        formats:
            - 'RFC3339'
        remove_fields: ["logtime"]
outputs:
    - Elasticsearch:
        hosts:
            - http://127.0.0.1:9200
        index: 'web-%{+2006-01-02}' #golang里面的渲染方式就是用数字, 而不是用YYMM.
        index_type: "logs"
        bulk_actions: 5000
        bulk_size: 20
        flush_interval: 60
```

## 字段格式约定

以 Add Filter 举例

```
fields:
    type: 'weblog'
    hostname: '[host]'
    name: '{{.firstname}}.{{.lastname}}'
    city: '[geo][cityname]'
    '[a][b]': '[stored][message]'
```

### 格式1 [XX][YY]

`city: '[geo][cityname]'` 是把 geo.cityname 的值赋值给 city 字段. 必须严格 [XX][YY] 格式, 前后不能有别的内容


### 格式2 {{XXX}}

如果含有 `{{XXX}}` 的内容, 就认为是 golang template 格式, 具体语法可以参考 [https://golang.org/pkg/text/template/](https://golang.org/pkg/text/template/). 前后及中间可以含有别的内容, 像 `name: 'my name is {{.firstname}}.{{.lastname}}'`

### 格式3 除了1,2 之外的其它

在不同Filter中, 可能意义不同. 像 Date 中的 src: logtime, 是说取 logtime 字段的值.  
Elasticsearch 中的 index_type: logs , 这里的 logs 不是指字段名, 就是字面值.


## INPUT

### 

## OUTPUT

## FILTER

### 通用字段

#### if

#### add_fields

例:

```
Grok:
    src: message
    match:
        - '^(?P<logtime>\S+) (?P<name>\w+) (?P<status>\d+)$'
        - '^(?P<logtime>\S+) (?P<status>\d+) (?P<loglevel>\w+)$'
    remove_fields: ['message']
    add_fields:
      grok_result: 'ok'
```

当Filter执行成功时, 可以添加一些字段. 如果Filter失败, 则忽略. 下面具体的Filter说明中, 提到的"返回false", 就是指Filter失败

#### remove_fields

例子如上. 当Filter执行成功时, 可以删除一些字段. 如果Filter失败, 则忽略.

#### failTag

当Filter执行失败时, 可以添加内容到 `tags` 字段. 如果Filter成功, 则忽略. 如果 tags 字段已经存在, 则将 tags 设置为数组并添加新的数据.

```
Grok:
    src: message
    match:
        - '^(?P<logtime>\S+) (?P<name>\w+) (?P<status>\d+)$'
        - '^(?P<logtime>\S+) (?P<status>\d+) (?P<loglevel>\w+)$'
    remove_fields: ['message']
    add_fields:
      grok_result: 'ok'
    failTag: grokfail

#### overwrite

配置的新字段要不要覆盖之前已有的字段, 默认 true


### Add

```
Add:
  overwrite: true
  fields:
      name: childe
      hostname: '[host]'
      logtime: '{{.date}} {{.{time}}
      message: '[stored][message]'
      '[a][b]': '[stored][message]'
```

1. 增加 name 字段, 内容是 childe
2. 增加 hostname 字段, 内容是原 host 字段中的内容. (相当于改名)
3. 增加 logtime 字段, 内容是 date 和 time 两个字段的拼接
4. 增加 message 字段, 是 event.stored.message 中的内容
4. 将 event.stored.message 中的内容写入 event.a.b 字段中(如果没有则创建)

overwrite: true 的情况下, 这些新字段会覆盖老字段(如果有的话).

### Convert

```
- Convert:
    remove_if_fail: false
    setto_if_fail: 0
    fields:
        time_taken:
            to: float
        sc_bytes:
            to: int

- Convert:
    remove_if_fail: false
    setto_if_fail: true
    fields:
        status:
            to: bool
```

#### remove_if_fail

如果转换失败刚删除这个字段, 默认 false

#### setto_if_fail: XX

如果转换失败, 刚将此字段的值设置为 XX . 优先级比 remove_if_fail 低.  如果 remove_if_fail 设置为 true, 则setto_if_fail 无效.

### Date

```
Date:
    src: 'logtime'
    target: '@timestamp'
    location: Asia/Shanghai
    add_year: false
    overwrite: true
    formats:
        - 'RFC3339'
        - '2006-01-02T15:04:05'
        - '2006-01-02T15:04:05Z07:00'
        - '2006-01-02T15:04:05Z0700'
        - '2006-01-02'
        - 'UNIX'
        - 'UNIX_MS'
    remove_fields: ["logtime"]
```

如果源字段不存在, 返回 false. 如果所有 formats 都匹配失败, 返回 false

#### src

源字段, 必须配置.

#### target

目标字段, 默认 `@timestamp`

#### overwrite

默认 true, 如果目标字段已经存在, 会覆盖

#### add_year

有些日志中的时间戳不带年份信息, 默认 false . add_year: true 可以先在源字段最前面加四位数的年份信息然后再解析.

#### formats

必须设置. 格式参考 [https://golang.org/pkg/time/](https://golang.org/pkg/time/)

除此外, 还有 UNIX UNIX_MS 两个可以设置

### Drop

丢弃此条消息, 配置 if 条件使用

```
Drop:
    if:
        - '{{if .name}}y{{end}}'
        - '{{if eq .name "childe"}}y{{end}}'
        - '{{if or (before . "-24h") (after . "24h")}}y{{end}}'
```

### Filters

目的是为了一个 if 条件后跟多个Filter

```
Filters:
    if:
        - '{{if eq .name "childe"}}y{{end}}'
    filters:
        - Add:
            fields:
                a: 'xyZ'
        - Lowercase:
            fields: ['url', 'domain']
```

### Grok

```
Grok:
    src: message
    match:
        - '^(?P<logtime>\S+) (?P<name>\w+) (?P<status>\d+)$'
        - '^(?P<logtime>\S+) (?P<status>\d+) (?P<loglevel>\w+)$'
    ignore_blank: true
    remove_fields: ['message']
```

源字段不存在, 返回 false. 所有格式不匹配, 返回 false

#### src

源字段, 默认 message

#### match

依次匹配, 直到有一个成功.

#### ignore_blank

默认 true. 如果匹配到的字段为空字符串, 则忽略这个字段. 如果 ignore_blank: false , 则添加此字段, 其值为空字符串.

### IPIP

根据 IP 信息补充地址信息, 会生成如下字段.

country_name province_name city_name 

下面四个字段视情况生成, 可能会缺失. latitude longitude location country_code

如果没有源字段, 或者寻找失败, 返回 false

```
IPIP:
    src: clientip
    target: geoip
    database: /opt/gohangout/mydata4vipday2.datx
```

#### database

数据库地址. 数据可以在 [https://www.ipip.net/](https://www.ipip.net/) 下载

#### src

源字段, 必须设置

#### target

目标字段, 如果不设置, 则将IPIP Filter生成的所有字段写入到根一层.


### Json

如果源字段不存在, 或者Json.parse 失败, 返回 false

```
Json:
    field: request
    target: request_fields
```

#### field

源字段

#### target

目标字段, 如果不设置, 则将Json Filter生成的所有字段写入到根一层.

### LinkMetric

做简单的流式统计, 统计多个字段之间的聚合数据.

```
LinkMetric:
    fieldsLink: 'domain->serverip->status_code'
    timestamp: '@timestamp'
    reserveWindow: 1800
    batchWindow: 600
    windowOffset: 0
    accumulateMode: cumulative
    drop_original_event: false
```

每600s输出一次, 输出结果形式如下:

```
{"@timestamp":1540794825600,"domain":"www.ctrip.com","serverip":"10.0.0.100","status_code":"200",count:10}
{"@timestamp":1540794825600,"domain":"www.ctrip.com","serverip":"10.0.0.200","status_code":"404",count:1}
...
```

#### fieldsLink

字段以 `->` 间隔, 统计一定时间内的聚合信息

#### timestamp

使用哪个字段做时间戳. 这个字段必须是通过 Date Filter 生成的(保证是 time.Time 类型)

#### batchWindow

多长时间内的数据聚合在一起, 单独是秒. 每隔X秒输出一次. 如果设置为1800 (半小时), 那么延时半小时以上的数据会被丢弃.

#### reserveWindow

保留多久的数据, 单独是秒. 因为数据可能会有延时, 所以需要额外保存一定时间的数据在内存中.

#### accumulateMode

两种聚合模式. 

1. cumulative 累加模式. 假设batchWindow 是300, reserveWindow 是 1800. 在每5分钟时, 会输出过去5分钟的一批聚合数据, 同时因为延时的存在, 可能还会有(过去10分钟-过去5分钟)之间的一批数据. cumulative 配置下, 会保留(过去10分钟-过去5分钟)之前count值的内存中, 新的数据进来时, 累加到一起, 下个5分钟时, 输出一个累加值.

2. separate 独立模式. 每个5分钟输出之后, 把各时间段的值清为0, 从头计数.

#### windowOffset

延时输出, 默认为0. 如果设置 windowOffset 为1 , 那么每个5分钟输出时, 最近的一个window数据保留不会输出.

#### drop_original_event

是否丢弃原始数据, 默认为 false. 如果设置为true, 则丢弃原始数据, 只输出聚合统计数据.

### LinkStatsMetric

和 LinkMetric 类似, 但最后一个字段需要是数字类型, 对它进行统计.

### Lowercase

### Remove

### Rename

### Split

```
Split:
  src: message
  sep: "\t"
  maxSplit: -1
  fields: ['logtime', 'hostname', 'uri', 'return_code']
  ignore_blank: true
  overwrite: true
```

#### src

数据来源字段, 默认 message , 如果字段不存在, 返回false

#### sep

分隔符, 在 strings.SplitN(src, sep, maxSplit) 中用被调用. 必须配置.

如果分隔符包含不可见字符, yaml配置以及gohangout也是支持的, 像下面这样

```
sep: "\x01"
```

#### maxSplit

在 strings.SplitN(src, sep, maxSplit) 中用被调用, 默认 -1, 代表无限制

#### fields

如果分割后的字符串数组长度与 fields 长度不一样, 返回false

#### ignore_blank

如果分割后的某字段为空, 刚不放后 event 中, 默认 true

### Uppercase