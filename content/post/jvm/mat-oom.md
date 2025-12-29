
---
title: mat工具分析oom
date: '2025-12-28T09:38:23+08:00'

categories:
- jvm
---

代码：

```java
	@Scheduled(cron = "0 17 16 * * ?")
	private static void extracted() {
		List<User> list = new ArrayList<>(10) ;
		System.out.println("start");
		AtomicInteger age = new AtomicInteger(1);
		while (true){
			User user = new User("zhangsan", age.getAndAdd(1));
		list.add(user);
		}
	}

```



虚拟机参数配置-Xmx10m -Xms10m -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=path/to/dump.hprof

也可以及时手动jmap -dump:format=b,file=dump.hprof <PID>，下载日志分析，但需要及时处理，不然可以GC掉，日志不完整，或者出现其他oom，被覆盖



### mat工具

![](F:\github\java\Tommy-s-Note\notes\img\jvm\mat初始界面.png)



可以查看到熟悉的类

![image-20250726173718480](..\img\jvm\mat分析2.png)



如果上述不易发现，可以查看线程

![image-20250726173852927](..\img\jvm\mat线程分析.png)



也可以查看

![image-20250726174202389](..\img\jvm\leak.png)

进而查看

![image-20250726174247759](..\img\jvm\leak-suspects-detail.png)

![image-20250726174327640](..\img\jvm\mat-thread-into-detail.png)

![image-20250726174412218](..\img\jvm\mat-thread.png)

![image-20250726174435378](..\img\jvm\mat线程列表.png)

![image-20250726174455745](..\img\jvm\mat-thread-detail.png)