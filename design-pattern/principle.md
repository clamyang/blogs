---
title: 面向对象五大原则
comments: true
---



SOLID原则：由 5 个设计原则组成的，它们分别是：单一职责原则、开闭原则、里式替换原则、接口隔离原则和依赖反转原则，依次对应 SOLID 中的 S、O、L、I、D 这 5 个英文字母，



SRP单一职责原则 Single Responsibility Principle；

KISS保持简单 Keep It Simple and Stupid；

YAGNI不需要原则 You Ain’t Gonna Need It ； 

DRY 不要重复原则 Don’t Repeat Yourself ； 

LOD 迪米特法则 Law of Demeter。



<!--more-->

## 单一职责

1. 如何理解单一职责原则（SRP）？- 
   - 一个类只负责完成一个职责或者功能。不要设计大而全的类，要设计粒度小、功能单一的类。单一职责原则是为了实现代码高内聚、低耦合，提高代码的复用性、可读性、可维护性。
2. 如何判断类的职责是否足够单一？
   - 不同的应用场景、不同阶段的需求背景、不同的业务层面，对同一个类的职责是否单一，可能会有不同的判定结果。实际上，一些侧面的判断指标更具有指导意义和可执行性，比如，出现下面这些情况就有可能说明这类的设计不满足单一职责原则：类中的代码行数、函数或者属性过多；类依赖的其他类过多，或者依赖类的其他类过多；私有方法过多；比较难给类起一个合适的名字；类中大量的方法都是集中操作类中的某几个属性。
3. 类的职责是否设计得越单一越好？
   - 单一职责原则通过避免设计大而全的类，避免将不相关的功能耦合在一起，来提高类的内聚性。同时，类职责单一，类依赖的和被依赖的其他类也会变少，减少了代码的耦合性，以此来实现代码的高内聚、低耦合。但是，如果拆分得过细，实际上会适得其反，反倒会降低内聚性，也会影响代码的可维护性。



## 开放-封闭

假设一开始我们的计算器最初只提供加法运算，后续产品又想支持别的运算，这时候**我们最好是通过增加代码的方式实现，而不是改动现有的代码**。



尽管最初设计时没有考虑到，但是，在应对后续某一个变化时，我们应该对其进行抽象分类，**我们希望的是在开发工作展开不久就知道可能发生的变化。查明可能发生的变化所等待的时间越长，要创建正确的抽象就越困难[ASD]。**



就比如最初的加法类，我们在很多个地方都用到了，后续添加减法类的时候，改动就不是一件小事情了，需要替换更多的内容。

![](https://s2.loli.net/2022/06/08/5OZwjf724JpFNKy.png)



***开放-封闭原则是面向对象设计的核心所在。遵循这个原则可以带来面向对象技术所声称的巨大好处，也就是可维护、可扩展、可复用、灵活性好。开发人员应该仅对程序中呈现出频繁变化的那些部分做出抽象，然而，对于应用程序中的每个部分都刻意地进行抽象同样不是一个好主意。拒绝不成熟的抽象和抽象本身一样重要[ASD]***



## 依赖反转

这里其实有个很形象的例子，因为自己一直在写面向过程的代码，Go 里边的 interface 用的也比较少，也正如书中讲到的例子一样，假设我们在对数据库进行操作时往往会把语句封装成具体的函数，使用上层的业务逻辑调用底层的这写函数，如 Figure 1 中展示的。![](https://s2.loli.net/2022/06/08/d8DR4uKUkErAhxm.png)

同时这里也会遇到一个问题，如果说在给另一个用户部署我们程序的时候，人家不想用 MySQL 但是业务逻辑是不需要动的，我们这时候也很难复用上层的业务逻辑（因为是和 MySQL 紧耦合的关系）。



那这时候应该怎么办呢？



我们可以根据不同的数据库抽象出来一层接口，只要是数据库就至少支持增、删、改、查四种操作，如下：

```go
type database interface {
    Insert() error
    Update() error
    Delete() error
    Select() error
}
```



然后底层去实现这个接口，MySQL 实现增、删、改、查，然后增加 Mongo、Redis 数据库依次实现这四个接口。



这样上层在调用的时候是通过接口来实现的，但是在 Go 里边这块还是有点模糊，我们当前的一个业务场景跟这个原则十分符合，只不过底层是各种不同的云服务，上层就是业务逻辑代码，这里代码中维护了一个全局map，把每一个底层云加入到这个 map 中，如下：

```go
// 伪代码如下
var drivers map[string]ICloudInterface
```



然后在每一个云的 package 下执行 register，这样业务逻辑层，通过数据库中字段不同的值，在这里获取到对应的实体，调用对应的方法即可，做到了底层实现和业务逻辑的解耦，同时修改阿里云部分的内容并不会影响华为云的逻辑。



> 之前看到过一句话，任何中间层的出现都是为了解耦。



*从业务逻辑中找到具体的底层对象，如果不通过全局变量来实现，不知道还没有别的更好的办法，* 等待着自己去发掘。



## 里氏替换

子类对象（object of subtype/derived class）能够替换程序（program）中父类对象（object of base/parent class）出现的任何地方，并且保证原来程序的逻辑行为（behavior）不变及正确性不被破坏。



```java

public class Transporter {
  private HttpClient httpClient;
  
  public Transporter(HttpClient httpClient) {
    this.httpClient = httpClient;
  }

  public Response sendRequest(Request request) {
    // ...use httpClient to send request
  }
}

public class SecurityTransporter extends Transporter {
  private String appId;
  private String appToken;

  public SecurityTransporter(HttpClient httpClient, String appId, String appToken) {
    super(httpClient);
    this.appId = appId;
    this.appToken = appToken;
  }

  @Override
  public Response sendRequest(Request request) {
    if (StringUtils.isNotBlank(appId) && StringUtils.isNotBlank(appToken)) {
      request.addPayload("app-id", appId);
      request.addPayload("app-token", appToken);
    }
    return super.sendRequest(request);
  }
}

public class Demo {    
  public void demoFunction(Transporter transporter) {    
    Reuqest request = new Request();
    //...省略设置request中数据值的代码...
    Response response = transporter.sendRequest(request);
    //...省略其他逻辑...
  }
}

// 里式替换原则
Demo demo = new Demo();
demo.demofunction(new SecurityTransporter(/*省略参数*/););
```



可以直接用 SecurityTransporter 替换掉 Transporter，即子类替换掉父类。尽管这样看起来里氏替换和多态很类似，但是他俩完全不是一回事。



多态是面向对象编程的一大特性，也是面向对象编程语言的一种语法。它是一种代码实现的思路。而里式替换是一种设计原则，是用来指导继承关系中子类该如何设计的，子类的设计要保证在替换父类的时候，不改变原有程序的逻辑以及不破坏原有程序的正确性。（摘抄自极客时间）



哪些代码明显违背了 LSP？

- 子类违背父类声明要实现的功能
- 子类违背父类对输入、输出、异常的约定
- 子类违背父类注释中所罗列的任何特殊说明



***2022-6-8 暂封***

