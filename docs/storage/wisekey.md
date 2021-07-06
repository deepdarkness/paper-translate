# WiscKey 在固态存储中将键值分开 

Lanyue Lu, Thanumalayan Sankaranarayana Pillai,
Andrea C. Arpaci-Dusseau, Remzi H. Arpaci-Dusseau

威斯康星大学麦迪逊分校

## 摘要

WiscKey, 一个基于LSM树的持久键值存储，具有面向性能的数据布局，该功能将键与值分开，以最大程度地减少I/O放大。 WiscKey的设计经过SSD的高度优化，充分利用了设备的顺序性能和随机性能。我们通过微基准测试和YCSB工作负载展示了WiscKey的优势。微基准测试结果表明，WiscKey的速度比LevelDB快2.5倍至111倍，而数据库的加载速度则快1.6倍至14倍。在所有六个YCSB工作负载中，WiscKey比LevelDB和RocksDB都快。 

## 1 介绍

持久键值存储在各种现代数据密集型应用程序中扮演着至关重要的角色，包括Web索引[16、48]，电子商务[24]，重复数据删除[7、22]，照片存储[12]，云 数据[32]，社交网络[9、25、51]，在线游戏[23]，消息传递[1、29]，软件存储库[2]和广告[20]。 通过启用有效的插入，点查找和范围查询，键值存储成为了这一组不断增长的重要应用程序的基础。 

## 2 背景和动机

### 2.1 Log-Structured Merge-Tree

## 3 WiscKey

### 3.1 设计目标
