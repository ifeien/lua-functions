# redis+lua 验证码功能 #
## 需求 ##
系统中处于安全性考虑经常会使用验证码来检验用户的身份或者非机器人行为，通常的方法都是使用程序生成验证码并将验证码保存在内存、session、或者缓存中。等用户提交时，根据业务需求（错误3次以内，15分钟有效时间）判定验证码是否正确。这中间的需求一般都比较通用限定错误次数，限定有效时间ttl。

### 实现 ###
这里使用redis的lua脚本功能来设计一个比较通用的验证码逻辑，不再依赖于特定的业务代码，使用lua有助于减少代码与缓存之间的网络交互次数。整个逻辑可以分成2部分：
* 设置验证码以及规则(错误次数，ttl)
* 校验输入的验证码

这里分步骤说明：
#### 第一步 设置规则 ####
输入参数有4个：
 * key 
 * 验证码
 * 错误次数
 * ttl（可选）

代码如下：
<pre>
local key = KEYS[1]
local code = ARGV[1]
local retry = ARGV[2]
redis.pcall('set',key,code.."|"..retry)
if #ARGV > 2 then
    if tonumber(ARGV[3])>0 then
        redis.pcall("expire",key,ARGV[3])
    end
end
return 1
</pre>

redis返回值1，对于这个操作来说是不可能失败的，返回1的意思是成功设置了1条验证码

#### 第二步 校验  ####
输入参数2个：
* key
* 验证码

代码如下：
<pre>
local key = KEYS[1]
local tryCode = ARGV[1]
local val = redis.pcall("get",key)
if val then
    local i = string.find(val, "|")
    if i then
        local code = string.sub(val, 0, i-1)
        local cnt = tonumber(string.sub(val, i+1))
        if code==tryCode then
            redis.pcall("del",key)
            return "1"
        elseif cnt>1 then
            local ttl = tonumber(redis.pcall("ttl",key))
            if ttl and ttl>0 then
                redis.pcall("SETEX",key,ttl,code.."|"..tostring(cnt-1))
            elseif ttl==-1 then
                redis.pcall("SET",key,code.."|"..tostring(cnt-1))
            end
        else
            redis.pcall("del",key)
        end
    end
end
return "0"
</pre>

返回值：
* “1”：验证码正确
* “0”：验证码错误

这里不使用true和false的原因是lua和redis之间数据转换的问题， Lua 的布尔值 false 转换成 Redis 的 Nil bulk 回复
