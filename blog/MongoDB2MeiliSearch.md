---
title: MongoDB 向 MeiliSearch 同步的踩坑记录
date: 2025-04-12 00:00:00
updated: 2025-04-15 03:06:29
tags:
  - [同步]
  - [数据库]
  - [MongoDB]
  - [MeiliSearch]
urlname: MongoDB2MeiliSearch
categories:
- Technology
---

使用 meilisync

感谢yzqzss的大力支持

tl;dr: fork并大改了程序，[参见](https://github.com/Ovler-Young/meilisync)

## 崩坏的前端

先尝试使用了他的Admin Console，https://github.com/long2ice/meilisync-admin

只有AMD64，没有ARM的镜像。谁还没有x86_64的机器啊，尝试在非数据库的机器上使用。我当时还没意识到从A机拉数据到B机再塞给C机是啥概念，但总之当时犯傻了。

下镜像、运行……等等这个数据库同步的admin为什么还要MySQL和Redis的DBurl啊？不理解但是当时配置了。

随后……

1. 没有初始账户 ref [#7](https://github.com/long2ice/meilisync-admin/issues/7)

   解决方案是手动写邮箱密码进数据库，还要手搓 bcrypt hash

2. 创建MongoDB数据源，报错Unknown option user [#11](https://github.com/long2ice/meilisync-admin/issues/11)

   原因竟然是，不同数据库在后端配置文件需要的参数不同，[网页只按照PostgreSQL的user进行了传参](https://github.com/long2ice/meilisync-web/blob/main/src/views/SourceView.vue)，而且后端[没处理直接塞](https://github.com/long2ice/meilisync-admin/blob/26c20861b99e6202c9d4fe980387aaac2e71dbfb/meilisync_admin/api/source.py#L68)，sync程序原地爆炸。

   使用抓包改包重放解决。

   本来写了fix但发现PostgreSQL的参就是对的，那爱咋咋地吧我感觉我管不了。也应该没人看到这还想用吧……应该吧。

3. 设置好一切了之后还是跑不起来……后端有如下报错(删除一吨内容)：

   ```python
   2025-04-11 21:38:42.156 | INFO     | uvicorn.protocols.http.httptools_impl:send:496 - 10.0.1.1:64000 - "POST /api/sync HTTP/1.1" 500
   ERROR:    Exception in ASGI application
   Traceback (most recent call last):
     File "/meilisync_admin/meilisync_admin/models.py", line 64, in meili_client
       self.meilisearch.api_url,
       ^^^^^^^^^^^^^^^^^^^^^^^^
   AttributeError: 'QuerySet' object has no attribute 'api_url'
   ```

   额……看起来不是简单配置文件的问题了……

于是尝试使用cli，避免是那鬼畜的Admin Console导致的问题以为马上就是终结的开始，原来只是开始的终结。

## 滞后的docker

找到实际进行同步的程序，https://github.com/long2ice/meilisync

我还是尝试了Docker，毕竟是”Recommended“的方法。但是按照他Readme写的compose，

```yaml
version: "3"
services:
  meilisync:
    image: long2ice/meilisync
    volumes:
      - ./config.yml:/meilisync/config.yml
    restart: always
```

拉下来的镜像有问题。

我遇到的是 TypeError: 'async for' requires an object with __aiter__ method, got list [#94](https://github.com/long2ice/meilisync/issues/94)
但对应还有 TypeError: 'async for' requires an object with __aiter__ method, got coroutine [#76](https://github.com/long2ice/meilisync/issues/76)

嗯，得用dev呢。

啊对和上文一样，他MongoDB的user的字段是username，和模板不一样。鬼知道当时我怎么能灵光一现想出是username的。

后面配置文件来回几次之后不想搞docker了，故转为本地cli。

## 本地Python

本地cli阶段，一切似乎向着好的方向好起来了。

为数不多的几个问题就是，虽然有 `pip install meilisync[mongo]` for MongoDB，但是只安装这个是不够的。实际运行任何命令都会狠狠的告诉你，缺这缺那。你只能 `pip install meilisync[all]` for all.

还有小bug要打zsh。

```sh
$ pip install meilisync[mongo]
zsh: no matches found: meilisync[mongo]
```

终于配置好config，一切似乎在向好发展……或者是么？

### 爆炸的进度

参考 [#17](https://github.com/long2ice/meilisync/issues/17) 中的[回复](https://github.com/long2ice/meilisync/issues/17#issuecomment-1718921628)，当以MongoDB作为数据源时，`progress.json`  可能不会自动生成，导致一堆一堆的`TypeError: meilisync.progress.file.File.set() argument after ** must be a mapping, not NoneType`。

解决方案，先touch一下`progress.json`，再写进案例……

```json
{"resume_token": {"_data": "8267FBA647000000022B042C0100296E5A10046F963A9EB7AB4D14B8CF191E8E5E8D67463C6F7065726174696F6E54797065003C696E736572740046646F63756D656E744B65790046645F6964006467FBA6470D168B18625CC73E000004"}}#                                                                                   
```

### 狠狠的log

测试没发现大问题之后，改配置文件关了debug，使用nohup运行起来之后我就去干别的事了，直到硬盘告警把我拉回到shell。同步数据库到新地方的确很耗空间，我也准备好了。但我实在没想到是中间的这位硬盘先爆炸了——增速甚至大于Meili数据库的机器——人畜无害的同步器给我摔了6个G的日志到我脸上。

这不可能啊，我明明在配置文件中写了debug=false啊——

tail了一下巨大的log，发现他把每一条同步的内容全纯文本的记录下来了……

其实是默认的插件实例导致的。

在配置文件中存在如下的内容：

```yaml
debug: false
plugins:
  - meilisync.plugin.Plugin
```

其中plugin部分实际引用的是 https://github.com/long2ice/meilisync/blob/dev/meilisync/plugin.py

```python
class Plugin:
    is_global = False

    async def pre_event(self, event: Event):
        logger.debug(f"pre_event: {event}, is_global: {self.is_global}")
        return event

    async def post_event(self, event: Event):
        logger.debug(f"post_event: {event}, is_global: {self.is_global}")
        return event
```

在这个plugin中，不管配置文件中debug的设置为何值，都会写入debug。

解决方案有三种：

1. 不引用这个plugin

2. 修改plugin内容

3. 修改全局的log级别：

   Meilisync使用loguru，参考其[文档](https://loguru.readthedocs.io/en/stable/resources/troubleshooting.html#how-do-i-set-the-logging-level)可以通过设置level实现，再根据其[环境变量](https://loguru.readthedocs.io/en/stable/api/logger.html#env)的相关文档，可以设置`LOGURU_LEVEL`，可采用值如下表：

   | Level name | Severity value | Logger method                                                |
   | ---------- | -------------- | ------------------------------------------------------------ |
   | `TRACE`    | 5              | [`logger.trace()`](https://loguru.readthedocs.io/en/stable/api/logger.html#loguru._logger.Logger.trace) |
   | `DEBUG`    | 10             | [`logger.debug()`](https://loguru.readthedocs.io/en/stable/api/logger.html#loguru._logger.Logger.debug) |
   | `INFO`     | 20             | [`logger.info()`](https://loguru.readthedocs.io/en/stable/api/logger.html#loguru._logger.Logger.info) |
   | `SUCCESS`  | 25             | [`logger.success()`](https://loguru.readthedocs.io/en/stable/api/logger.html#loguru._logger.Logger.success) |
   | `WARNING`  | 30             | [`logger.warning()`](https://loguru.readthedocs.io/en/stable/api/logger.html#loguru._logger.Logger.warning) |
   | `ERROR`    | 40             | [`logger.error()`](https://loguru.readthedocs.io/en/stable/api/logger.html#loguru._logger.Logger.error) |
   | `CRITICAL` | 50             | [`logger.critical()`](https://loguru.readthedocs.io/en/stable/api/logger.html#loguru._logger.Logger.critical) |

   然后设置环境变量：
   
   Unix下：
   
   ```bash
   export LOGURU_LEVEL=INFO
   ```
   
   Windows下：
   
   PowerShell
   
   ```powershell
   $env:LOGURU_LEVEL="INFO"
   ```
   
   CMD
   
   ```cmd
   set LOGURU_LEVEL=INFO
   ```
   

后续来看节省了一吨的空间……

在各种debug中，把这个服务从B迁移到了C，也就是Meili Search所在的地方。这在后来被证明提升了非常多的速度。

### Index的疑虑

我的数据中有id一项，但实际使用中会冒出各种问题，还是使用_id作为主键。

## 动手改脚本

### 矫正了类型

似乎一切正常的运行了一阵子之后，程序自己就死掉了。几次检查之后发现是 `TypeError: Object of type ObjectId is not JSON serializable`。此时的进度大概都是1,140,000条数据。

同样有GitHub Issue，[#16](https://github.com/long2ice/meilisync/issues/16)，说是”fixed“。检查了一下本地代码，的确包含了fix的内容。但似乎还有出现在 [#102](https://github.com/long2ice/meilisync/issues/102)，这次就没有任何回复了。

最难搞的不是修代码，而是让它再出错。由于出错之后进度就g了，每次都是从头开始，而每次跑到错误地方需要20分钟，消耗了几个小时在这个上面……还有，运行期间CPU全都拉满……我还应该庆幸不是用的小服务商机器会被拉闸……

对了，这段时间，用上了sentry.io。很奇怪作者在这个同步工具中特别留了sentry.io的口子，但它真的非常有用。也许作者知道会在各种地方出bug？

### 本地的修改

于是再实现了一下检测与修复。一开始尝试改造那个plugin，但一开始实在没理顺内容，最后决定硬改源码！添加了额外的类型检查。不想一次次的pip install，直接进`site-packages`改文件，又快又好。

修复完 `ObjectId` 不久，看超过半小时都没问题，进度也在一点点的走，正准备去睡觉，结果又有报错，这次是`Object of type datetime is not JSON serializable`，类似 [#31](https://github.com/long2ice/meilisync/issues/31) 。同样的，加检查。这次是在5,270,000的位置，大概三分之一。修好之后能接着同步，也过了一半，于是我放心的去睡觉了。

然后早上醒来又是晴天霹雳，在差不多三分之二的时候，就会冒出`Client error '408 Request Timeout' for url 'http://127.0.0.1:7700/tasks/xxxx`。实在没办法，索引的东西太多太多，算不过来也就越积压越多，直至彻底boom。但不正常的是，这个问题也被修过，在 [#13](https://github.com/long2ice/meilisync/issues/13) 有提起过，但不知道啥原因，还是爆炸了。此时我检查了一下积压了多少，发现运行一个小时就会积压半个小时……我应该庆幸没有发生Too many open files的问题……

索引慢是没办法的……欸等等，说到底这索引就不应该这么快加才对吧！

### 延迟的索引

我决定尝试延迟，然后发现可怕的事实：在创建index的时候，meilisync 没有指定任何的字段索引选项。所以文档中的每个字段都会显示并可搜索，这耗费了超级多的资源，也造成可怕的浪费。我们完全没必要一开始就索引全部的内容。相反，我们在从远端同步数据的时候，不应该建立任何的索引，而应该等到直到所有来自源的内容全都都成功被插入了之后，再做索引的处理，而且应该能在config file中指定哪些字段被索引，包括索引的类型（searchable, sortable,  filterable, none )

于是写了。目前写了从远端同步数据的时候，不应该建立任何的索引的逻辑，同步速度提升了一万倍。

然后写了在同步完之后通过改setting设置索引的功能，一切看似非常正常，直到一觉睡醒还是没有任何索引。

这不应该啊。检查之后发现，虽然提交了修改索引的任务，但跑到一大半爆炸了：

```python
Index `nmbxd`: internal: MDB_TXN_FULL: Transaction has too many dirty pages - transaction too big.
```

终于不是meilisync的问题了！

### 优化的内存

仔细确认之后发现，其实不是跑了几个小时。它跑了几十分钟就transaction too big了，然后缩小batch重试，直到咋都试不出来。

锻炼！

经@yzqzss提醒了最佳实现是先设定attribution，再同步index，遵循最佳实现，但那样的话可能再出现408。要解决408得实现队列，我比较懒不想再写代码了。

随后找了下竟然找到了能减少index时内存占用的flag，参考 https://github.com/meilisearch/meilisearch/issues/3603 ，`--experimental-reduce-indexing-memory-usage`，的确一用就灵，一次成功。

以及关于更新，运行meilisync refresh即可。

再之后就是写systemd来定时进行同步了，如下。

```toml
# /etc/systemd/system/meilisync.timer
[Unit]
Description=Run meilisync refresh nmbxd weekly on Monday at 5 AM

[Timer]
# Run every Monday at 5:00 AM local time
OnCalendar=Mon *-*-* 05:00:00
Persistent=false

[Install]
WantedBy=timers.target
```

```toml
# /etc/systemd/system/meilisync.service
[Unit]
Description=Meilisync Refresh nmbxd
#After=network.target

[Service]
Type=oneshot
WorkingDirectory=/path/to/meilisync/config/
Environment='LOGURU_LEVEL=DEBUG' 
ExecStart=/etc/meilisync/meilisync/bin/meilisync refresh
```

配置文件`/path/to/meilisync/config/config.yml`

```yaml
debug: false
progress:
  type: file
source:
  type: mongo
  host: REDACTED
  port: REDACTED
  username: 'REDACTED'
  password: 'REDACTED'
  database: REDACTED
meilisearch:
  api_url: http://127.0.0.1:REDACTED
  api_key: REDACTED
  insert_size: 10000
  insert_interval: 10
sync:
  - table: REDACTED
    index: REDACTED
    full: true
    pk: _id
    attributes:
      id: [filterable, sortable]
      fid: [filterable]
      img: [filterable]
      ext: [filterable, sortable]
      now: [filterable, sortable]
      name: [searchable]
      title: [searchable]
      content: [searchable]
      parent: [filterable, sortable]
      type: [filterable]
      userid: [filterable]
sentry:
  dsn: ''
  environment: 'production'
```
