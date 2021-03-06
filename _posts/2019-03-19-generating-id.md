---
layout:     post                    # 使用的布局（不需要改）
title:      Generating unique IDs in a distributed environment      # 标题 
subtitle:       #副标题
date:       2019-03-19              # 时间
author:     ZYT                     # 作者
header-img: img/uniqueId.jpg   #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:
    - 基础架构        #标签
---

# 解决方案

## UUID

UUIDs are 128-bit hexadecimal numbers that are globally unique. The chances of the same UUID getting generated twice is negligible.

优点：

- 性能非常高：本地生成，没有网络消耗。

缺点：

- 不易于存储：UUID太长，16字节128位，通常以36长度的字符串表示，很多场景不适用。
- 信息不安全：基于MAC地址生成UUID的算法可能会造成MAC地址泄露。
- ID不适用作为DB主键。

## MongoDB's ObjectId

MongoDB’s ObjectIDs are 12-byte (96-bit) hexadecimal numbers that are made up of -

- 4-byte epoch timestamp in seconds,
- 3-byte machine identifier,
- 2-byte process id, and
- 3-byte counter, starting with a random value.

## Twitter Snowflake

Twitter snowflake is a dedicated network service for generating 64-bit unique IDs at high scale. The IDs generated by this service are roughly time sortable.

The IDs are made up of the following components:

- Epoch timestamp in millisecond precision - 41 bits (gives us 69 years with a custom epoch)
- Configured machine id - 10 bits (gives us up to 1024 machines)
- Sequence number - 12 bits (A local counter per machine that rolls over every 4096)

**理论上snowflake方案的QPS约为409.6w/s**

优点：

- 毫秒数在高位，自增序列在低位，整个ID都是趋势递增的。
- 不依赖数据库等第三方系统，以服务的方式部署，稳定性更高，生成ID的性能也是非常高的。
- 可以根据自身业务特性分配bit位，非常灵活。

缺点：

- 强依赖机器时钟，如果机器上时钟回拨，会导致发号重复或者服务会处于不可用状态。

可行的解决方案：

1. 利用 ZooKeeper 等协调服务管理时间
2. 关闭 NTP 服务

### Snowflake 的各种语言实现

表示 timestamp 的41位，可以支持我们使用69年。当然，我们的时间毫秒计数不会真的从 1970 年开始记，那样我们的系统跑到 2039/9/7 23:47:35 就不能用了，所以这里的 timestamp 实际上只是相对于某个时间的增量，比如我们的系统上线是 2019-01-01，那么我们可以把这个 timestamp 当作是从 2019-01-01 00:00:00.000的偏移量。

#### python
```python
"""
┌─┐┌┐┌┌─┐┬ ┬┌─┐┬  ┌─┐┬┌─┌─┐
└─┐││││ ││││├┤ │  ├─┤├┴┐├┤
└─┘┘└┘└─┘└┴┘└  ┴─┘┴ ┴┴ ┴└─┘
"""

import time
import random
import threading


class Snowflake(object):
    # 22 bit = 2 region + 8 worker + 12 sequence (< 4096)
    region_id_bits = 2
    worker_id_bits = 8
    sequence_bits = 12

    MAX_REGION_ID = -1 ^ (-1 << region_id_bits)
    MAX_WORKER_ID = -1 ^ (-1 << worker_id_bits)
    SEQUENCE_MASK = -1 ^ (-1 << sequence_bits)

    WORKER_ID_SHIFT = sequence_bits
    REGION_ID_SHIFT = sequence_bits + worker_id_bits
    TIMESTAMP_LEFT_SHIFT = sequence_bits + worker_id_bits + region_id_bits

    def __init__(self, worker_id, region_id=0):
        self.twepoch = 1495977602000
        self.last_timestamp = -1
        self.sequence = 0

        # assert 0 <= worker_id <= Snowflake.MAX_WORKER_ID
        # assert 0 <= region_id <= Snowflake.MAX_REGION_ID

        self.worker_id = worker_id
        self.region_id = region_id

        self.lock = threading.Lock()

    def generate(self, bus_id=None):
        return self.next_id(
            True if bus_id is not None else False, bus_id if bus_id is not None else 0
        )

    def next_id(self, is_padding, bus_id):
        with self.lock:
            timestamp = self.get_time()
            padding_num = self.region_id

            if is_padding:
                padding_num = bus_id

            if timestamp < self.last_timestamp:
                try:
                    raise ValueError(
                        "Clock moved backwards. Refusing to"
                        "generate id for {0} milliseconds.".format(
                            self.last_timestamp - timestamp
                        )
                    )
                except ValueError as error:
                    print(error)

            if timestamp == self.last_timestamp:
                self.sequence = (self.sequence + 1) & Snowflake.SEQUENCE_MASK
                if self.sequence == 0:
                    timestamp = self.tail_next_millis(self.last_timestamp)
            else:
                self.sequence = random.randint(0, 9)

            self.last_timestamp = timestamp

            res_id = (
                (timestamp - self.twepoch) << Snowflake.TIMESTAMP_LEFT_SHIFT
                | (padding_num << Snowflake.REGION_ID_SHIFT)
                | (self.worker_id << Snowflake.WORKER_ID_SHIFT)
                | self.sequence
            )

            return res_id

    def tail_next_millis(self, last_timestamp):
        timestamp = self.get_time()
        while timestamp <= last_timestamp:
            timestamp = self.get_time()
        return timestamp

    def get_time(self):
        return int(time.time() * 1000)


if __name__ == "__main__":
    print(Snowflake(0).generate())
```

#### golang
```
package main

import (
	"fmt"

	"github.com/bwmarrin/snowflake"
)

func main() {

	// Create a new Node with a Node number of 1
	node, err := snowflake.NewNode(1)
	if err != nil {
		fmt.Println(err)
		return
	}

	fmt.Println(stepMask)
	// Generate a snowflake ID.
	id := node.Generate()

	// Print out the ID in a few different ways.
	fmt.Printf("ID       : %d\n", id)
	// Print out the ID's sequence number
	fmt.Printf("ID Step  : %d\n", id.Step())
}
```

#### java
```java
import java.net.NetworkInterface;
import java.security.SecureRandom;
import java.time.Instant;
import java.util.Enumeration;

/**
 * Distributed Sequence Generator.
 * Inspired by Twitter snowflake: https://github.com/twitter/snowflake/tree/snowflake-2010
 *
 * This class should be used as a Singleton.
 * Make sure that you create and reuse a Single instance of SequenceGenerator per node in your distributed system cluster.
 */
public class SequenceGenerator {
    private static final int TOTAL_BITS = 64;
    private static final int EPOCH_BITS = 42;
    private static final int NODE_ID_BITS = 10;
    private static final int SEQUENCE_BITS = 12;

    private static final int maxNodeId = (int)(Math.pow(2, NODE_ID_BITS) - 1);
    private static final int maxSequence = (int)(Math.pow(2, SEQUENCE_BITS) - 1);

    // Custom Epoch (January 1, 2015 Midnight UTC = 2015-01-01T00:00:00Z)
    private static final long CUSTOM_EPOCH = 1420070400000L;

    private final int nodeId;

    private volatile long lastTimestamp = -1L;
    private volatile long sequence = 0L;

    // Create SequenceGenerator with a nodeId
    public SequenceGenerator(int nodeId) {
        if(nodeId < 0 || nodeId > maxNodeId) {
            throw new IllegalArgumentException(String.format("NodeId must be between %d and %d", 0, maxNodeId));
        }
        this.nodeId = nodeId;
    }

    // Let SequenceGenerator generate a nodeId
    public SequenceGenerator() {
        this.nodeId = createNodeId();
    }


    public synchronized long nextId() {
        long currentTimestamp = timestamp();

        if(currentTimestamp < lastTimestamp) {
            throw new IllegalStateException("Invalid System Clock!");
        }

        if (currentTimestamp == lastTimestamp) {
            sequence = (sequence + 1) & maxSequence;
            if(sequence == 0) {
                // Sequence Exhausted, wait till next millisecond.
                currentTimestamp = waitNextMillis(currentTimestamp);
            }
        } else {
            // reset sequence to start with zero for the next millisecond
            sequence = 0;
        }

        lastTimestamp = currentTimestamp;

        long id = currentTimestamp << (TOTAL_BITS - EPOCH_BITS);
        id |= (nodeId << (TOTAL_BITS - EPOCH_BITS - NODE_ID_BITS));
        id |= sequence;
        return id;
    }


    // Get current timestamp in milliseconds, adjust for the custom epoch.
    private static long timestamp() {
        return Instant.now().toEpochMilli() - CUSTOM_EPOCH;
    }

    // Block and wait till next millisecond
    private long waitNextMillis(long currentTimestamp) {
        while (currentTimestamp == lastTimestamp) {
            currentTimestamp = timestamp();
        }
        return currentTimestamp;
    }

    private int createNodeId() {
        int nodeId;
        try {
            StringBuilder sb = new StringBuilder();
            Enumeration<NetworkInterface> networkInterfaces = NetworkInterface.getNetworkInterfaces();
            while (networkInterfaces.hasMoreElements()) {
                NetworkInterface networkInterface = networkInterfaces.nextElement();
                byte[] mac = networkInterface.getHardwareAddress();
                if (mac != null) {
                    for(int i = 0; i < mac.length; i++) {
                        sb.append(String.format("%02X", mac[i]));
                    }
                }
            }
            nodeId = sb.toString().hashCode();
        } catch (Exception ex) {
            nodeId = (new SecureRandom().nextInt());
        }
        nodeId = nodeId & maxNodeId;
        return nodeId;
    }
}
```
