Narayana定期恢复XA事务
---

> * 原文链接 : [Narayana periodic recovery of XA transactions](http://jbossts.blogspot.com/2018/01/narayana-periodic-recovery-of-xa.html)

Let's talk about the transaction recovery with details specific to Narayana.
This blog post is related to JTA transactions. If you configure recovery for JTS, still you can find a relevant information here but then you will need to consult [the Narayana documentation](https://narayana.io/docs/product/index.html#d0e1281).

让我们用Narayana特有的细节来讨论事务恢复。
这篇博客文章与JTA事务相关。如果您为JTS配置恢复，您仍然可以在这里找到相关信息，但是您需要查阅[Narayana文档](https://narayana.io/docs/product/index.html#d0e1281)。

> What is the transaction recovery
> 什么是事务恢复

The transaction recovery is process needed when an active transaction fails for some reason. It could be a crash of process of the transaction manager (JVM) or connection to the resource (database)could fail or any other reason for failure.

事务恢复是活动事务由于某种原因失败时所需要的过程。可能是事务管理器(JVM)的进程崩溃，也可能是到资源(数据库)的连接失败，或者其他原因导致失败。

The failure of the transaction could happen at various points of the transaction lifetime and the point define the state which the in-progress transaction was left at. The state could be just an in-memory state which is left behind and transaction manager relies on the resource transaction timeout to release it. But it could the transaction state after prepare was called (by successful prepare call the 2PC transaction confirms that is capable to finish transaction and more of what it promises to finish the transaction with commit). There has to be a process which finishes such transaction remainders. And that process is the transaction recovery.

事务的失败可能发生在事务生命周期的不同时间点，这个时间点定义了处于进行中的事务所处的状态。状态可以是留在内存中的状态，事务管理器依赖于资源事务超时来释放它。但是，它可以在调用prepare之后执行事务状态(通过成功的prepare调用，2PC事务确认能够完成事务，并承诺使用commit完成更多事务)。必须有一个进程来完成这样的事务余数。这个过程就是事务恢复。

Let's review three variants of failures which serves three different transaction states. Their results and needs of termination will guide us through the work of the transaction recovery process.

让我们回顾一下服务于三种不同事务状态的三种失效变体。它们的结果和终止的需要将指导我们掌握事务恢复过程的工作方式。

Transaction manager runs a global transaction which includes several transaction branches (when using term from XA specification). In our article about 2PC we used (not precisely) term resource-located transaction instead of the transaction branch.

事务管理器运行一个全局事务，其中包括几个事务分支(当使用来自XA规范的术语时)。在我们关于2PC的文章中，我们使用了(不是很精确地)资源定位事务这个术语来代替事务分支。

Let's say we have a global transaction containing data insertion to a database plus sending a message to a message broker queue.

假设我们有一个全局事务，其中包含对数据库的数据插入以及向消息代理队列发送消息。

We will examine the crash of transaction manager (the JVM process) where each point represents one of the three example cases. The examples show the timeline of actions and differ in time when the crash happens.

我们将研究事务管理器(JVM进程)的崩溃，其中每个点表示三个示例用例中的一个。示例显示了操作的时间轴，并在崩溃发生时显示不同的时间。

The global transaction was started and insertion to database happened, now JVM crashes. (no message was sent to queue). In this situation, all the transaction metadata is saved only in the memory. After JVM is restarted the transaction manager has no notion of existence of the global transaction in time before.
But the insertion to the database already happened and the database has some work in progress already. But there was no promise by prepare call to end with commit and everything was stored only as an in-memory state thus transaction manager relies on the database to abort the work itself. Normally it happens when transaction timeout expires.

> 启动了全局事务，发生了数据库插入，现在JVM崩溃了。(没有消息发送到队列)。

在这种情况下，所有的事务元数据都只保存在内存中。在JVM重新启动之后，事务管理器之前不知道全局事务的存在。
但是数据库的插入已经发生了，而且数据库的一些工作已经在进行中了。但是，准备调用并没有承诺以提交结束，所有内容都只存储在内存中，因此事务管理器依赖于数据库来中止工作本身。通常在事务超时过期时发生。

The global transaction was started, data was inserted into the database and message was sent to the queue. The global transaction is asking to commit. The two-phase commit begins – the prepare is called on the database (resource-located transaction). Now the transaction manager (the JVM) crashes. If an interleaving data manipulation would be permitted then the 2PC commit would fail. But calling of prepare means the promise of the successful end. Thus the call of prepare causes locks to be taken to prevent other transactions to interleave.

> 启动全局事务，将数据插入数据库，并将消息发送到队列。全局事务请求提交。两阶段提交开始——对数据库(位于资源的事务)调用prepare。现在事务管理器(JVM)崩溃了。

如果允许交叉数据操作，则2PC提交将失败。但对prepare的调用，意味着对事务成功结束的希望。因此，对prepare的调用会导致采取锁来防止其他事务交叉。

When transaction manager is restarted but again no notion of the transaction could be found as all the in-memory state was cleared. And there was nothing to be saved in the Narayana transaction log so far.

当事务管理器重新启动时，由于所有内存中的状态都被清除，仍然找不到事务的痕迹。到目前为止，在Narayana事务日志中没有保存任何内容。

But the database transaction is in the prepared state and with locks. On top of it, the transaction in the prepared state can't be rolled-back by transaction timeout and needs to wait for some other party to finish it.

但是数据库事务处于准备状态，并且带有锁。最重要的是，处于准备状态的事务无法通过事务超时回滚，需要等待其他方完成它。

The global transaction was started, data inserted into the database and message was sent to the queue. The transaction was asked to commit. The two-phase commit begins – the prepare is called on the database and on the message queue too. A record success of the prepare phase is saved to the Narayana transaction log too. Now the transaction manager (JVM) crashes.

> 启动全局事务，将数据插入数据库，并将消息发送到队列。请求提交事务。两阶段提交开始——准备也在数据库和消息队列上被调用。准备阶段的记录成功也被保存到Narayana事务日志中。现在事务管理器(JVM)崩溃了。

After the transaction manager is restarted there is no in-memory state but we can observe the record in the Narayana transaction log and that database and the JMS queue resource-located transactions are in the prepared state with locks.

事务管理器重新启动后，内存中没有状态，但是我们可以观察Narayana事务日志中的记录，并且数据库和JMS队列的带资源定位的事务都处于带锁的准备状态。

In the later cases the transaction state survived the JVM crash - once only at the side of locked records of a database, in other cases, a record is present in transaction log too. In the first case only in memory transaction representation was used where transaction manager is not responsible to finish it.

在后两种情况下，事务状态在JVM崩溃后仍然存在——一种情况是数据库中的存在防止事务交叉的锁，在另一种情况下，数据库中防止事务交叉的锁存在的同时，事务日志中也会出现一条记录。第一种情况下数据库内存中的事务负责自身的回滚，事务管理器不负责完成它。

The work of finishing the unfinished transactions belongs to the recovery manager. The purpose of the recovery manager is to periodically check the state of the Narayana transaction log and resource transaction logs (unfinished resource-located transactions – it runs the JTA API call of XAResource.recover().
If an in-doubt transaction is found the recovery manager either roll it back - for example in the second case, or commit it as the whole prepare phase was originally finished with success, see the third case.

完成未完成事务的工作属于恢复管理器。恢复管理器的目的是定期检查Narayana事务日志和资源事务日志(未完成的带资源定位的事务——它运行XAResource.recover()的JTA API调用)的状态。
如果发现可疑事务，恢复管理器要么将其回滚——例如第二种情况，要么像崩溃前整个准备阶段最初成功完成时一样将其提交，请参见第三种情况。

Narayana periodic recovery in details

> Narayana定期恢复细节

The periodic recovery is the configurable process. That brings flexibility of the usage but made necessary to use proper settings if you want to run it.
We recommend checking the Narayana documenation, the chapter Failure Recovery.

周期性恢复是一个可配置的过程。这带来了灵活性的使用，但如果你想要运行它必要使用适当的设置。我们建议查看Narayana文档的故障恢复章节。

The recovery runs periodicity (by default each two minutes) - the period could be changed by setting system property RecoveryEnvironmentBean.periodicRecoveryPeriod). When launched it iterates over all registered recovery modules (see Narayana codebase com.arjuna.ats.arjuna.recovery.RecoveryModule) and it runs the following sequence: calling the method periodicWorkFirstPass on all recovery modules, waiting time defined by RecoveryEnvironmentBean.recoveryBackoffPeriod, calling the method RecoveryEnvironmentBean.recoveryBackoffPeriod on all recovery modules.
When you want to run standard JTA XA transactions (JTS differs, you can check the config example in the Narayana code base) then you needs to configure the XARecoveryModule for the usage. The XARecoveryModule then brings to the play need of configuring XAResourceOrphanFilters which manage finishing in-doubt transactions when available only at the resource side (the second case represents such scenario).

恢复运行周期性(默认情况下每两分钟一次)——可以通过设置系统属性RecoveryEnvironmentBean.periodicRecoveryPeriod来更改恢复周期。
启动后，它将遍历所有已注册的恢复模块(参见Narayana codebase com.arjuna.ats.arjuna.recovery.RecoveryModule)，并按以下顺序运行:在所有恢复模块上调用periodicWorkFirstPass方法，等待由RecoveryEnvironmentBean.recoveryBackoffPeriod定义的时间，在所有恢复模块上调用方法RecoveryEnvironmentBean.recoveryBackoffPeriod.
当您想要运行标准的JTA XA事务(JTS不同，您可以检查Narayana代码库中的配置示例)时，您需要配置XARecoveryModule。然后，XARecoveryModule需要配置XAResourceOrphanFilters，它在仅在资源端可用时管理完成有疑问的事务(第二种情况表示这种场景)。

Narayana periodic recovery configuration
You may ask how all this is configured, right?
The Narayana configuration is held in "beans". The "beans" contains properties which are retrieved by getter method calls all over the Narayana code. So configuration of the Narayana behaviour means redefining values of the demanded bean properties.
Let's check what are the beans relevant for setting the transaction management and recovery for XA transactions. We will use the jdbc transactional driver quickstart as an example.
The releavant beans are following
CoreEnvironmentBean
CoordinatorEnvironmentBean
JTAEnvironmentBean
RecoveryEnvironmentBean
JDBCEnvironmentBean

> Narayana定期恢复配置

您可能会问这一切是如何配置的，对吗?
Narayana配置保存在bean中。bean包含了需要在Narayana代码中通过getter方法获取的属性。因此，配置Narayana行为意味着重新定义所需bean属性的值。
让我们看看与设置XA事务的事务管理和恢复相关的bean。我们将使用jdbc事务驱动方式的quickstart程序作为示例。
下面是相关的bean
CoreEnvironmentBean
CoordinatorEnvironmentBean
JTAEnvironmentBean
RecoveryEnvironmentBean
JDBCEnvironmentBean

To configure the values of the properties you need to define it one of the following ways
via system property – see example in the quickstart pom.xml.
We can see that the property is passed at the JVM argument line.
via use of the descriptor file jbossts-properties.xml.
This is usually the main source of configuration in the standalone applications using Narayana. You can see the example jbossts-properties.xml and observe that as the standalone application is not the exception.
The descriptor has to be at the classpath for the Narayana will be able to access it.
via call of bean setter methods.
This is the programatic approach and is normally used mainly in managed environment as they are application servers as WildFly is.

要配置属性的值，您需要使用以下方法之一来定义它
通过系统属性-参见快速入门例子中的pom.xml。
我们可以看到属性是在JVM参数行传递的。
通过使用描述符文件jbossts-properties.xml。
这通常是使用Narayana的独立应用程序中配置的主要来源。您可以看到示例jbossts-properties.xml，并注意到作为独立的应用程序也不例外。
描述符必须位于Narayana的类路径中，以便能够访问它。
通过调用bean setter方法。
这是一种编程方法，通常主要在托管环境中使用，因为它们和WildFly一样是应用服务器。

The usage of the system property has precedence over use of the descriptor jbossts-properties.xml.
The usage of the programatic call of the setter method has precedence over use of system properties.
The default settings for the used narayana-idlj-jts.jar artifact can be seen at <https://github.com/jbosstm/narayana/blob/master/ArjunaJTS/narayana-jts-idlj/src/main/resources/jbossts-properties.xml>. Those are (with combination of settings inside of particular beans) default values used when you don't have any properties file defined.
For more details on configuration check the Narayana.io documentation.

系统属性的使用优先于描述符jbossts-properties.xml的使用。
setter方法的程序调用的使用优先于系统属性的使用。
narayana-idlj-jts.jar 使用的默认设置可以在<https://github.com/jbosstm/narayana/blob/master/ArjunaJTS/narayana-jts-idlj/src/main/resources/jbossts-properties.xml>中找到.当您没有定义任何属性文件时，这些是(特定bean中的设置组合)默认值。
有关配置的更多细节，请查看Narayana.io文档。

If you want to use the programatic approach and call the bean setters you need to gain the bean instance first. That is normally done by calling a static method of PropertyManager. There are various of them depending what you want to configure.
The relevant for us are:
arjPropertyManager for CoreEnvironmentBean etc.
recoveryPropertyManager for RecoveryEnvironmentBean
jdbcPropertyManager for JDBCEnvironementBean

如果您想使用编程方法并调用bean setter，您需要首先获得bean实例。这通常是通过调用PropertyManager的静态方法来完成的。根据您想要配置的内容，有各种各样的配置。
与我们有关的是:
arjPropertyManager for CoreEnvironmentBean etc.
recoveryPropertyManager for RecoveryEnvironmentBean
jdbcPropertyManager for JDBCEnvironementBean

We will examine the programmatic approach at the example of the jdbc transactional driver quickstart inside of the recovery utility class where property controlling values of XAResourceRecovery is reset in the code.

我们将在 jdbc transactional driver quickstart 示例的 the recovery utility 类中研究可编程的功能，其中在代码中重置了XAResourceRecovery的属性控制值。

If you search to understand what should be the exact name of the system property or entry in jbossts-properties.xml the rule of thumb is to take the short class name of the bean, add the dot and the name of the property at the end.
For example let's say you want to redefine time period for the periodic recovery cycle. Then you need to visit the RecoveryEnvironmentBean, find the name of the variable – which is periodicRecoveryPeriod. By using the rule of thumb will use the name RecoveryEnvironmentBean.periodicRecoveryPeriod for redefinition of the default 2 minutes value.

如果您想了解jbossts-properties.xml中的系统属性或条目的确切名称，依靠经验法则是获取bean的简短类名，并在末尾添加点和属性名。
例如，假设您想要重新定义周期性恢复周期的时间段。然后您需要访问RecoveryEnvironmentBean，查找变量的名称——它是periodicRecoveryPeriod。通过使用经验法则，将使用名称RecoveryEnvironmentBean.periodicRecoveryPeriod用于重新定义默认为2分钟的值。

Some bean uses annotations @PropertyPrefix which offers other way of naming for the property for settings it up. In case of the periodicRecoveryPeriod we can use system property with name com.arjuna.ats.arjuna.recovery.periodicRecoveryPeriod to reset it in the same way.

一些bean使用@PropertyPrefix注解，它提供了对属性进行设置的其他命名方式。对于periodicRecoveryPeriod，我们可以使用system属性名com.arjuna.ats.arjuna.recovery.periodicRecoveryPeriod 以同样的方式重置它。

Note: an interesting link could be the settings for the integration with Tomcat which useses programatic approach, see NarayanaJtaServletContextListener.

注意:一个有趣的链接可能是与Tomcat集成的设置，它使用的是编程方法，请参阅NarayanaJtaServletContextListener。

Thinking about XA recovery
I hope you have a better picture of recovery setup and how that works now.
The XARecoveryModule has the responsibility for handling recovery of 2PC XA transactions. The module is responsible for committing unfinished transactions and for handling orphans by running registered XAResourceOrphanFilter.

考虑XA恢复
我希望你对事务恢复过程的设置和工作有方式有了更好的理解。
XARecoveryModule负责处理2PC XA事务的恢复。
该模块负责提交未完成的事务，并通过运行注册的XAResourceOrphanFilter来处理成为孤岛的分支事务。

As you could see we configured two RecoveryModules – XARecoveryModule and AtomicActionRecoveryModule in the jbossts-properties.xml descriptor.
The AtomicActionRecoveryModule is responsible for loading resource from object store and if it is serializable and as the whole saved in the Narayana transaction log then it could be deserialized and used immediately during recovery.
This is not the case often. When the XAResource is not serializable (which is hard to achieve for example for database where we need to have a connection to do any work) the Narayana offers resource initiated recovery. That requires a class (a code and a settings) that could provide XAResources for the recovery purposes. For getting the resource we need a connection (to database, to jms broker...). The XARecoveryModule uses objects of two interfaces to get such information (to get the XAResources for recovery).
Those interfaces are

XAResourceRecoveryHelper
XAResourceRecovery

正如您所看到的，我们在jbossts-properties.xml描述符中配置了两个recoverymodule——XARecoveryModule和AtomicActionRecoveryModule。
AtomicActionRecoveryModule负责从对象存储中加载资源，如果它是可序列化的，并且作为整体保存在Narayana事务日志中，那么它可以在恢复期间被反序列化并立即使用。
这种情况并不常见。当XAResource不是可序列化的(这很难实现，例如对于数据库，我们需要一个连接来做任何工作)，Narayana提供恢复过程启动所需的资源。这需要一个类(一个代码和一个设置)，它可以为恢复目的提供XAResource。为了获得资源，我们需要一个连接(到数据库、到jms代理……)。XARecoveryModule使用两个接口的对象来获取这些信息(获取用于恢复的XAResources)。
这些接口是

XAResourceRecoveryHelper
XAResourceRecovery

Both interfaces then contain method to retrieve the resources (XAResourceRecoveryHelper.getXAResources(),XAResourceRecovery.getXAResource()). The XARecoveryModule then ask all the received XAResources to find in-doubt transactions (by calling XAResource.recovery()) (aka. resource located transactions).
The found in-doubt transactions are then paired with transactions in Narayana transaction log store. If the match is found the XAResource.commit() could be called.
Maybe you wonder what both interfaces are mostly the same – which kind of true – but the use differs. The XAResourceRecoveryHelper is designed (and only available) to be used in programatic way. For adding the helper amogst other ones you need to call XARecoveryModule.addXAResourceRecoveryHelper(). You can even deregister the helper by method call XARecoveryModule.removeXAResourceRecoveryHelper.
The XAResourceRecovery is configured not directly in the code but via property com.arjuna.ats.jta.recovery.XAResourceRecovery. This is not viable for dynamic changes as in normal circumstances it's not possible to reset it – even when you try to change the values by call of JTAEnvironmentBean.setXaResourceRecoveryClassNames().

然后，两个接口都包含检索资源的方法(XAResourceRecoveryHelper.getXAResources()， xaresourcerecover . getxaresource())。然后，XARecoveryModule请求所有接收到的XAResources查找有问题的事务(通过调用XAResource.recovery()) (又名: 资源定位事务)。
然后将找到的可疑事务与Narayana事务日志存储中的事务配对。如果找到匹配项，就可以调用XAResource.commit()。
也许你想知道这两种接口在大多数情况下是相同的——哪一种是正确的——但是它们的使用是不同的。XAResourceRecoveryHelper被设计成(且仅可用)以编程方式使用。要添加帮助器而不是其他的，您需要调用XARecoveryModule.addXAResourceRecoveryHelper()。您甚至可以通过方法调用xarecoverymodule.remoaresourcerecoveryhelper来取消helper的注册。
XAResourceRecovery不是直接在代码中配置的，而是通过属性com.arjuna.ats.jta.recovery.XAResourceRecovery配置的。这对于动态更改是不可行的，因为在正常情况下不可能重置它—即使您试图通过调用JTAEnvironmentBean.setXaResourceRecoveryClassNames()来更改值。

Running the recovery manager
We have explained how to configure the recovery properties but we haven't pointed down one important fact – in the standalone application there is no automatic launch of the recovery manager. You need manually to start it.
A good point is that's quite easy (if you don't use ORB) and it's fine to call just

      RecoveryManager manager = RecoveryManager.manager();
      manager.initialize()

> 运行恢复管理器
我们已经解释了如何配置恢复属性，但是我们没有指出一个重要的事实——在独立的应用程序中没有自动启动恢复管理器。您需要手动启动它。

很好的一点是，这非常容易(如果您不使用ORB)，只需要调用

      RecoveryManager manager = RecoveryManager.manager();
      manager.initialize()

This runs an indirect recovery manager (RecoveryManager.INDIRECT_MANAGEMENT) which spawns a thread which runs periodically the recovery process. If you feel that you need to run the periodic recovery in times you want (periodic timeout value is then not used) you can use direct management and call to run it manually

      RecoveryManager manager = RecoveryManager.manager(RecoveryManager.DIRECT_MANAGEMENT);
      manager.initialize();
      manager.scan();

For stopping the recovery manager to work use the terminate call.

      manager.terminate();

这将运行一个间接恢复管理器(RecoveryManager.INDIRECT_MANAGEMENT)，它生成一个线程，该线程定期运行恢复进程。如果您认为需要在需要的时间内运行周期性恢复(然后不使用周期性超时值)，您可以使用direct management并调用来手动运行它

      RecoveryManager manager = RecoveryManager.manager(RecoveryManager.DIRECT_MANAGEMENT);
      manager.initialize();
      manager.scan();

要停止恢复管理器工作，请使用terminate调用。

      manager.terminate();

Summary
This blog post tried to introduce process of transaction recovery in Narayana.
The goal was to present settings necessary to be set for the recovery would work in an expected way for XA transactions and shows how to start the recovery manager in your application.

总结
这篇博客文章试图介绍Narayana中的事务恢复过程。
我们的目标是为XA事务提供必要的恢复设置，并展示如何在应用程序中启动恢复管理器。
