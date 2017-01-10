---
published: true
layout: post
title: Scala Akka MapReduce 示例
category: Scala
tags: Akka MapReduce
time: 2017.01.10 10:40:00
keywords: 
description: 经典的mapreduce示例, 统计单词频次

---

## 直接上代码, 都有注释, 后面有贴运行结果

    package com.kute.akka.mapreduce


    /**
     *
     * akka mapreduce 示例
     *
     * 目的:
     *  输入3个字符串(代表要执行的三个子任务), 最后将任务结果汇总, 统计 每个单词出现的次数
     *
     * 过程:
     *  1. 对每条消息(子任务) 进行单词分隔初始化计数为1(中间可以作其他操作,例如 过滤不需要统计的单词等 map操作)
     *  2. 对每条消息(字符串) 进行单词频次累加(每个字符串中重复出现的单词次数累加-reduce)
     *  3. 上面两步 是计算出了单个任务中单词的频次, 接下来再把 三个任务进行汇总-reduce
     *  4. 结束, 这就是 map-reduce(总任务划分为子任务, 子任务 进行映射计算等, 最后汇总结果返回)
     *
     * @author kute
     * @time 2016-01-23
     */
    
    import java.util.StringTokenizer
    import akka.actor._
    import scala.collection.immutable._
    
    
    class Word(val word: String, val count: Int)
    case class Result()
    class MapData(val dataList: List[Word])
    class ReduceData(val reduceDataMap: Map[String, Int])
    
    object MapReduceApplication extends App {
    
      val _system = ActorSystem("MapReduceApp")
      val master = _system.actorOf(Props[MasterActor], name = "master")
    
      println("Scala begin!")
    
      // 发送 三条消息(每条代表一个字子任务)
      master ! "The quick brown fox tried to jump over the lazy dog and fell on the dog"
      master ! "Dog is man's best friend"
      master ! "Dog and Fox belong to the same family"
    
      Thread.sleep(500)
      master ! new Result
    
      Thread.sleep(500)
      _system.shutdown
      println("Scala done!")
    }
    
    class MasterActor extends Actor with ActorLogging{
    
      val aggregateActor: ActorRef = context.actorOf(Props[AggregateActor], name = "aggregate")
      val reduceActor: ActorRef = context.actorOf(Props(classOf[ReduceActor], aggregateActor), name = "reduce")
      val mapActor: ActorRef = context.actorOf(Props(classOf[MapActor], reduceActor), name = "map")
    
      var tasksCount: Int = _
    
      def receive: Receive = {
        case message: String => {
          tasksCount = tasksCount + 1
          log.info(s"map-task-${tasksCount} begin exec msg:[${message}}]")
          mapActor ! (message, tasksCount)
        }
        case message: Result => {
          log.info("print result....")
          aggregateActor ! message
        }
      }
    }
    
    /**
     * 过滤,映射
     * @param reduceActor
     */
    class MapActor(reduceActor: ActorRef) extends Actor with ActorLogging{
    
      val STOP_WORDS_LIST = List("a", "am", "an", "and", "are", "as", "at", "be",
        "do", "go", "if", "in", "is", "it", "of", "on", "the", "to")
    
      def receive: Receive = {
        case (message: String, taskCount: Int) => {
          log.info(s"map-task-${taskCount} begin map....")
          reduceActor ! (evaluateExpression(message), taskCount)
        }
      }
    
      /**
       * 将 每条消息 按单词分隔,初始化计数为1
       * @param line
       * @return
       */
      def evaluateExpression(line: String): MapData = {
        var dataList = List[Word]()
        val parser: StringTokenizer = new StringTokenizer(line)
        while (parser.hasMoreTokens()) {
          val word: String = parser.nextToken().toLowerCase()
          if (!STOP_WORDS_LIST.contains(word)) {
            dataList = new Word(word, 1) :: dataList
          }
        }
        return new MapData(dataList)
      }
    }
    
    /**
     * 单map-task计算
     * 将单条消息中的单词计数进行汇总
     * @param aggregateActor
     */
    class ReduceActor(aggregateActor: ActorRef) extends Actor with ActorLogging{
    
      def receive: Receive = {
        case (message: MapData, taskCount: Int) => {
          log.info(s"map-task-${taskCount} begin reduce....")
          aggregateActor ! (reduce(message.dataList), taskCount)
        }
      }
    
      def reduce(dataList: List[Word]): ReduceData = {
        var reducedMap = new HashMap[String, Int]
        for (wc: Word <- dataList) {
          val word: String = wc.word
          if (reducedMap.contains(word)) {
            val count: Int = reducedMap.get(word).get + wc.count
            reducedMap += word -> count
          } else {
            reducedMap += word -> wc.count
          }
        }
        return new ReduceData(reducedMap)
      }
    }
    
    /**
     * 汇总所有的map-task
     */
    class AggregateActor extends Actor with ActorLogging{
    
      var finalReducedMap = new HashMap[String, Int]
    
      def receive: Receive = {
        case (message: ReduceData, taskCount: Int) => {
          log.info(s"receive reduce map-task-${taskCount} and to cal-sum")
          if(taskCount == 3) {
            log.info("tasks over,begin cal sum")
          }
          aggregateInMemoryReduce(message.reduceDataMap)
        }
        case message: Result =>
          println(finalReducedMap.toString())
      }
    
      def aggregateInMemoryReduce(reducedMap: Map[String, Int]) {
        var count: Int = 0
        reducedMap.foreach((entry: (String, Int)) =>
          if (finalReducedMap.contains(entry._1)) {
            count = entry._2 + finalReducedMap.get(entry._1).getOrElse(0)
            finalReducedMap += entry._1 -> count
          } else
            finalReducedMap += entry._1 -> entry._2)
      }
    }
    
## 运行结果

    Scala begin!
    [INFO] [01/10/2017 23:10:44.402] [MapReduceApp-akka.actor.default-dispatcher-2] [akka://MapReduceApp/user/master] map-task-1 begin exec msg:[The quick brown fox tried to jump over the lazy dog and fell on the dog}]
    [INFO] [01/10/2017 23:10:44.403] [MapReduceApp-akka.actor.default-dispatcher-2] [akka://MapReduceApp/user/master] map-task-2 begin exec msg:[Dog is man's best friend}]
    [INFO] [01/10/2017 23:10:44.403] [MapReduceApp-akka.actor.default-dispatcher-2] [akka://MapReduceApp/user/master] map-task-3 begin exec msg:[Dog and Fox belong to the same family}]
    [INFO] [01/10/2017 23:10:44.403] [MapReduceApp-akka.actor.default-dispatcher-2] [akka://MapReduceApp/user/master/map] map-task-1 begin map....
    [INFO] [01/10/2017 23:10:44.404] [MapReduceApp-akka.actor.default-dispatcher-2] [akka://MapReduceApp/user/master/map] map-task-2 begin map....
    [INFO] [01/10/2017 23:10:44.404] [MapReduceApp-akka.actor.default-dispatcher-2] [akka://MapReduceApp/user/master/map] map-task-3 begin map....
    [INFO] [01/10/2017 23:10:44.405] [MapReduceApp-akka.actor.default-dispatcher-3] [akka://MapReduceApp/user/master/reduce] map-task-1 begin reduce....
    [INFO] [01/10/2017 23:10:44.407] [MapReduceApp-akka.actor.default-dispatcher-3] [akka://MapReduceApp/user/master/reduce] map-task-2 begin reduce....
    [INFO] [01/10/2017 23:10:44.407] [MapReduceApp-akka.actor.default-dispatcher-3] [akka://MapReduceApp/user/master/reduce] map-task-3 begin reduce....
    [INFO] [01/10/2017 23:10:44.407] [MapReduceApp-akka.actor.default-dispatcher-2] [akka://MapReduceApp/user/master/aggregate] receive reduce map-task-1 and to cal-sum
    [INFO] [01/10/2017 23:10:44.408] [MapReduceApp-akka.actor.default-dispatcher-2] [akka://MapReduceApp/user/master/aggregate] receive reduce map-task-2 and to cal-sum
    [INFO] [01/10/2017 23:10:44.409] [MapReduceApp-akka.actor.default-dispatcher-2] [akka://MapReduceApp/user/master/aggregate] receive reduce map-task-3 and to cal-sum
    [INFO] [01/10/2017 23:10:44.409] [MapReduceApp-akka.actor.default-dispatcher-2] [akka://MapReduceApp/user/master/aggregate] tasks over,begin cal sum
    [INFO] [01/10/2017 23:10:44.904] [MapReduceApp-akka.actor.default-dispatcher-2] [akka://MapReduceApp/user/master] print result....
    Map(lazy -> 1, jump -> 1, best -> 1, fell -> 1, friend -> 1, dog -> 4, belong -> 1, man's -> 1, over -> 1, same -> 1, brown -> 1, tried -> 1, quick -> 1, family -> 1, fox -> 2)
    Scala done!
    
    Process finished with exit code 0

