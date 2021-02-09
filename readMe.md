[TOC]



### demo-designPattern为部分设计模式在项目中的实践运用 

#### 一、装饰器模式

```
attributes package为装饰器模式的运用，实际业务规则为系统根据一批数据进行分析计算，产出5类计算结果，这5类计算结果在页面查询展示时均需要拼接2类信息：准备金和评估属性。故而抽取出了装饰器模型，5类计算结果为待装饰目标类，需拼接的2类信息为装饰类。 
```



#### 二、建造者模式

```
calucate package为建造者模式的运用，实际业务规则为系统根据一批数据进行分析计算，同时产出5类计算结果。可将计算结果看做一个整体对象，5类结果分别为组成部分。复杂对象的创建适合使用建造者模式。
```

 

#### 三、责任链模式

```
detailUpload package责任链模式的运用：实际业务规则为系统需对上传的excel文件进行校验，按照校验规则可以分为4类校验：
        1）、单行数据的基本格式校验 。
        2）、单行数据的业务规则校验。 
        3）、整体数据的业务规则校验。 
        4）、整体数据对接第三方系统校验。
四部分内容形成校验责任链。
```



#### 四、策略模式

```
detailUpload package策略模式的运用：在校验过程中，由于数据有不同的类型，某些规则对应不同的校验方式，故而抽取出策略模型，由于策略非常多，为防止类爆炸，具体的策略实现采用了内部类的封装形式。 
```



#### 五、模板模式

```
export package为模板模式的运用：整个系统中，excel导出功能非常之多，然而导出功能的流程是非常固定的，为避免重复造轮子造成代码臃肿不规范，抽取出模板模式模型，统一全局导出功能。
```

 

#### 六、状态模式

```
task package为状态模式的运用：在评估任务中，有大任务和小任务之分，并且任务有状态变化，伴随不同状态有不同的处理方式，非常适合使用状态模式。
```



### demo-DDD领域驱动设计模型在项目中的实践运用

#### 一、为什么使用DDD（领域驱动设计domain driven design ）

#### 二、概念

##### 1、界限上下文Bounded context
	划分领域边界，边界内领域模型保持一致，强调内聚，并与边界外的领域模型解耦。

##### 2、实体Entity
	有唯一标识，可变的业务实体对象，它有着自己的生命周期。

##### 3、值对象Value Object
	没有唯一业务标识，通常依附于其他领域实体，值对象的内容不可变，要么被整体替换。

##### 4、聚合Aggregate
	是一组业务关联度很强的实体/值对象集合，每个聚合都有一个根实体（Root Entity），通过根实体可以路由到整个聚合。聚合与聚合之间通过引用聚合根ID进行关联，访问聚合内的对象只能通过聚合根操作。

##### 5、领域事件Domain Event
	领域中发生的异步处理事件、异步消息通知等，比如：异步写入的登录历史记录。通常借助消息队列实现。

##### 6、领域服务Domain Service
	当某些业务行为无法归类到某一个Entity/Value Object时，我们便可以创建领域服务来完成。

##### 7、领域对象工厂Factory
	用于复杂领域对象的创建/重建。重建是指通过respostory加载持久化对象后，重建领域对象。

##### 8、仓库Repository
	严格意义上将仓库是基础设施层的东西，但是为了保持领域模型的整体性，我们将仓库的接口定义放到领域中，这样可以在领域中约束实体/值对象的增删改查接口，同时还可以方便地完成仓库的内存形式实现，使得领域模型弱依赖于持久化层。

##### 9、命令查询职责分离CQRS

```

```



#### 三、功能重构

##### 1、明细数据上传：

**业务建模**：上传一个文件，一个文件可包含多个上传批次，若校验不通过，会生成错误详情。

**实体抽取**：上传记录、上传批次、批次数据、错误详情

**聚合划分**：光从这个业务模型来说，上传记录、上传批次、批次数据、错误详情即为一个聚合，明显的，聚合根为上传记录。然而系统后续会增加迭代的一个业务模型是：校验通过的上传批次会生成一个子账单、一个中间层任务。也就是说子账单、中间层与上传批次可能存在一对一的关系。上传记录本身在上传完成之后在其他的业务功能中不再用到，而子账单、中间层与上传批次的直接关联操作非常频繁，如果选取上传记录为聚合根，则子账单、中间层与上传批次的关系需要经过上传记录绕一圈导航至上传批次。为方便起见，此处将上传记录与后续的上传批次、批次数据、错误详情拆分为两个聚合，上传记录和上传批次分别为聚合根，如此，后续操作将方便许多。

**值对象抽取：**抽取出的值对象为：上传状态、批次状态、错误类型、错误信息

**领域事件：**该功能下涉及三个领域事件：

​					1、上传记录生成后，触发领域事件---解析上传文件，生成明细批次

​					2、明细批次校验通过后，触发领域事件---生成子账单

​					3、明细批次校验通过后，触发领域事件---生成中间层任务

**仓储：**一个聚合根对应一个仓库，仓库接口定义在domain层，仓库实现定义在基础设施infrastructure层，仓储用于衔接domain层与持久化层，处理domain对象与持久化对象的相互转换。

##### 2、子账单

**业务建模**：校验通过的上传批次生成一个子账单

**实体抽取**：子账单记录、子账单明细

**聚合划分**：子账单记录、子账单明细为一个聚合，子账单记录为聚合根

**领域事件：**无

**领域服务：**明细数据上传时，校验一通过的上传批次需要生成子账单对接第三方系统进行校验二，明细批次校验通过后，触发领域事件生成子账单，生成子账单这个操作在两个聚合内共用，故而将生成子账单抽取为一个领域服务。

**仓储：**一个聚合根对应一个仓库，仓库接口定义在domain层，仓库实现定义在基础设施infrastructure层，仓储用于衔接domain层与持久化层，处理domain对象与持久化对象的相互转换。