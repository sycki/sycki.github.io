# 分类算法 - 朴素贝叶斯

朴素贝叶斯是种简单而有效的分类算法，被应用在很多二元分类器中，那什么叫二元分类？也就是非A即B，假设我现在用朴素贝叶斯算法写一个分类器，然后输入一封邮件，它可以根据特征库来判断这封邮件是不是垃圾邮件。当然，它还可以用来处理多元分类的问题，比如：文章分类、拼写纠正等等。 

## 基本原理

在网上已经有很多关于朴素贝叶斯的介绍了，其中不乏一些讲得深刻而又通俗易懂的文章，本文不准备长篇大论探讨它的数学原理及深层含义，不过基本原理还是得讲一下，先说一下条件概率模型，以下摘抄自wiki： 

> 公式：p(C|F1,...,Fn)
> 解释：独立的类别变量C有若干类别，条件依赖于若干特征变量。 

意思是说，在F1,F2,F3...这些特征发生的情况下，类别C的发生概率，很容易理解。 那么，如果我事先知道类别C发生的情况下，F1,F2,F3...出现的概率，也就是p(F1,...,Fn|C)，那么也可以反过来求出p(C|F1,...,Fn)，这其实就是它基本原理了，以下是贝叶斯公式：
```
p(类别1|特征1,特征2,特征3...) = p(特征1,特征2,特征3...|类别1) * p(类别1) / p(特征1,特征2,特征3...)
```
解释： p(类别1|特征1,特征2,特征3...) 表示在特征1、特征2、特征3...发生的情况下，类别1发生的概率。 p(特征1,特征2,特征3...|类别1) 表示在类别1发生的情况下，特征1、特征2、特征3...发生的概率。 p(类别1) 表示类别1在所有类别中出的概率 p(特征1,特征2,特征3...) 表示特征1,特征2,特征3...在特征库中出现的概率 重复使用链式法则后，最终可以将公式转换为这样：
```
p(类别1|特征1,特征2,特征3...) = p(特征1|类别1) * p(特征2|类别1) * p(特征3|类别1) ...
```
这就是朴素贝叶斯公式了。 

## 实现邮件分类器

之前在研究Spark的MLLib时第一次接触到朴素贝叶斯算法（实际上已经是2016年的事了），并动手实现了一个简单的垃圾邮件识别程序，今天重新整理一下，我会在代码中一步一步讲明原理，先说一下实现邮件分类器的步骤： 

  1. 收集数据，准备一些正常的邮件和垃圾邮件，将它们放在两个不同的目录中
  2. 特征提取，要让程序判断一封邮件是不是垃圾邮件，就需要告诉程序，什么是正常邮件，什么是垃圾邮件，这一步叫特征提取，也可以叫做训练集，提取的方法很简单，就是分别计算出正常邮件中与垃圾邮件中所有单词出现的数次及概率，这样我们就有了两个训练集
  3. 识别邮件，这时需要写一个函数用来接收并识别给定的邮件，其原理也很简单，先将指定的邮件中的所有单词提取出来，然后计算这些单词在两个训练集出现的概率之积即可，这样就得到两个数字，从哪个训练集中计算出的数字大，那这封邮件就属于哪种类型。
  4. 参数调整，当程序基本实现后，需要对各步骤中的参数进行调整，比如在特征提取时，我去掉了只出现一次或两次的单词，又比如，在识别新邮件时，如果这封邮件中出现了训练集中没有的单词时，应该给定它一个默认的概率值
下面给出一个实际的例子，在本例中我大概用了三千封历史邮件，分别放在了两个目录中，例子中的数据实在是想不起来在哪下载的，所以请同学们自行收集吧。

首先是准备数据：
```
package com.algorithm.bayes
 
import org.apache.spark.SparkConf
import org.apache.spark.SparkContext
import scala.io.Source
import java.io.File
import scala.sys.process.ProcessBuilder.Source
import scala.collection.mutable.ArrayBuffer
import scala.collection.mutable.HashMap
 
/**
 * 利用贝叶斯实现的邮件分类器，提交给Spark执行
 */
object Classification {
   
  def main(args: Array[String]): Unit = {
     
    //初始化Spark
    val conf = new SparkConf().setAppName(this.getClass.getName)
    val sc = new SparkContext(conf)
     
    //该目录内均为正常邮件
    var easy = "/home/kxdmmr/src/ml-data/Email-data/easy_ham"
     
    //该目录内均为垃圾邮件
    var spam = "/home/kxdmmr/src/ml-data/Email-data/spam"
     
...
```
然后就是创建两个特征库，这个是重中之重：
```
//将所有正常邮件的文本都提取出来
var easyArr = sc.parallelize(getEmailText(easy))
//将所有垃圾邮件的文件都提取出来
var spamArr = sc.parallelize(getEmailText(spam))
 
//正常邮件中所有的单词,不去重
var easySplit = easyArr.flatMap { x => x.split("[^a-zA-Z]+") }
var easyLen = easySplit.count().toDouble
 
//垃圾邮件中所有的单词,不去重
var spamSplit = spamArr.flatMap { x => x.split("[^a-zA-Z]+") }
var spamLen = spamSplit.count().toDouble
 
//正常邮件中每个单词占所有单词的百分比及出现次数
//结果数据的格式为：HashMap[单词: String,(出现次数: Double,百分比: Double)]
var easyWordProb = new HashMap[String,(Double,Double)]()
easySplit.map { x => (x,1.0) }.reduceByKey(_+_).filter(x => x._2 > 2).collect().foreach(f => easyWordProb.put(f._1,(f._2,f._2/easyLen)))
 
//垃圾邮件中每个单词占所有单词的百分比及出现次数
//结果数据的格式为：HashMap[单词: String,(出现次数: Double,百分比: Double)]
var spamWordProb = new HashMap[String,(Double,Double)]()
spamSplit.map { x => (x,1.0) }.reduceByKey(_+_).filter(x => x._2 > 2).collect().foreach(f => spamWordProb.put(f._1,(f._2,f._2/spamLen)))
```
上面用到了一个函数，定义如下：
```
/**
 * 处理指定目录内的500封邮件，提取邮件内出现的所有单词
 * 并封结果封装成一个数组，将其返回
 */
def getEmailText(path: String):ArrayBuffer[String] ={
  var count = 0;
  var textArr = new ArrayBuffer[String]()
  var files = new File(path).listFiles()
  var temp = new StringBuilder()
  var isAdd = false
  for(f <- files){
    if(count ==500)
      return textArr;
    count += 1
    isAdd = false
    temp.delete(0, temp.length)
    var lines = Source.fromFile(f)
    try{
      for(line <- lines.getLines()){
        if(line.equals(""))
            isAdd = true
        else
          temp.append(line+",")
      }
    }catch{
      case e:Exception => println("************** "+e.toString())
    }
    lines.close()
    textArr.append(temp.toString)
  }
  textArr
}
```
最后写一个函数，用来处理指定的邮件：
```
    /**
     * 识别指定邮件的类型，将结果打印到控制台
     */
    def getType(file: File){
      var textArr = new ArrayBuffer[String]()
      var isAdd = false
      try{
        for(x <- Source.fromFile(file).getLines()){
          if(x.equals(""))
            isAdd = true
          else
            textArr.append(x+",")
        }
      }catch{case e:Exception => None}
      var result = textArr.flatMap { x => x.split("[^a-zA-Z]+") }
       
      var a = 1.0
      var b = 1.0
       
      for(i <- result ){
        var easyPro = easyWordProb.getOrElse(i,(1.0,0.0001))._2
        var spamPro = spamWordProb.getOrElse(i,(1.0,0.0001))._2
        a = easyPro * a
        b = spamPro * b
      }
       
      if(a > b)
        println("此邮件为正常邮件 "+a+" "+b)
      else
        println("此邮件为垃圾邮件 "+a+" "+b)
       
    }
  }
}
```

## 结语
以上就是实现一个邮件分类器的所有步骤，单独解释贝叶斯可能比较难懂，但实际用的时候发现它的理原极其简单而又直接，这也许就是它名字的由来吧，刚开始接触朴素贝叶斯时查阅了很多资料，其中有几篇让我印象非常深刻，也推荐大家看一看：

## 参考资料
* 首先是wiki，理所当然，它的专业性很强，如果你的数学功底比较扎实的话，有这一篇就够了：
[朴素贝叶斯分类器](https://zh.wikipedia.org/wiki/%E6%9C%B4%E7%B4%A0%E8%B4%9D%E5%8F%B6%E6%96%AF%E5%88%86%E7%B1%BB%E5%99%A8)
* 这篇文章写的很好，举了很多例子，探讨了朴素贝叶斯更为深层的意义，且专业程度比起wiki有过之而无不及：
[数学之美番外篇：平凡而又神奇的贝叶斯方法](http://mindhacks.cn/2008/09/21/the-magical-bayesian-method/)
* 如果你像我一样，数学课没有好好听，那就看看下面这篇文章，相当之通俗易懂，而又很容易看清朴素贝叶斯的本质：
[自己动手写贝叶斯分类器给图书分类](http://www.jianshu.com/p/f6a3f3200689)

关于
---

__作者__：张佳军

__阅读__：45

__点赞__：2

__创建__：2017-06-24
