---
{"dg-publish":true,"permalink":"/科研/Retina/MileStone/Issue 1189/"}
---

# Cache 性能

>负载：16 线程 load + update，每个线程各自 load 1kw，然后 update 100w

## 基准测试

使用 String（Thread id + num）作为 key，num 作为 value

|                   | load 耗时（s） | update 吞吐（ops/s） |
| ----------------- | ---------- | ---------------- |
| ConcurrentHashMap | 9.29       | 20592020.59（千万）  |
| Guava（1kw）        | 189.56     | 775682.36（十万）    |
| Guava（2e）         | 167.30     | 740260.94（十万）    |
| Caffeine（1kw）     | 145.77     | 1353523.39（百万）   |
| Caffeine（2e）      | 48.84      | 12130401.82（千万）  |

## Cache 预实验

使用 IndexProto.IndexKey 作为 key，num 作为 value

|                   | load 耗时（s） | update 吞吐（ops/s） |
| ----------------- | ---------- | ---------------- |
| ConcurrentHashMap | 467.14     | 291630.21（十万）    |
| Caffeine（1kw）     |            |                  |
| Caffeine（2e）      | timeout    |                  |

## Cache 优化思路

key 的类型严重影响了性能，主要原因在于自定义实现的 Hash 不够好
尝试将 IndexProto.IndexKey 分解，将 tableId，indexId，key 拼凑为 String 作为 key，timestamp，rowId 拼凑为 String 作为 value

|                   | load 耗时（s） | update 吞吐（ops/s） |
| ----------------- | ---------- | ---------------- |
| ConcurrentHashMap | 25.95      | 10876954.45（千万）  |
| Caffeine（1kw）     | 183.53     | 960730.15（十万）    |
| Caffeine（2e）      | 85.44      | 9900990.10（百万）   |

## 测试细节

* 测试程序的 jvm 设置了 -Xmx204800m，即200G
* num 是迭代中的值，比如 load 是 num 是 0 - (1kw -1)
* 负载中的 update 操作使用 get + put 替代，使用的 key 均是当前迭代值
* Guava 和 Caffeine 设置的过期时间均为 300s，在测试中不会被清理掉
* 总共进行 1.6e + 1600kw = 1.76e 插入，2e 缓存大小足够最佳
* 基准测试后直接排除 Guava，故后面实验没有
* Cache 预实验 key 中的 timestamp 设置为 0，保证正常 get 
* Cache 预实验使用 Caffeine 加载时间超过了 27 分钟，不再测试
