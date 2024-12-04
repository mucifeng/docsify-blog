## 分布式锁
### Redis实现方案
#### 案例一
实现思路: 主要是使用了redis 的setnx命令,缓存了锁。
```
只在键 key 不存在的情况下， 将键 key 的值设置为 value 。
若键 key 已经存在， 则 SETNX 命令不做任何动作。
SETNX 是『SET if Not eXists』(如果不存在，则 SET)的简写
```
```java
//伪代码
public static boolean lock(String key,String lockValue,int expire){
    if(null == key)
        return false;
    Jedis jedis = null;
    try {
        jedis = jedisManager.getResource();
        String res = jedis.setNX(key,lockValue); 
        if(OK.equals(res))
            jedis.expire(expire);
            return true;
        else
            return false;
    } catch (Exception e) {
        return false;
    }final {
        if(jedis != null) 
            jedis.close();
    }
}
这种写法接口，先setnx()再使用expire()设置key的过期时间，可以满足大多数需求.但是还是有必要完善的地方
问题一: 代码中首先是setnx，在expire还没成功时，系统崩溃导致key键没有过期时间一直存在。则其余需要lock的地方永远都没办法拿到锁
问题二: 在Redis哨兵模式中，可能会有同步不及时问题发生，A进程在机器AM中setnx A->AValue，几乎同时B进程在机器BM中setnx B->BValue。
    一系列操作完成后AM 和BM 机器还没同步。此时两台机器拿到的值是不一样的
问题三: 无法实现公平锁功能
```
#### 案例二
使用Redis Lua脚本的功能
```
Lock:
if redis.call('setnx',KEYS[1],ARGV[1]) == 1 
    then return redis.call('expire',KEYS[1],ARGV[2])  
else return 0 
end
这种写法 可以保证 setnx和expire是一个原子操作，避免了赋值的先后宕机问题，但是对于分布式Redis系统还是没能很好解决不一致问题
对于公平锁的实现，可以考虑使用redis中的list写在Lua中
公平锁伪代码：
if redis.call('setnx',KEYS[1],ARGV[1]) == 1 
    then return redis.call('expire',KEYS[1],ARGV[2])  
else 
    reids.call('rpush', '某工程:某业务:KEYS[1]', ARGV[1])
    return 0 
end
解锁伪代码:
if redis.call('get',KEYS[1]) == ARGV[1] 
    first = reids.call('lpop', '某工程:某业务:KEYS[1]')
    if first not null
    then LOCK操作
else return 0 
end
```
#### 案例三
使用开源Redisson库<br>
```
RLock lock = redisson.getLock("myLock");
lock.lock();
lock.unlock();
```
盗个图(侵删)<br>
![redisson](https://img2018.cnblogs.com/blog/759814/201811/759814-20181126113038352-583246905.png)
<br>
