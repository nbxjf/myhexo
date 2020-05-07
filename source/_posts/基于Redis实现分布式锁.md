---
title: 基于Redis实现分布式锁
date: 2017-12-13 15:58:00
categories: Java
tags:
 - Redis
 - 分布式锁
---

本文包含以下内容：

* 基于Redis实现分布式锁

<!-- more -->

Redis本身是单线程的，所以本身没有锁的概念。
所以分布式锁的实现原理是往Redis当中写入一个key(调用方法setnx)，写入成功相当于获取锁成功。写入失败也即是setnx方法返回0，获取锁失败。
注意锁的失效时间，否则容易造成死锁。

```Java
package com.yit.common.utils.redis;

import java.util.concurrent.TimeUnit;

import com.yit.annotation.NotThreadSafe;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import redis.clients.jedis.Jedis;

/**
 * 使用redis作为分布式锁实现，分布式，不可重入
 * 不要多线程共享同一个分布式锁，多线程共享同一个分布式锁反而会失去锁"分布式"的特性
 * 使用示例:
 * <pre> {@code
 *     DefaultJedisLockFactory jedisLockFactory = new DefaultJedisLockFactory("common", "default");
 *     JedisLock jedisLock = jedisLockFactory.buildLock(1);
 *     if (jedisLock.tryLock()) {
 *         try {
 *              System.out.println("acquire lock success, do something...");
 *          } finally {
 *              //锁释放
 *              jedisLock.release();
 *          }
 *      } else {
 *          System.out.println("acquire lock failed");
 *      }
 *  }</pre>
 *
 * @author Alois Belaska <alois.belaska@gmail.com>
 * @link https://github.com/abelaska/jedis-lock
 * Redis distributed lock implementation
 * (fork by  Bruno Bossola <bbossola@gmail.com>)
 */
@NotThreadSafe
public class JedisLock {
    /**
     * 默认过期时间
     */
    public static final int DEFAULT_EXPIRY_TIME_MILLIS = Integer.getInteger("com.common.jedis.lock.expiry.millis",
        60 * 1000);
    /**
     * 默认重新申请锁的重试等待时间
     */
    public static final int DEFAULT_ACQUIRY_RESOLUTION_MILLIS = Integer.getInteger(
        "com.common.jedis.lock.acquiry.resolution.millis", 100);
    /**
     * 分布式锁使用的redis锁命名空间
     */
    private static final String LOCK_NAMESAPACE = "_LOCK";
    private static final Logger logger = LoggerFactory.getLogger(JedisLock.class);
    private final String lockKeyPath;
    private final int lockExpiryInMillis;

    public int getLockExpiryInMillis() {
        return lockExpiryInMillis;
    }

    /**
     * Detailed constructor with default lock expiration of 60000 msecs.
     *
     * @param namespace 命名空间
     * @param lockKey   lock key (ex. account:1, ...)
     */
    public JedisLock(String namespace, String lockKey) {
        this(namespace, lockKey, DEFAULT_EXPIRY_TIME_MILLIS);
    }

    /**
     * Detailed constructor.
     *
     * @param namespace        命名空间
     * @param lockKey          lock key (ex. account:1, ...)
     * @param expiryTimeMillis lock expiration in miliseconds (default: 60000 msecs)
     */
    public JedisLock(String namespace, String lockKey, int expiryTimeMillis) {
        this.lockKeyPath = namespace + "/" + LOCK_NAMESAPACE + "/" + lockKey;
        this.lockExpiryInMillis = expiryTimeMillis;
    }

    /**
     * @return lock key path
     */
    public String getLockKeyPath() {
        return lockKeyPath;
    }

    /**
     * 0等，上锁失败立即返回false，不可重入
     * 分布式:依赖redis setnx原子操作
     * Acquire lock.
     * 注意: 如果争用到锁却不调用release函数（分布式锁自动失效），需要注意调用closeJedis，释放jedis实例
     *
     * @return true if lock is acquired, false acquire timeouted
     * @throws InterruptedException in case of thread interruption
     */
    public synchronized boolean tryLock() throws InterruptedException {
        return tryLock(0, TimeUnit.MILLISECONDS);
    }

    /**
     * 超时后返回false，不可重入
     * 分布式:依赖redis setnx原子操作
     * Acquire lock.
     * 注意: 如果争用到锁却不调用release函数（分布式锁自动失效），需要注意调用closeJedis，释放jedis实例
     *
     * @param timeout 锁超时时间
     * @param unit    时间单位
     * @return true if lock is acquired, false acquire timeouted
     * @throws InterruptedException in case of thread interruption
     */
    public synchronized boolean tryLock(int timeout, TimeUnit unit)
        throws InterruptedException {
        boolean locked = false;
        Jedis jedis = null;
        try {
            jedis = JedisClientFactory.getJedisClient();
            long timeoutMillis = unit.toMillis(timeout);
            while (timeoutMillis >= 0) {
                //NX -- Only set the key if it does not already exist.
                //XX -- Only set the key if it already exist.
                //EX seconds -- Set the specified expire time, in seconds.
                //PX milliseconds -- Set the specified expire time, in milliseconds.
                String result = jedis.set(lockKeyPath, "1", "NX", "PX",
                    lockExpiryInMillis);
                if ("OK".equalsIgnoreCase(result)) {
                    locked = true;
                    break;
                }
                //超时时间递减
                timeoutMillis -= DEFAULT_ACQUIRY_RESOLUTION_MILLIS;
                if (timeoutMillis > 0) {
                    Thread.sleep(DEFAULT_ACQUIRY_RESOLUTION_MILLIS);
                }
            }
        } catch (Exception e) {
            logger.error("{} trylock failed!", lockKeyPath, e);
        } finally {
            //及时释放jedis链接
            forceClose(jedis);
        }
        return locked;
    }

    /**
     * 释放锁
     * 注意：release已经隐含了forceClose动作，请勿在调用release之前进行jedis实例释放
     * Acquired lock release.
     */
    public synchronized void release() {
        Jedis jedis = null;
        try {
            jedis = JedisClientFactory.getJedisClient();
            jedis.del(lockKeyPath);
        } finally {
            forceClose(jedis);
        }
    }

    /**
     * 释放jedis链接,内部创建的jedis实例需要触发释放
     * 注意：release已经隐含了forceClose动作，请勿在调用release之前进行jedis实例释放
     */
    public synchronized void forceClose(Jedis jedis) {
        if (jedis != null) {
            try {
                jedis.close();
            } catch (Exception e) {
                logger.error("insure jedis is closed!", e);
            }
        } else {
            logger.info("jedis is closed!");
        }
    }

    /**
     * 是否已被锁
     * Check if owns the lock
     *
     * @return true if lock owned
     */
    public synchronized boolean isLocked() {
        Jedis jedis = null;
        boolean isExists = false;
        try {
            jedis = JedisClientFactory.getJedisClient();
            isExists = jedis.exists(lockKeyPath);
        } finally {
            forceClose(jedis);
        }
        return isExists;
    }
}
```


