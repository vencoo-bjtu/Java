已剪辑自: [https://www.yuque.com/hollis666/wo6vlm/ixgoek25ybmy7ws4](https://www.yuque.com/hollis666/wo6vlm/ixgoek25ybmy7ws4)

### 回答

`Spring`的事务传播机制**用于控制在多个事务方法相互调用时事务的行为。**

在复杂的业务场景中，**多个事务方法之间的调用可能会导致事务的不一致**，如出现**数据丢失**、**重复提交**等问题，使用事务传播机制可以避免这些问题的发生，**保证事务的一致性和完整性**。

`Spring`的事务规定了`7`种事务的传播级别，默认的传播机制是`REQUIRED`：

1. `REQUIRED`，如果不存在事务则开启一个事务，如果存在事务则加入之前的事务，总是只有一个事务在执行；

> 假如当前要执行的事务不在另外一个事务里，那么就起一个新的事务。
> 比如说，`ServiceB.methodB`的事务级别定义为`PROPAGATION_REQUIRED`，那么由于执行`ServiceA.methodA`的时候`ServiceA.methodA`已经起了事务，这时调用`ServiceB.methodB`，`ServiceB.methodB`看到自己已经运行在`ServiceA.methodA`的事务内部，就不再起新的事务。
> 而假如`ServiceA.methodA`运行的时候发现自己没有在事务中，他就会为自己分配一个事务。这样，在`ServiceA.methodA`或者在`ServiceB.methodB`内的任何地方出现异常事务都会被回滚。
> 即使`ServiceB.methodB`事务已被提交，接下来在`Service.methodA`中`fail`要回滚，`ServiceB.methodB`也会回滚。

2. `REQUIRES_NEW`，每次执行新开一个事务；

> 假如设计 `ServiceA.methodA` 的事务级别为`PROPAGATION_REQUIRED`，`ServiceB.methodB`的事务级别为`PROPAGATION_REQUIRES_NEW`，那么当执行到`ServiceB.methodB`的时候`ServiceA.methodA`所在的事务就会挂起，`ServiceB.methodB`会起一个新的事务，等待`ServiceB.methodB`的事务完成以后他才继续执行。
> 他与`PROPAGATION_REQUIRED`的事务区别在于事务的回滚程度了。
> 因为`ServiceB.methodB`是新起一个事务，那么就是存在两个不同的事务。如果`ServiceB.methodB`已经提交，那么`ServiceA.methodA`失败回滚，`ServiceB.methodB`是不会回滚的。如果`ServiceB.methodB`失败回滚，如果他抛出的异常被`ServiceA.methodA`捕获，`ServiceA.methodA`仍可能完成提交。

3. `SUPPORTS`，有事务则加入事务，没有事务则普通执行；
4. `NOT_SUPPORTED`，有事务则暂停该事务，没有则普通执行；

> 当前不支持事务。
> 比如`ServiceA.methodA`的事务级别是`PROPAGATION_REQUIRED`，而`ServiceB.methodB`的事务级别是`PROPAGATION_` `NOT_SUPPORTED`，那么当执行到`ServiceB.methodB`时`ServiceA.methodA`的事务挂起，等`ServiceB.methodB`以非事务的状态运行完，再继续`ServiceA.methodA`的事务。

5. `MANDATORY`，强制有事务，没有事务则报异常；
6. `NEVER`，有事务则报异常；

> 不能在事务中运行。
> 假设`ServiceA.methodA`的事务级别是`PROPAGATION_REQUIRED`，`ServiceB.methodB`的事务级别是`PROPAGATION_NEVER`。那么`ServiceB.methodB`就要抛出异常了。

7. `NESTED`，如果之前有事务，则创建嵌套事务，嵌套事务回滚不影响父事务，反之父事务影响嵌套事务。

> 理解`Nested`的关键是`savepoint`。他与`PROPAGATION_REQUIRES_NEW`的区别是`PROPAGATION_REQUIRES_NEW`另起一个事务将会与他的父事务相互独立，而`Nested`的事务和他的父事务是相依的他的提交是要等和他的父事务一块提交的。也就是说，如果父事务最后回滚，他也要回滚的。
> 而`Nested`事务的好处是他有一个`savepoint`。当发现有多种类型的`Bean`时，`@Primary`注解会通知`loC`容器优先使用它所标注的`Bean`进行注入；`@Quelifier`注解可以与`@AutoWired`注解组合使用，达到通过类型和名称一起筛选`Bean`的效果。




### 扩展

#### 用法

假设有两个业务方法`A`和`B`，方法`A`在方法`B`中被调用，需要在事务中保证它们的一致性，如果方法`A`或方法`B`中的任何一个方法发生异常，则需要回滚事务。

使用`Spring`的事务传播机制，可以在方法`A`和方法`B`上使用相同的事务管理器，并通过设置相同的传播行为来保证事务的一致性和完整性。具体实现如下：

```java
1 @Service
2 public class TransactionFooService {
3  @Autowired
4  private FooDao fooDao;
5  
6  @Transactional(
7   propagation = Propagation.REQUIRED, 
8   rollbackFor = Exception.class
9  )
10  public void methodA() throws Exception {
11   // do something
12   fooDao.updateFoo();
13  }
14 }
15 
16 
17 @Service
18 public class TransactionBarService {
19  @Autowired
20  private BarDao barDao;
21 
22  @Autowired
23  private TransactionFooService transactionFooService;
24  @Transactional(
25   propagation = Propagation.REQUIRED, 
26   rollbackFor = Exception.class
27  )
28  public void methodB() throws Exception {
29   // do something
30   barDao.updateBar();
31   transactionFooService.methodA();
32  }
33 }
```

在上述示例中，方法`A`和方法`B`都使用了`REQUIRED`的传播行为，表示如果当前存在事务，则在当前事务中执行；如果当前没有事务，则创建一个新的事务。如果在方法`A`或方法`B`中出现异常，则整个事务会自动回滚。



#### rollbackFor

`rollbackFor`是`Spring`事务中的一个**属性**，**用于指定哪些异常会触发事务回滚。**

在一个事务方法中，如果发生了`rollbackFor`属性指定的异常或其子类异常，则事务会回滚。如果不指定`rollbackFor`，则默认情况下只有`RuntimeException`和`Error`会触发事务回滚。





#### 场景题

问：一个长的事务方法`a`，在读写分离的情况下，里面既有读库操作，也有写库操作，再调用个读库方法`b`，方法`b`该用什么传播机制呢？

这种情况，读方法如果是最后一步，直接`not_supported`就行了，避免读报错导致数据回滚。如果是中间步骤，最好还是要`required`，因为异常失败需要回滚一下。

