---
title: CONCURRENT
date: '2025-12-28T09:38:23+08:00'

categories:
- juc
---


# CONCURRENT



文件导入导出

```java
@Test
    public void importPipelineBatchingExcel(InputStream inputStream) {

        ImportExcel importExcel = new ImportExcel();
        LinkedList<PipelineBatchingExcelVO> read = ExcelUtil.read(inputStream, PipelineBatchingExcelVO.class);
        ConcurrentHashMap<String, PipelineEntity> concurrentHashMap = new ConcurrentHashMap<>();
        List<PipelineBatchingEntity> result = new ArrayList<>();
        if (read.size() <= 100) {
            List<PipelineBatchingEntity> list = new CopyOnWriteArrayList<>();

            for (PipelineBatchingExcelVO pipelineBatchingVO : read) {
                PipelineBatchingEntity entity = importExcel.getByPipelineBatchingVO(pipelineBatchingVO, concurrentHashMap);
                list.add(entity);
            }
            pipelineBatchingService.insertBatch(list);
        } else {
            List<List<PipelineBatchingExcelVO>> partition = ListUtils.partition(read, 100);
            CompletableFuture[] completableFutures = partition.stream().map(item -> CompletableFuture.supplyAsync(() -> {
                List<PipelineBatchingEntity> list = new ArrayList<>();
                for (PipelineBatchingExcelVO pipelineBatchingVO : item) {
                    PipelineBatchingEntity entity = importExcel.getByPipelineBatchingVO(pipelineBatchingVO, concurrentHashMap);
                    list.add(entity);
                }
                return list;
            }, threadPoolExecutor).whenComplete((r, c) -> result.addAll(r))).toArray(CompletableFuture[]::new);
            CompletableFuture.allOf(completableFutures).join();

            List<List<PipelineBatchingEntity>> partition1 = Lists.partition(result, 10000);
            for (List<PipelineBatchingEntity> pipelineBatchingEntities : partition1) {
                pipelineBatchingService.insertBatch(pipelineBatchingEntities);
            }
        }
    }
}
```

```java
public void exportTest(HttpServletResponse response) {
    List<PipelineBatchingEntity> pipelineBatchingEntities = pipelineBatchingMapper.selectList(null);
    int selectCount = pipelineBatchingEntities.size();
    int pageSize = 1000;
    double pages = Math.ceil((double) selectCount / pageSize);
    HashMap<String, PipelineEntity> map = new HashMap<>();
    List<CompletableFuture<List<PipelineBatchingExportVO>>> futures = new ArrayList<>();
    List<PipelineBatchingExportVO> result = new ArrayList<>();
    for (int i = 1; i <= pages; i++) {
        int finalI = i;
        CompletableFuture<List<PipelineBatchingExportVO>> submit = CompletableFuture.supplyAsync(() -> {
            //查出来的分页数据
            List<PipelineBatchingEntity> entities = pipelineBatchingEntities.stream().skip((long) (finalI - 1) * pageSize).limit(pageSize).collect(Collectors.toList());
            List<PipelineBatchingExportVO> res = new ArrayList<>();
            for (PipelineBatchingEntity item : entities) {
                PipelineEntity pipelineEntity;
                pipelineEntity = map.get(item.getPipelineId());
                if (pipelineEntity == null) {
                    pipelineEntity = pipelineService.selectById(item.getPipelineId());
                    map.put(item.getPipelineId(), pipelineEntity);
                }
                PipelineBatchingExportVO vo = new PipelineBatchingExportVO();
                BeanUtil.copyProperties(item, vo);
                vo.setMatchingNum(item.getMatchingNum());
                vo.setComponentName(item.getProdName());
                BeanUtil.copyProperties(pipelineEntity, vo);
                vo.setPipelineNo(pipelineEntity.getPipeCode());
                vo.setPipelineNo2(pipelineEntity.getPipeCode());
                vo.setSegmentNumber(item.getPipeBatchingSectionCode());
                vo.setUnitNo(pipelineEntity.getUnitCode());
                res.add(vo);
            }
            return res;
        }, threadPoolExecutor).whenComplete((r, c) -> result.addAll(r));
        futures.add(submit);
    }
    CompletableFuture.allOf(futures.toArray(new CompletableFuture[0])).join();
    //排序
    ExcelUtil.export(response, "管道配料信息", result, PipelineBatchingExportVO.class);
}
```





gitee地址  https://gitee.com/phui/share-concurrent

```java
package org.ph.share;

import java.util.StringJoiner;

/**
 * 小工具
 */
public class SmallTool {

    public static void sleepMillis(long millis) {
        try {
            Thread.sleep(millis);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    public static void printTimeAndThread(String tag) {
        String result = new StringJoiner("\t|\t")
                .add(String.valueOf(System.currentTimeMillis()))
                .add(String.valueOf(Thread.currentThread().getId()))
                .add(Thread.currentThread().getName())
                .add(tag)
                .toString();
        System.out.println(result);
    }

}
```

### 1、createThread

```java
public class _01_CreateThread_Run {
    public static void main(String[] args) {
        Thread thread = new Thread(){
            @Override
            public void run() {
                System.out.println("我是子线程");
            }
        };
        thread.start();
        System.out.println("main 结束");
    }
}
```

```java
public class _02_CreateThread_Runnable {
    public static void main(String[] args) {
        Thread thread = new Thread(
                () -> System.out.println("我是子线程")
        );
        thread.start();
        System.out.println("main 结束");
    }
}
```

```java
public class _03_CreateThread_FutureTask {
    public static void main(String[] args) {
        Callable<String> callable = () -> {
            System.out.println("我是子任务");
            return "sub task done";
        };
        FutureTask<String> task = new FutureTask(callable);
        Thread thread = new Thread(task);
        thread.start();
        System.out.println("子线程启动");
        try {
            String subResult = task.get(5, TimeUnit.MINUTES);
            System.out.println("子线程返回值：" + subResult);
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (ExecutionException e) {
            Throwable cause = e.getCause();
            e.printStackTrace();
        } catch (TimeoutException e) {
            e.printStackTrace();
        }
        System.out.println("main 结束");
    }
}
```

```java
public class ThreadStateTest {
    public static void main(String[] args) throws InterruptedException {
        Thread thread = new Thread();
        System.out.println("1- " + thread.getState());
        thread.start();
        System.out.println("2- " + thread.getState());
        TimeUnit.SECONDS.sleep(1);
        System.out.println("3- " + thread.getState());
    }
}
```

### 2、CompletableFuture_start

##### **supplyAsync**

```java
public class _01_supplyAsync {
    public static void main(String[] args) {
        SmallTool.printTimeAndThread("小白进入餐厅");
        SmallTool.printTimeAndThread("小白点了 番茄炒蛋 + 一碗米饭");

        CompletableFuture<String> cf1 = CompletableFuture.supplyAsync(() -> {
            SmallTool.printTimeAndThread("厨师炒菜");
            SmallTool.sleepMillis(200);
            SmallTool.printTimeAndThread("厨师打饭");
            SmallTool.sleepMillis(100);
            return "番茄炒蛋 + 米饭 做好了";
        });

        SmallTool.printTimeAndThread("小白在打王者");
        SmallTool.printTimeAndThread(String.format("%s ,小白开吃", cf1.join()));
    }
}
```



##### **thenCompose**

```java
public class _02_thenCompose {
    public static void main(String[] args) {
        SmallTool.printTimeAndThread("小白进入餐厅");
        SmallTool.printTimeAndThread("小白点了 番茄炒蛋 + 一碗米饭");

        CompletableFuture<String> cf1 = CompletableFuture.supplyAsync(() -> {
            SmallTool.printTimeAndThread("厨师炒菜");
            SmallTool.sleepMillis(200);
            return "番茄炒蛋";
        }).thenCompose(dish -> CompletableFuture.supplyAsync(() -> {
            SmallTool.printTimeAndThread("服务员打饭");
            SmallTool.sleepMillis(100);
            return dish + " + 米饭";
        }));

        SmallTool.printTimeAndThread("小白在打王者");
        SmallTool.printTimeAndThread(String.format("%s 好了,小白开吃", cf1.join()));
    }

    /**
     * 用 applyAsync 也能实现
     */
    private static void applyAsync() {
        SmallTool.printTimeAndThread("小白进入餐厅");
        SmallTool.printTimeAndThread("小白点了 番茄炒蛋 + 一碗米饭");

        CompletableFuture<String> cf1 = CompletableFuture.supplyAsync(() -> {
            SmallTool.printTimeAndThread("厨师炒菜");
            SmallTool.sleepMillis(200);
            CompletableFuture<String> race = CompletableFuture.supplyAsync(() -> {
                SmallTool.printTimeAndThread("服务员打饭");
                SmallTool.sleepMillis(100);
                return " + 米饭";
            });
            return "番茄炒蛋" + race.join();
        });

        SmallTool.printTimeAndThread("小白在打王者");
        SmallTool.printTimeAndThread(String.format("%s 好了,小白开吃", cf1.join()));
    }
}
```



##### thenCombine

```java
public class _03_thenCombine {
    public static void main(String[] args) {
        SmallTool.printTimeAndThread("小白进入餐厅");
        SmallTool.printTimeAndThread("小白点了 番茄炒蛋 + 一碗米饭");

        CompletableFuture<String> cf1 = CompletableFuture.supplyAsync(() -> {
            SmallTool.printTimeAndThread("厨师炒菜");
            SmallTool.sleepMillis(200);
            return "番茄炒蛋";
        }).thenCombine(CompletableFuture.supplyAsync(() -> {
            SmallTool.printTimeAndThread("服务员蒸饭");
            SmallTool.sleepMillis(300);
            return "米饭";
        }), (dish, rice) -> {
            SmallTool.printTimeAndThread("服务员打饭");
            SmallTool.sleepMillis(100);
            return String.format("%s + %s 好了", dish, rice);
        });

        SmallTool.printTimeAndThread("小白在打王者");
        SmallTool.printTimeAndThread(String.format("%s ,小白开吃", cf1.join()));

    }


    /**
     * 用 applyAsync 也能实现
     */
    private static void applyAsync() {
        SmallTool.printTimeAndThread("小白进入餐厅");
        SmallTool.printTimeAndThread("小白点了 番茄炒蛋 + 一碗米饭");

        CompletableFuture<String> cf1 = CompletableFuture.supplyAsync(() -> {
            SmallTool.printTimeAndThread("厨师炒菜");
            SmallTool.sleepMillis(200);
            return "番茄炒蛋";
        });
        CompletableFuture<String> race = CompletableFuture.supplyAsync(() -> {
            SmallTool.printTimeAndThread("服务员蒸饭");
            SmallTool.sleepMillis(300);
            return "米饭";
        });
        SmallTool.printTimeAndThread("小白在打王者");

        String result = String.format("%s + %s 好了", cf1.join(), race.join());
        SmallTool.printTimeAndThread("服务员打饭");
        SmallTool.sleepMillis(100);

        SmallTool.printTimeAndThread(String.format("%s ,小白开吃", result));
    }
}
```

### 3、CompletableFuture_advance

##### thenApply

```java
package org.ph.share._04_CompletableFuture_advance;

import org.ph.share.SmallTool;

import java.util.concurrent.CompletableFuture;

public class _01_thenApply {
    public static void main(String[] args) {
        SmallTool.printTimeAndThread("小白吃好了");
        SmallTool.printTimeAndThread("小白 结账、要求开发票");

        CompletableFuture<String> invoice = CompletableFuture.supplyAsync(() -> {
            SmallTool.printTimeAndThread("服务员收款 500元");
            SmallTool.sleepMillis(100);
            return "500";
        }).thenApplyAsync(money -> {
            SmallTool.printTimeAndThread(String.format("服务员开发票 面额 %s元", money));
            SmallTool.sleepMillis(200);
            return String.format("%s元发票", money);
        });

        SmallTool.printTimeAndThread("小白 接到朋友的电话，想一起打游戏");

        SmallTool.printTimeAndThread(String.format("小白拿到%s，准备回家", invoice.join()));
    }


    private static void one() {
        SmallTool.printTimeAndThread("小白吃好了");
        SmallTool.printTimeAndThread("小白 结账、要求开发票");

        CompletableFuture<String> invoice = CompletableFuture.supplyAsync(() -> {
            SmallTool.printTimeAndThread("服务员收款 500元");
            SmallTool.sleepMillis(100);
            SmallTool.printTimeAndThread("服务员开发票 面额 500元");
            SmallTool.sleepMillis(200);
            return "500元发票";
        });

        SmallTool.printTimeAndThread("小白 接到朋友的电话，想一起打游戏");

        SmallTool.printTimeAndThread(String.format("小白拿到%s，准备回家", invoice.join()));
    }


    private static void two() {
        SmallTool.printTimeAndThread("小白吃好了");
        SmallTool.printTimeAndThread("小白 结账、要求开发票");

        CompletableFuture<String> invoice = CompletableFuture.supplyAsync(() -> {
            SmallTool.printTimeAndThread("服务员收款 500元");
            SmallTool.sleepMillis(100);

            CompletableFuture<String> waiter2 = CompletableFuture.supplyAsync(() -> {
                SmallTool.printTimeAndThread("服务员开发票 面额 500元");
                SmallTool.sleepMillis(200);
                return "500元发票";
            });

            return waiter2.join();
        });

        SmallTool.printTimeAndThread("小白 接到朋友的电话，想一起打游戏");

        SmallTool.printTimeAndThread(String.format("小白拿到%s，准备回家", invoice.join()));
    }
}
```

##### applyToEither

```java
package org.ph.share._04_CompletableFuture_advance;

import org.ph.share.SmallTool;

import java.util.concurrent.CompletableFuture;
import java.util.function.Function;

public class _02_applyToEither {
    public static void main(String[] args) {
        SmallTool.printTimeAndThread("张三走出餐厅，来到公交站");
        SmallTool.printTimeAndThread("等待 700路 或者 800路 公交到来");

        CompletableFuture<String> bus = CompletableFuture.supplyAsync(() -> {
            SmallTool.printTimeAndThread("700路公交正在赶来");
            SmallTool.sleepMillis(100);
            return "700路到了";
        }).applyToEither(CompletableFuture.supplyAsync(() -> {
            SmallTool.printTimeAndThread("800路公交正在赶来");
            SmallTool.sleepMillis(200);
            return "800路到了";
        }), firstComeBus -> firstComeBus);

        SmallTool.printTimeAndThread(String.format("%s,小白坐车回家", bus.join()));
    }
}
```



##### exceptionally

```java
package org.ph.share._04_CompletableFuture_advance;

import org.ph.share.SmallTool;

import java.util.concurrent.CompletableFuture;

public class _03_exceptionally {
    public static void main(String[] args) {
        SmallTool.printTimeAndThread("张三走出餐厅，来到公交站");
        SmallTool.printTimeAndThread("等待 700路 或者 800路 公交到来");

        CompletableFuture<String> bus = CompletableFuture.supplyAsync(() -> {
            SmallTool.printTimeAndThread("700路公交正在赶来");
            SmallTool.sleepMillis(100);
            return "700路到了";
        }).applyToEither(CompletableFuture.supplyAsync(() -> {
            SmallTool.printTimeAndThread("800路公交正在赶来");
            SmallTool.sleepMillis(200);
            return "800路到了";
        }), firstComeBus -> {
            SmallTool.printTimeAndThread(firstComeBus);
            if (firstComeBus.startsWith("700")) {
                throw new RuntimeException("撞树了……");
            }
            return firstComeBus;
        }).exceptionally(e -> {
            SmallTool.printTimeAndThread(e.getMessage());
            SmallTool.printTimeAndThread("小白叫出租车");
            return "出租车 叫到了";
        });

        SmallTool.printTimeAndThread(String.format("%s,小白坐车回家", bus.join()));
    }
}
```

### 4、CompletableFuture_expand



##### thenCompose

```java
package org.ph.share._05_CompletableFuture_expand;

import org.ph.share.SmallTool;

import java.util.concurrent.CompletableFuture;

public class _02_thenCompose {
    public static void main(String[] args) {
        CompletableFuture<String> cf1 = CompletableFuture.supplyAsync(() -> {
            SmallTool.printTimeAndThread("厨师炒菜");
            SmallTool.sleepMillis(200);
            return "番茄炒蛋";
        }).thenCompose(dish -> {
            SmallTool.printTimeAndThread("服务员A 准备打饭，但是被领导叫走，打饭交接给服务员B");

            return CompletableFuture.supplyAsync(() -> {
                SmallTool.printTimeAndThread("服务员B 打饭");
                SmallTool.sleepMillis(100);
                return dish + " + 米饭";
            });
        });

        SmallTool.printTimeAndThread(cf1.join()+"好了，开饭");
    }
}
```

### 5、CompletableFuture_performance

##### thenCompose

```java
package org.ph.share._05_CompletableFuture_expand;

import org.ph.share.SmallTool;

import java.util.concurrent.CompletableFuture;

public class _02_thenCompose {
    public static void main(String[] args) {
        CompletableFuture<String> cf1 = CompletableFuture.supplyAsync(() -> {
            SmallTool.printTimeAndThread("厨师炒菜");
            SmallTool.sleepMillis(200);
            return "番茄炒蛋";
        }).thenCompose(dish -> {
            SmallTool.printTimeAndThread("服务员A 准备打饭，但是被领导叫走，打饭交接给服务员B");

            return CompletableFuture.supplyAsync(() -> {
                SmallTool.printTimeAndThread("服务员B 打饭");
                SmallTool.sleepMillis(100);
                return dish + " + 米饭";
            });
        });

        SmallTool.printTimeAndThread(cf1.join()+"好了，开饭");
    }
}
```



##### terribleCodeImprove

```java
package org.ph.share._07_CompletableFuture_performance;

import org.ph.share.SmallTool;

import java.util.ArrayList;
import java.util.concurrent.CompletableFuture;

public class _02_terribleCodeImprove {
    public static void main(String[] args) {

        SmallTool.printTimeAndThread("小白和小伙伴们 进餐厅点菜");
        long startTime = System.currentTimeMillis();
        // 点菜
        ArrayList<Dish> dishes = new ArrayList<>();
        for (int i = 1; i <= 10; i++) {
            Dish dish = new Dish("菜" + i, 1);
            dishes.add(dish);
        }
        // 做菜
        ArrayList<CompletableFuture> cfList = new ArrayList<>();
        for (Dish dish : dishes) {
            CompletableFuture<Void> cf = CompletableFuture.runAsync(() -> dish.make());
            cfList.add(cf);
        }
        // 等待所有任务执行完毕
        CompletableFuture.allOf(cfList.toArray(new CompletableFuture[cfList.size()])).join();

        SmallTool.printTimeAndThread("菜都做好了，上桌 " + (System.currentTimeMillis() - startTime));

    }
}
```



##### goodCode

```java
package org.ph.share._07_CompletableFuture_performance;

import org.ph.share.SmallTool;

import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ForkJoinPool;
import java.util.stream.IntStream;

public class _03_goodCode {
    public static void main(String[] args) {
        //-Djava.util.concurrent.ForkJoinPool.common.parallelism=8
        System.setProperty("java.util.concurrent.ForkJoinPool.common.parallelism", "12");

        SmallTool.printTimeAndThread("小白和小伙伴们 进餐厅点菜");
        long startTime = System.currentTimeMillis();

        CompletableFuture[] dishes = IntStream.rangeClosed(1, 12)
                .mapToObj(i -> new Dish("菜" + i, 1))
                .map(dish -> CompletableFuture.runAsync(dish::make))
                .toArray(size -> new CompletableFuture[size]);

        CompletableFuture.allOf(dishes).join();

        SmallTool.printTimeAndThread("菜都做好了，上桌 " + (System.currentTimeMillis() - startTime));

    }
}
```



commonPoolSize

```java
package org.ph.share._07_CompletableFuture_performance;

import java.util.concurrent.ForkJoinPool;

public class _04_commonPoolSize {
    public static void main(String[] args) {

        // Returns the number of processors available to the Java virtual machine
        System.out.println(Runtime.getRuntime().availableProcessors());
        // 查看 当前线程数
        System.out.println(ForkJoinPool.commonPool().getPoolSize());
        // 查看 最大线程数
        System.out.println(ForkJoinPool.getCommonPoolParallelism());

    }
}
```



##### customThreadPool

```java
package org.ph.share._07_CompletableFuture_performance;

import org.ph.share.SmallTool;

import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.stream.IntStream;

public class _05_customThreadPool {
    public static void main(String[] args) {
        ExecutorService threadPool = Executors.newCachedThreadPool();

        SmallTool.printTimeAndThread("小白和小伙伴们 进餐厅点菜");
        long startTime = System.currentTimeMillis();

        CompletableFuture[] dishes = IntStream.rangeClosed(1, 12)
                .mapToObj(i -> new Dish("菜" + i, 1))
                .map(dish -> CompletableFuture.runAsync(dish::makeUseCPU, threadPool))
                .toArray(CompletableFuture[]::new);

        CompletableFuture.allOf(dishes).join();

        threadPool.shutdown();

        SmallTool.printTimeAndThread("菜都做好了，上桌 " + (System.currentTimeMillis() - startTime));

    }
}
```



##### thenRunAsync_threadReuse

```java
package org.ph.share._07_CompletableFuture_performance;

import org.ph.share.SmallTool;

import java.util.concurrent.CompletableFuture;
import java.util.concurrent.SynchronousQueue;
import java.util.concurrent.ThreadPoolExecutor;
import java.util.concurrent.TimeUnit;

public class _06_thenRunAsync_threadReuse {
    public static void main(String[] args) {
        ThreadPoolExecutor executor = new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                0, TimeUnit.MILLISECONDS,
                new SynchronousQueue<Runnable>());

        CompletableFuture.runAsync(() -> SmallTool.printTimeAndThread("A"), executor)
                .thenRunAsync(() -> SmallTool.printTimeAndThread("B"), executor)
                .join();

    }
}
```





```java
package org.ph.share._07_CompletableFuture_performance;

import org.ph.share.SmallTool;

import java.util.concurrent.TimeUnit;

/**
 * 菜
 */
public class Dish {
    // 菜名
    private String name;
    // 制作时长 (秒)
    private Integer productionTime;

    public Dish(String name, Integer productionTime) {
        this.name = name;
        this.productionTime = productionTime;
    }

    // 做菜
    public void make() {
        SmallTool.sleepMillis(TimeUnit.SECONDS.toMillis(this.productionTime));
        SmallTool.printTimeAndThread(this.name + " 制作完毕，来吃我吧");
    }

    // 做菜
    public void makeUseCPU() {
        SmallTool.printTimeAndThread(this.name + " 制作完毕，来吃我吧" + compute());
    }

    /**
     * 用来模拟 1秒钟的耗时操作
     * 如果你的电脑比较强，可以增大循环次数，否则，需要减少循环次数
     */
    private static long compute() {
        long startTime = System.currentTimeMillis();
        long result = 0;
        // 只是用来模拟耗时操作，没有任何意义
        for (int i = 0; i < Integer.MAX_VALUE / 3; i++) {
            result += i * i % 3;
        }
        return System.currentTimeMillis() - startTime;
    }

    public static void main(String[] args) {
        System.out.println(compute());
    }


}
```

### 6、interrupt

```java
package org.ph.share._08_Interrupt;

import org.ph.share.SmallTool;

import java.util.concurrent.TimeUnit;

public class _01_ThreadState {
    public static void main(String[] args) {
        Thread thread = new Thread();
        System.out.println("1- " + thread.getState());
        thread.start();
        System.out.println("2- " + thread.getState());

        try {
            TimeUnit.MILLISECONDS.sleep(1);
        } catch (InterruptedException e) {
            SmallTool.printTimeAndThread("产生中断" + e.getMessage());
        }

        System.out.println("3- " + thread.getState());
    }
}
```



```java
package org.ph.share._08_Interrupt;

import org.ph.share.SmallTool;

import java.util.Random;

public class _02_TwoCarCrossBridge {
    public static void main(String[] args) {

        Thread carTwo = new Thread(() -> {
            SmallTool.printTimeAndThread("卡丁2号 准备过桥");
            SmallTool.printTimeAndThread("发现1号在过，开始等待");
            try {
                Thread.sleep(3000);
            } catch (InterruptedException e) {
                SmallTool.printTimeAndThread("卡丁2号 开始过桥");
            }
            SmallTool.printTimeAndThread("卡丁2号 过桥完毕");
        });


        Thread carOne = new Thread(() -> {
            SmallTool.printTimeAndThread("卡丁1号 开始过桥");
            int timeSpend = new Random().nextInt(500) + 1000;
            SmallTool.sleepMillis(timeSpend);
            SmallTool.printTimeAndThread("卡丁1号 过桥完毕 耗时:" + timeSpend);
//            SmallTool.printTimeAndThread("卡丁2号的状态" + carTwo.getState());
            carTwo.interrupt();
        });

        carOne.start();
        carTwo.start();

    }
}
```

##### Interrupt

```java
package org.ph.share._08_Interrupt;

import org.ph.share.SmallTool;

public class _03_Interrupt {
    public static void main(String[] args) {
        Thread carOne = new Thread(() -> {
            long startMills = System.currentTimeMillis();
            while (System.currentTimeMillis() - startMills < 3) {
//                if (Thread.currentThread().isInterrupted()) {
                if (Thread.interrupted()) {
                    SmallTool.printTimeAndThread("向左开1米");
                } else {
                    SmallTool.printTimeAndThread("往前开1米");
                }
            }
        });

        carOne.start();

        SmallTool.sleepMillis(1);
        carOne.interrupt();

    }
}
```



BeforehandInterrupt

```java
package org.ph.share._08_Interrupt;

import org.ph.share.SmallTool;

public class _04_BeforehandInterrupt {
    public static void main(String[] args) {

        Thread.currentThread().interrupt();

        try {
            SmallTool.printTimeAndThread("开始睡眠");
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            SmallTool.printTimeAndThread("发生中断");
        }

        SmallTool.printTimeAndThread("结束睡眠");

    }
}
```





```java
package org.ph.share._08_Interrupt;

import org.ph.share.SmallTool;

import java.util.concurrent.TimeUnit;

public class _05_QAndA {
    public static void main(String[] args) {
        Thread thread = new Thread(() -> {
            SmallTool.printTimeAndThread("开始睡眠");
            forceSleep(3);
            SmallTool.printTimeAndThread("结束睡眠");
        });
        thread.start();
        thread.interrupt();
    }

    @SuppressWarnings("BusyWait")
    public static void forceSleep(int second) {
        long startTime = System.currentTimeMillis();
        long sleepMills = TimeUnit.SECONDS.toMillis(second);

        while ((startTime + sleepMills) > System.currentTimeMillis()) {
            long sleepTime = startTime + sleepMills - System.currentTimeMillis();
            if (sleepTime <= 0) {
                break;
            }
            try {
                Thread.sleep(sleepTime);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
```

### 7、BlockingQueue_start

```java
package org.ph.share._09_BlockingQueue_start;

import java.util.LinkedList;
import java.util.Queue;

public class _01_Queue_demo {
    public static void main(String[] args) {
        Queue<String> queue = new LinkedList<>();
        queue.offer("one");
        queue.offer("two");
        queue.offer("three");

        System.out.println("--------开始打印--------");
        System.out.println(queue.poll());
        System.out.println(queue.poll());
        System.out.println(queue.poll());
        System.out.println("--------结束打印--------");
    }
}
```

##### OneProducer_OneConsumer

```java
package org.ph.share._09_BlockingQueue_start;

import org.ph.share.SmallTool;

import java.util.LinkedList;
import java.util.List;
import java.util.Queue;
import java.util.stream.Collectors;

public class _02_OneProducer_OneConsumer {
    public static void main(String[] args) {
        Queue<String> shaobingQueue = new LinkedList<>();

        List<String> xiaoBaiMsg = new LinkedList<>();
        List<String> roadPeopleAMsg = new LinkedList<>();

        Thread xiaoBai = new Thread(() -> {
            for (int i = 0; i < 10; i++) {
                String tmp = String.format("第%d个烧饼", i + 1);
                shaobingQueue.add(tmp);
                xiaoBaiMsg.add(String.format("%d 小白制作了 [%s]", System.currentTimeMillis(), tmp));
            }
        });

        Thread roadPeopleA = new Thread(() -> {
            for (int i = 0; i < 10; i++) {
                roadPeopleAMsg.add(String.format("%d  路人甲 买到了 [%s]", System.currentTimeMillis(), shaobingQueue.poll()));
            }
        });

        xiaoBai.start();
        roadPeopleA.start();

        try {
            xiaoBai.join();
            roadPeopleA.join();
        } catch (InterruptedException e) {
            SmallTool.printTimeAndThread("join 产生中断" + e.getMessage());
        }

        System.out.println(xiaoBaiMsg.stream().collect(Collectors.joining("\n")));
        System.out.println("--------------------------");   // 分隔线
        System.out.println(roadPeopleAMsg.stream().collect(Collectors.joining("\n")));
    }
}
```

##### OneProducer_OneConsumer_SharedVariable

```java
package org.ph.share._09_BlockingQueue_start;

import org.ph.share.SmallTool;

import java.util.LinkedList;
import java.util.List;
import java.util.Queue;
import java.util.stream.Collectors;

public class _03_OneProducer_OneConsumer_SharedVariable {
    public static void main(String[] args) {
        final int count = 1200;
        Queue<String> shaobingQueue = new LinkedList<>();

        List<String> xiaoBaiMsg = new LinkedList<>();
        List<String> roadPeopleAMsg = new LinkedList<>();

        Thread xiaoBai = new Thread(() -> {
            for (int i = 0; i < count; i++) {
                String tmp = String.format("第%d个烧饼", i+1);
                shaobingQueue.add(tmp);
                xiaoBaiMsg.add(String.format("%d 小白制作了 [%s]，当前数量 %d", System.currentTimeMillis(), tmp, shaobingQueue.size()));
            }
        });

        Thread roadPeopleA = new Thread(() -> {
            for (int i = 0; i < count; i++) {
                roadPeopleAMsg.add(String.format("%d  路人甲 买到了 [%s]", System.currentTimeMillis(), shaobingQueue.poll()));
            }
        });

        xiaoBai.start();
        roadPeopleA.start();

        try {
            xiaoBai.join();
            roadPeopleA.join();
        } catch (InterruptedException e) {
            SmallTool.printTimeAndThread("join 产生中断" + e.getMessage());
        }

        List<String> xiaoBaiMsgSub = xiaoBaiMsg.subList(xiaoBaiMsg.size() - 1, xiaoBaiMsg.size());
        System.out.println(xiaoBaiMsgSub.stream().collect(Collectors.joining("\n")));
        System.out.println("--------------------------");   // 分隔线
        List<String> roadPeopleAMsgSub = roadPeopleAMsg.subList(roadPeopleAMsg.size() - 5, roadPeopleAMsg.size());
        System.out.println(roadPeopleAMsgSub.stream().collect(Collectors.joining("\n")));
    }
}
```

##### OneProducer_MultiConsumer

```java
package org.ph.share._09_BlockingQueue_start;

import org.ph.share.SmallTool;

import java.util.LinkedList;
import java.util.List;
import java.util.Queue;
import java.util.function.Predicate;
import java.util.stream.Collectors;

public class _04_OneProducer_MultiConsumer {
    public static void main(String[] args) {
        final int count = 30;
        Queue<String> shaobingQueue = new LinkedList<>();

        List<String> xiaoBaiMsg = new LinkedList<>();
        List<String> roadPeopleAMsg = new LinkedList<>();
        List<String> roadPeopleBMsg = new LinkedList<>();

        Thread xiaoBai = new Thread(() -> {
            for (int i = 0; i < count; i++) {
                String tmp = String.format("第%d个烧饼", i + 1);
                shaobingQueue.add(tmp);
                xiaoBaiMsg.add(String.format("%d 小白制作了 [%s]", System.currentTimeMillis(), tmp));
            }
        });

        Thread roadPeopleA = new Thread(() -> {
            for (int i = 0; i < count; i++) {
                roadPeopleAMsg.add(String.format("%d  路人甲 买到了 [%s]", System.currentTimeMillis(), shaobingQueue.poll()));
            }
        });
        Thread roadPeopleB = new Thread(() -> {
            for (int i = 0; i < count; i++) {
                roadPeopleBMsg.add(String.format("%d  路人乙 买到了 [%s]", System.currentTimeMillis(), shaobingQueue.poll()));
            }
        });


        xiaoBai.start();
        roadPeopleA.start();
        roadPeopleB.start();

        try {
            xiaoBai.join();
            roadPeopleA.join();
            roadPeopleB.join();
        } catch (InterruptedException e) {
            SmallTool.printTimeAndThread("join 产生中断" + e.getMessage());
        }

        List<String> xiaoBaiMsgSub = xiaoBaiMsg.subList(xiaoBaiMsg.size() - 1, xiaoBaiMsg.size());
        System.out.println(xiaoBaiMsgSub.stream().collect(Collectors.joining("\n")));
        System.out.println("--------------------------");   // 分隔线

        Predicate<String> notContainsNull = str -> !str.contains("null");
        System.out.println(roadPeopleAMsg.stream().filter(notContainsNull).collect(Collectors.joining("\n")));
        System.out.println("--------------------------");   // 分隔线
        System.out.println(roadPeopleBMsg.stream().filter(notContainsNull).collect(Collectors.joining("\n")));
    }
}
```



| 方式         | 抛出异常 | 有返回值，不抛出异常 | 阻塞 等待 | 超时等待 |
| ------------ | -------- | -------------------- | --------- | -------- |
| 添加         | add      | offer()              | put()     | offer(,) |
| 移除         | remove   | poll()               | take()    | poll(,)  |
| 检测队首元素 | element  | peek()               |           |          |
|              |          |                      |           |          |



##### LinkedBlockingQueue_take

```java
package org.ph.share._09_BlockingQueue_start;

import org.ph.share.SmallTool;

import java.util.LinkedList;
import java.util.List;
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.LinkedBlockingQueue;
import java.util.stream.Collectors;

public class _05_LinkedBlockingQueue_take {
    public static void main(String[] args) {
        BlockingQueue<String> blockingQueue = new LinkedBlockingQueue<>(3);

        try {
            blockingQueue.take();
        } catch (InterruptedException e) {
            SmallTool.printTimeAndThread("取元素被中断");
        }
    }
}
```

##### LinkedBlockingQueue_put

```java
package org.ph.share._09_BlockingQueue_start;

import org.ph.share.SmallTool;

import java.util.LinkedList;
import java.util.List;
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.LinkedBlockingQueue;
import java.util.stream.Collectors;

public class _06_LinkedBlockingQueue_put {
    public static void main(String[] args) {
        BlockingQueue<String> blockingQueue = new LinkedBlockingQueue<>(1);

        try {
            blockingQueue.put("one");
            SmallTool.printTimeAndThread("one放进去了");

            blockingQueue.put("two");
            SmallTool.printTimeAndThread("two放进去了");

        } catch (InterruptedException e) {
            SmallTool.printTimeAndThread("取元素被中断");
        }
    }
}
```



##### LinkedBlockingQueue_Scenes

```java
package org.ph.share._09_BlockingQueue_start;

import org.ph.share.SmallTool;

import java.util.LinkedList;
import java.util.List;
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.LinkedBlockingQueue;
import java.util.stream.Collectors;

public class _07_LinkedBlockingQueue_Scenes {
    public static void main(String[] args) {
        BlockingQueue<String> shaobingQueue = new LinkedBlockingQueue<>(3);

        List<String> xiaoBaiMsg = new LinkedList<>();
        List<String> chefAMsg = new LinkedList<>();
        List<String> roadPeopleAMsg = new LinkedList<>();
        List<String> roadPeopleBMsg = new LinkedList<>();

        Thread xiaoBai = new Thread(() -> {
            for (int i = 0; i < 5; i++) {
                String shaobing = String.format("小白的 第%d个烧饼", i + 1);
                try {
                    shaobingQueue.put(shaobing);
                } catch (InterruptedException e) {
                    SmallTool.printTimeAndThread("小白被中断" + e.getMessage());
                }
                xiaoBaiMsg.add(String.format("%d 小白制作了 [%s]", System.currentTimeMillis(), shaobing));
            }
        });
        Thread chushiA = new Thread(() -> {
            for (int i = 0; i < 5; i++) {
                String shaobing = String.format("厨师A的 第%d个烧饼", i + 1);
                try {
                    shaobingQueue.put(shaobing);
                } catch (InterruptedException e) {
                    SmallTool.printTimeAndThread("厨师A被中断" + e.getMessage());
                }
                chefAMsg.add(String.format("%d 厨师A制作了 [%s]", System.currentTimeMillis(), shaobing));
            }
        });

        Thread roadPeopleA = new Thread(() -> {
            for (int i = 0; i < 5; i++) {
                String shaobing = null;
                try {
                    shaobing = shaobingQueue.take();
                } catch (InterruptedException e) {
                    SmallTool.printTimeAndThread("路人甲被中断" + e.getMessage());
                }
                roadPeopleAMsg.add(String.format("%d  路人甲 买到了 [%s]", System.currentTimeMillis(), shaobing));
            }
        });

        Thread roadPeopleB = new Thread(() -> {
            for (int i = 0; i < 5; i++) {
                String shaobing = null;
                try {
                    shaobing = shaobingQueue.take();
                } catch (InterruptedException e) {
                    SmallTool.printTimeAndThread("路人乙被中断" + e.getMessage());
                }
                roadPeopleBMsg.add(String.format("%d  路人乙 买到了 [%s]", System.currentTimeMillis(), shaobing));
            }
        });

        xiaoBai.start();
        chushiA.start();
        roadPeopleA.start();
        roadPeopleB.start();

        try {
            xiaoBai.join();
            chushiA.join();
            roadPeopleA.join();
            roadPeopleB.join();
        } catch (InterruptedException e) {
            SmallTool.printTimeAndThread("join 产生中断" + e.getMessage());
        }

        System.out.println(xiaoBaiMsg.stream().collect(Collectors.joining("\n")));
        System.out.println(chefAMsg.stream().collect(Collectors.joining("\n")));
        System.out.println("--------------------------");   // 分隔线
        System.out.println(roadPeopleAMsg.stream().collect(Collectors.joining("\n")));
        System.out.println(roadPeopleBMsg.stream().collect(Collectors.joining("\n")));
    }
}
```

### 8、BlockingQueue_basic

##### BlockingQueue_take

```java
package org.ph.share._10_BlockingQueue_basic;

import org.ph.share.SmallTool;

import java.util.concurrent.BlockingQueue;
import java.util.concurrent.LinkedBlockingQueue;

public class _01_BlockingQueue_take {
    public static void main(String[] args) {
        BlockingQueue<String> shaobingQueue = new LinkedBlockingQueue<>(3);

        Thread xiaoBai = new Thread(() -> {
            SmallTool.printTimeAndThread("小白 收拾东西，准备开张");
            SmallTool.printTimeAndThread("小白 接到电话 往家里跑");

        });

        Thread roadPeopleA = new Thread(() -> {
            SmallTool.printTimeAndThread("路人甲 来买烧饼");
            try {
                String shaobing = shaobingQueue.take();
                SmallTool.printTimeAndThread("路人甲 买到了烧饼: " + shaobing);
            } catch (InterruptedException e) {
                SmallTool.printTimeAndThread("路人甲 被中断" + e.getMessage());
            }
        });

        xiaoBai.start();
        try {
            Thread.sleep(10);   // 先等小白收拾一下，再让路人甲出场
        } catch (InterruptedException e) {
            SmallTool.printTimeAndThread("主线程 被中断" + e.getMessage());
        }
        roadPeopleA.start();
    }
}
```



##### BlockingQueue_poll

```java
package org.ph.share._10_BlockingQueue_basic;

import org.ph.share.SmallTool;

import java.util.concurrent.BlockingQueue;
import java.util.concurrent.LinkedBlockingQueue;
import java.util.concurrent.TimeUnit;

public class _02_BlockingQueue_poll {
    public static void main(String[] args) {
        BlockingQueue<String> shaobingQueue = new LinkedBlockingQueue<>(3);

        Thread xiaoBai = new Thread(() -> {
            SmallTool.printTimeAndThread("小白 收拾东西，准备开张");
            SmallTool.printTimeAndThread("小白 接到电话 往家里跑");

        });

        Thread roadPeopleA = new Thread(() -> {
            SmallTool.printTimeAndThread("路人甲 来买烧饼");
            try {
                String shaobing = shaobingQueue.poll(2, TimeUnit.SECONDS);
                SmallTool.printTimeAndThread("路人甲 买到了烧饼: " + shaobing);
            } catch (InterruptedException e) {
                SmallTool.printTimeAndThread("路人甲 被中断" + e.getMessage());
            }
        });

        xiaoBai.start();
        try {
            Thread.sleep(10);   // 先等小白收拾一下，再让路人甲出场
        } catch (InterruptedException e) {
            SmallTool.printTimeAndThread("主线程 被中断" + e.getMessage());
        }
        roadPeopleA.start();
    }
}
```

##### BlockingQueue_poll_return

```java
package org.ph.share._10_BlockingQueue_basic;

import org.ph.share.SmallTool;

import java.util.concurrent.BlockingQueue;
import java.util.concurrent.LinkedBlockingQueue;
import java.util.concurrent.TimeUnit;

public class _03_BlockingQueue_poll_return {
    public static void main(String[] args) {
        BlockingQueue<String> shaobingQueue = new LinkedBlockingQueue<>(3);

        Thread xiaoBai = new Thread(() -> {
            SmallTool.printTimeAndThread("小白 收拾东西，准备开张");
            SmallTool.printTimeAndThread("小白 接到电话 往家里跑");
        });

        Thread roadPeopleA = new Thread(() -> {
            SmallTool.printTimeAndThread("路人甲 来买烧饼");
            try {
                String shaobing = shaobingQueue.poll(2, TimeUnit.SECONDS);
                if (shaobing == null) {
                    SmallTool.printTimeAndThread("路人甲 没有买到烧饼");
                } else {
                    SmallTool.printTimeAndThread("路人甲 买到了烧饼: " + shaobing);
                }
            } catch (InterruptedException e) {
                SmallTool.printTimeAndThread("路人甲 被中断" + e.getMessage());
            }
        });

        xiaoBai.start();
        try {
            Thread.sleep(10);   // 先等小白收拾一下，再让路人甲出场
        } catch (InterruptedException e) {
            SmallTool.printTimeAndThread("主线程 被中断" + e.getMessage());
        }
        roadPeopleA.start();
    }
}
```



##### BlockingQueue_poll_0

```java
package org.ph.share._10_BlockingQueue_basic;

import org.ph.share.SmallTool;

import java.util.concurrent.BlockingQueue;
import java.util.concurrent.LinkedBlockingQueue;
import java.util.concurrent.TimeUnit;

public class _04_BlockingQueue_poll_0 {
    public static void main(String[] args) {
        BlockingQueue<String> shaobingQueue = new LinkedBlockingQueue<>(3);

        Thread xiaoBai = new Thread(() -> {
            SmallTool.printTimeAndThread("小白 收拾东西，准备开张");
            SmallTool.printTimeAndThread("小白 接到电话 往家里跑");
        });

        Thread roadPeopleA = new Thread(() -> {
            SmallTool.printTimeAndThread("路人甲 来买烧饼");
            String shaobing = shaobingQueue.poll();
            if (shaobing == null) {
                SmallTool.printTimeAndThread("路人甲 没有买到烧饼");
            } else {
                SmallTool.printTimeAndThread("路人甲 买到了烧饼: " + shaobing);
            }
        });

        xiaoBai.start();
        try {
            Thread.sleep(10);   // 先等小白收拾一下，再让路人甲出场
        } catch (InterruptedException e) {
            SmallTool.printTimeAndThread("主线程 被中断" + e.getMessage());
        }
        roadPeopleA.start();
    }
}
```

### 9、BlockingQueue_who_else_1

##### LinkedBlockingQueue_unfair

```java
package org.ph.share._11_BlockingQueue_who_else_1;

import org.ph.share.SmallTool;

import java.util.concurrent.BlockingQueue;
import java.util.concurrent.LinkedBlockingQueue;
import java.util.concurrent.TimeUnit;

public class _01_LinkedBlockingQueue_unfair {

    public static void main(String[] args) {
        BlockingQueue<String> shaobingQueue = new LinkedBlockingQueue<>(3);

        new Thread(() -> {
            try {
                SmallTool.printTimeAndThread("来买烧饼");
                String shaobing = shaobingQueue.poll(1, TimeUnit.SECONDS);
                String tag = shaobing == null ? "再见, 以后不来了" : "买到烧饼了";
                SmallTool.printTimeAndThread(tag);
            } catch (InterruptedException e) {
                SmallTool.printTimeAndThread("被中断" + e.getMessage());
            }
        }, "张三").start();

        SmallTool.sleepMillis(100);     // 模拟张三先到

        new Thread(() -> {
            try {
                SmallTool.printTimeAndThread("来买烧饼");
                String shaobing = shaobingQueue.poll(1, TimeUnit.SECONDS);
                String tag = shaobing == null ? "草, 掀摊子! " : "买到烧饼了";
                SmallTool.printTimeAndThread(tag);
            } catch (InterruptedException e) {
                SmallTool.printTimeAndThread("被中断" + e.getMessage());
            }
        }, "李四").start();

        shaobingQueue.offer("芝麻烧饼");
    }
}
```

##### ArrayBlockingQueue_fair

```java
package org.ph.share._11_BlockingQueue_who_else_1;

import org.ph.share.SmallTool;

import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.TimeUnit;

public class _02_ArrayBlockingQueue_fair {
    private static BlockingQueue<String> shaobingQueue = new ArrayBlockingQueue<>(3,true);

    public static void main(String[] args) {
        new Thread(() -> {
            try {
                SmallTool.printTimeAndThread("来买烧饼");
                String shaobing = shaobingQueue.poll(1, TimeUnit.SECONDS);
                String tag = shaobing == null ? "再见, 以后不来了" : "买到烧饼了";
                SmallTool.printTimeAndThread(tag);
            } catch (InterruptedException e) {
                SmallTool.printTimeAndThread("被中断" + e.getMessage());
            }
        }, "张三").start();

        SmallTool.sleepMillis(100);     // 模拟张三先到

        new Thread(() -> {
            try {
                SmallTool.printTimeAndThread("来买烧饼");
                String shaobing = shaobingQueue.poll(1, TimeUnit.SECONDS);
                String tag = shaobing == null ? "草, 不敢掀昊天宗的摊子! " : "买到烧饼了";
                SmallTool.printTimeAndThread(tag);
            } catch (InterruptedException e) {
                SmallTool.printTimeAndThread("被中断" + e.getMessage());
            }
        }, "李四").start();

        shaobingQueue.offer("芝麻烧饼");
    }
}
```



```java
package org.ph.share._11_BlockingQueue_who_else_1;

import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.BlockingQueue;

public class _03_ArrayBlockingQueue_0 {

    public static void main(String[] args) {
        BlockingQueue<String> shaobingQueue = new ArrayBlockingQueue<>(0,true);
    }
}
```



##### SynchronousQueue

```java
package org.ph.share._11_BlockingQueue_who_else_1;

import org.ph.share.SmallTool;

import java.util.concurrent.BlockingQueue;
import java.util.concurrent.SynchronousQueue;

public class _04_SynchronousQueue {
    public static void main(String[] args) {
        BlockingQueue<String> shaobingQueue = new SynchronousQueue<>();

        new Thread(() -> {
            try {
                SmallTool.printTimeAndThread("准备开始做烧饼");
                shaobingQueue.put("芝麻烧饼1号");
                SmallTool.printTimeAndThread("卖出去了第1个烧饼");

                SmallTool.sleepMillis(2000);    // 休息两秒钟 再继续做

                shaobingQueue.put("芝麻烧饼2号");
                SmallTool.printTimeAndThread("卖出去了第2个烧饼");
            } catch (InterruptedException e) {
                SmallTool.printTimeAndThread("被中断" + e.getMessage());
            }
        }, "小白").start();

        new Thread(() -> {
            try {
                SmallTool.sleepMillis(1000);    // 还不饿，先等一秒

                SmallTool.printTimeAndThread("买到了" + shaobingQueue.take());
                SmallTool.printTimeAndThread("瞬间吃完，继续买");
                SmallTool.printTimeAndThread("买到了" + shaobingQueue.take());
            } catch (InterruptedException e) {
                SmallTool.printTimeAndThread("被中断" + e.getMessage());
            }
        }, "张三").start();

    }
}
```

### 10、BlockingQueue_who_else_2

```
package org.ph.share._12_BlockingQueue_who_else_2;

import java.util.concurrent.BlockingQueue;
import java.util.concurrent.PriorityBlockingQueue;

public class _01_PriorityBlockingQueue_start {
    public static void main(String[] args) {
        BlockingQueue<Integer> blockingQueue = new PriorityBlockingQueue<>();

        blockingQueue.offer(3);
        blockingQueue.offer(1);
        blockingQueue.offer(2);

        while (!blockingQueue.isEmpty()) {
            System.out.println(blockingQueue.poll());
        }
    }
}
```

```
package org.ph.share._12_BlockingQueue_who_else_2;

import java.util.concurrent.BlockingQueue;
import java.util.concurrent.PriorityBlockingQueue;

public class _02_PriorityBlockingQueue_Pancake {

    public static void main(String[] args) {
        BlockingQueue<Pancake> blockingQueue = new PriorityBlockingQueue<>();

        blockingQueue.offer(new Pancake(0));
    }

    private record Pancake(
            /**
             * 美味程度
             *  0 --> 好吃
             *  1 --> 一般
             *  2 --> 难吃
             */
            Integer flavor
    ) {
    }
}
```

```
package org.ph.share._12_BlockingQueue_who_else_2;

import org.ph.share.SmallTool;

import java.util.concurrent.BlockingQueue;
import java.util.concurrent.PriorityBlockingQueue;

public class _03_PriorityBlockingQueue_Pancake_Comparable {

    public static void main(String[] args) {
        BlockingQueue<Pancake> blockingQueue = new PriorityBlockingQueue<>();

        Thread xiaobai = new Thread(() -> {
            blockingQueue.offer(new Pancake(2));
            SmallTool.printTimeAndThread("做好第1个烧饼");

            blockingQueue.offer(new Pancake(0));
            SmallTool.printTimeAndThread("做好第2个烧饼");

            blockingQueue.offer(new Pancake(1));
            SmallTool.printTimeAndThread("做好第3个烧饼");
        }, "小白");

        xiaobai.start();
        try {
            xiaobai.join();     // 让小白顺利做完 3个烧饼
        } catch (InterruptedException e) {
            SmallTool.printTimeAndThread("被中断" + e.getMessage());
        }

        new Thread(() -> {
            SmallTool.printTimeAndThread("买到烧饼" + blockingQueue.poll());
        }, "张三").start();

    }

    private record Pancake(
            /**
             * 美味程度
             *  0 --> 好吃
             *  1 --> 一般
             *  2 --> 难吃
             */
            Integer flavor
    ) implements Comparable<Pancake> {
        @Override
        public int compareTo(Pancake o) {
            return this.flavor.compareTo(o.flavor);
        }
    }
}
```

```java
package org.ph.share._12_BlockingQueue_who_else_2;

import org.ph.share.SmallTool;

import java.util.Comparator;
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.PriorityBlockingQueue;

public class _04_PriorityBlockingQueue_Pancake_Comparable_enum {

    public static void main(String[] args) {
//        BlockingQueue<Pancake> blockingQueue = new PriorityBlockingQueue<>(
//                3,
//                (o1, o2) -> o1.flavor.compareTo(o2.flavor)
//        );
        BlockingQueue<Pancake> blockingQueue = new PriorityBlockingQueue<>(
                3,
                Comparator.comparing(Pancake::flavor)
        );

        Thread xiaobai = new Thread(() -> {
            blockingQueue.offer(new Pancake(Flavor.UNPALATABLE));
            SmallTool.printTimeAndThread("做好第1个烧饼");

            blockingQueue.offer(new Pancake(Flavor.DELICIOUS));
            SmallTool.printTimeAndThread("做好第2个烧饼");

            blockingQueue.offer(new Pancake(Flavor.EDIBLE));
            SmallTool.printTimeAndThread("做好第3个烧饼");
        }, "小白");

        xiaobai.start();
        try {
            xiaobai.join();     // 让小白顺利做完 3个烧饼
        } catch (InterruptedException e) {
            SmallTool.printTimeAndThread("被中断" + e.getMessage());
        }

        new Thread(() -> {
            SmallTool.printTimeAndThread("买到烧饼" + blockingQueue.poll());
        }, "张三").start();

    }

    private record Pancake(Flavor flavor) {
    }

    private enum Flavor {
        DELICIOUS, // 好吃
        EDIBLE,     // 一般(能下口)
        UNPALATABLE // 难吃
    }

}
```