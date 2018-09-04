# redis+lua请求频率限制 #
## 需求 ##
在业务系统中，限流大部分情况下都是一些网关或者应用的全局配置，这样的目的是最大程度的剥离业务逻辑，降低耦合度。但是在某些场景下，
限流需要根据具体的业务数据来确定规则，这个时候全局的限流可能达不到要求，就需要在业务层面加入限流措施。

## 实现 ##
大部分情况下，我们会采用一些现成的模型或者类库在本地完成，比如guava的RateLimit，或者自己实现令牌桶、漏铜算法。
这里使用redis+lua简单实现一个单位时间内请求限流的demo，感兴趣的可以扩展成漏洞或者令牌通。

代码如下：

<pre>
local key = KEYS[1]
local count = tonumber(ARGV[1])
local expire = tonumber(ARGV[2])
local cnt = tonumber(redis.pcall('get',key))
if cnt then
    if cnt>=1 then
        redis.pcall("DECR",key)
        return "1"
    end
else
    if count>=1 then
        redis.pcall("SETEX",key,expire,count-1)
        return "1"
    end
end
return "0"
</pre>


输入参数：
* key    redis的KEY
* count  限定但是时间内的请求数
* expire 单位时间-秒

返回值：
* “0”：请求拒绝
* “1”：允许请求
