---
layout: single
title: DelayQueue
date: 2015-01-24 10:36:30
category: java
---

DelayQueue 是一个支持延时获取元素的使用优先级队列实现的无边界阻塞队列。

所以可以使用在缓存系统的设计上，或者定时任务调度上等。

下面使用一个列子掩饰使用DelayQueue做一个定时任务调度。

首先是一个实现了Delayed接口的延时任务，应为DelayQueue的元素必须要是要实现了Delayed的接口的。

<!--more-->

```java
class DelayedTask implements Runnable, Delayed {

    private static int counter = 0;
    protected static List<DelayedTask> sequence = new ArrayList<DelayedTask>();
    private final int id = counter++;
    private final int delayTime;      //需要延迟的时间
    private final long triggerTime;   //实际的执行时间
    public DelayedTask(int delayInMillis) {
        delayTime = delayInMillis;
        triggerTime = System.nanoTime() + NANOSECONDS.convert(delayTime, MILLISECONDS);
        System.out.println("triggerTime:"+triggerTime+"    delayTime:"+delayTime);
        sequence.add(this);
    }

    @Override
    public int compareTo(Delayed o) {
        DelayedTask that = (DelayedTask)o;
        if (triggerTime < that.triggerTime) return -1;
        if (triggerTime > that.triggerTime) return 1;
        return 0;
    }

    @Override
    public long getDelay(TimeUnit unit) {
        return unit.convert(triggerTime - System.nanoTime(), NANOSECONDS);
    }

    @Override
    public void run() {
        System.out.println(this + " ");
        System.out.println("triggerTime:"+triggerTime+"    delayTime:"+delayTime);
    }

    @Override
    public String toString() {
        return String.format("[%1$-4d]", delayTime) + " Task " + id;
    }

    //延迟队列如果要在执行玩最后一个元素的时候退出的话，最后一个任务需要结束这个队列
    public static class EndSentinel extends DelayedTask {
        private ExecutorService exec;
        public EndSentinel(int delay, ExecutorService exec) {
            super(delay);
            this.exec = exec;
        }
        @Override
        public void run() {
            System.out.println(this + " calling shutDownNow()");
            exec.shutdownNow();
        }
    }
}
```

DelayedTaskConsumer队列的实际消费者

```java
class DelayedTaskConsumer implements Runnable {
    private DelayQueue<DelayedTask> tasks;
    public DelayedTaskConsumer(DelayQueue<DelayedTask> tasks) {
        this.tasks = tasks;
    }
    @Override
    public void run() {
        try {
            while(!Thread.interrupted()) {
                tasks.take().run();//用while无线循环延迟队列如果有获得可以执行的任务，就进行执行。
            }
        } catch (InterruptedException e) {
            // TODO: handle exception
        }
        System.out.println("Finished DelaytedTaskConsumer.");
    }
}
```

最后是测试类

```java
public class DelayQueueDemo {
    public static void main(String[] args) {
        int maxDelayTime = 5000;/
        Random random = new Random();
        ExecutorService exec = Executors.newCachedThreadPool();
        DelayQueue<DelayedTask> queue = new DelayQueue<DelayedTask>();
        //填充10个休眠时间随机的任务
        for (int i = 0; i < 10; i++) {
            queue.put(new DelayedTask(random.nextInt(maxDelayTime)));
        }
        //设置结束的时候的特殊任务，这里不一定是要写成内部类。
        queue.add(new DelayedTask.EndSentinel(maxDelayTime, exec));
        exec.execute(new DelayedTaskConsumer(queue));
    }
}
```

Demo中使用了一个日期工具类TimeUnit，这个类可以方便的进行时间转化，不懂的可以Google之。
