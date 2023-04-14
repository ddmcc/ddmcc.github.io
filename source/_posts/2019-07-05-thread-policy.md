---
layout: post
title:  "线程池拒绝策略"
date:   2019-07-05 11:21:00
categories: 并发编程
tags: ThreadPool
author: ddmcc
---

* content
{:toc}


## 线程池四种拒绝策略

在线程池配置文件中配置RejectedExecutionHandler属性，可以为线程池指定拒绝策略。

在ThreadPoolExecutor类中，有四个内部类为JDK自带的四种拒绝策略。CallerRunsPolicy，AbortPolicy，DiscardPolicy，DiscardOldestPolicy。
也可以自定义拒绝策略，实现RejectedExecutionHandler的rejectedExecution方法。



#### CallerRunsPolicy

    public static class CallerRunsPolicy implements RejectedExecutionHandler {
        /**
         * Creates a {@code CallerRunsPolicy}.
         */
        public CallerRunsPolicy() { }

        /**
         * Executes task r in the caller's thread, unless the executor
         * has been shut down, in which case the task is discarded.
         *
         * @param r the runnable task requested to be executed
         * @param e the executor attempting to execute this task
         */
        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
            if (!e.isShutdown()) {
                r.run();
            }
        }
    }

从源码上可以看出直接在 execute 方法中同步调用被拒绝的任务。


#### AbortPolicy

    public static class AbortPolicy implements RejectedExecutionHandler {
        /**
         * Creates an {@code AbortPolicy}.
         */
        public AbortPolicy() { }

        /**
         * Always throws RejectedExecutionException.
         *
         * @param r the runnable task requested to be executed
         * @param e the executor attempting to execute this task
         * @throws RejectedExecutionException always
         */
        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
            throw new RejectedExecutionException("Task " + r.toString() +
                                                 " rejected from " +
                                                 e.toString());
        }
    }


直接抛出异常RejectedExecutionException，丢弃任务。（jdk默认策略，队列满并线程满时直接拒绝添加新任务，并抛出异常）


#### DiscardPolicy

    public static class DiscardPolicy implements RejectedExecutionHandler {
        /**
         * Creates a {@code DiscardPolicy}.
         */
        public DiscardPolicy() { }

        /**
         * Does nothing, which has the effect of discarding task r.
         *
         * @param r the runnable task requested to be executed
         * @param e the executor attempting to execute this task
         */
        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        }
    }

可以看出这种策略什么都没做，直接丢弃任务。


#### DiscardOldestPolicy

    public static class DiscardOldestPolicy implements RejectedExecutionHandler {
        /**
         * Creates a {@code DiscardOldestPolicy} for the given executor.
         */
        public DiscardOldestPolicy() { }

        /**
         * Obtains and ignores the next task that the executor
         * would otherwise execute, if one is immediately available,
         * and then retries execution of task r, unless the executor
         * is shut down, in which case task r is instead discarded.
         *
         * @param r the runnable task requested to be executed
         * @param e the executor attempting to execute this task
         */
        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
            if (!e.isShutdown()) {
                e.getQueue().poll();
                e.execute(r);
            }
        }
    }

获取当前线程池的队列并调用poll()方法删除队列头的任务（即下一个执行的任务），然后添加该任务。

