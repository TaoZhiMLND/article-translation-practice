Narayana定期恢复XA事务
======================

> * 原文链接 : [Narayana periodic recovery of XA transactions](http://jbossts.blogspot.com/2018/01/narayana-periodic-recovery-of-xa.html)

让我们用Narayana特有的细节来讨论事务恢复。
这篇博客文章与JTA事务相关。如果您为JTS配置恢复，您仍然可以在这里找到相关信息，但是您需要查阅[Narayana文档](https://narayana.io/docs/product/index.html#d0e1281)。

> 什么是事务恢复

事务恢复是活动事务由于某种原因失败时所需要的过程。可能是事务管理器(JVM)的进程崩溃，也可能是到资源(数据库)的连接失败，或者其他原因导致失败。

事务的失败可能发生在事务生命周期的不同时间点，这个时间点定义了处于进行中的事务所处的状态。状态可以是留在内存中的状态，事务管理器依赖于资源事务超时来释放它。但是，它可以在调用prepare之后执行事务状态(通过成功的prepare调用，2PC事务确认能够完成事务，并承诺使用commit完成更多事务)。必须有一个进程来完成这样的事务余数。这个过程就是事务恢复。

让我们回顾一下服务于三种不同事务状态的三种失效变体。它们的结果和终止的需要将指导我们掌握事务恢复过程的工作方式。

事务管理器运行一个全局事务，其中包括几个事务分支(当使用来自XA规范的术语时)。在我们关于2PC的文章中，我们使用了(不是很精确地)资源定位事务这个术语来代替事务分支。

假设我们有一个全局事务，其中包含对数据库的数据插入以及向消息代理队列发送消息。

我们将研究事务管理器(JVM进程)的崩溃，其中每个点表示三个示例用例中的一个。示例显示了操作的时间轴，并在崩溃发生时显示不同的时间。

> 启动了全局事务，发生了数据库插入，现在JVM崩溃了。(没有消息发送到队列)。

在这种情况下，所有的事务元数据都只保存在内存中。在JVM重新启动之后，事务管理器之前不知道全局事务的存在。
但是数据库的插入已经发生了，而且数据库的一些工作已经在进行中了。但是，准备调用并没有承诺以提交结束，所有内容都只存储在内存中，因此事务管理器依赖于数据库来中止工作本身。通常在事务超时过期时发生。

> 启动全局事务，将数据插入数据库，并将消息发送到队列。全局事务请求提交。两阶段提交开始——对数据库(位于资源的事务)调用prepare。现在事务管理器(JVM)崩溃了。

如果允许交叉数据操作，则2PC提交将失败。但对prepare的调用，意味着对事务成功结束的希望。因此，对prepare的调用会导致采取锁来防止其他事务交叉。

当事务管理器重新启动时，由于所有内存中的状态都被清除，仍然找不到事务的痕迹。到目前为止，在Narayana事务日志中没有保存任何内容。

但是数据库事务处于准备状态，并且带有锁。最重要的是，处于准备状态的事务无法通过事务超时回滚，需要等待其他方完成它。

> 启动全局事务，将数据插入数据库，并将消息发送到队列。请求提交事务。两阶段提交开始——准备也在数据库和消息队列上被调用。准备阶段的记录成功也被保存到Narayana事务日志中。现在事务管理器(JVM)崩溃了。

事务管理器重新启动后，内存中没有状态，但是我们可以观察Narayana事务日志中的记录，并且数据库和JMS队列的带资源定位的事务都处于带锁的准备状态。

在后两种情况下，事务状态在JVM崩溃后仍然存在——一种情况是数据库中的存在防止事务交叉的锁，在另一种情况下，数据库中防止事务交叉的锁存在的同时，事务日志中也会出现一条记录。第一种情况下数据库内存中的事务负责自身的回滚，事务管理器不负责完成它。

完成未完成事务的工作属于恢复管理器。恢复管理器的目的是定期检查Narayana事务日志和资源事务日志(未完成的带资源定位的事务——它运行XAResource.recover()的JTA API调用)的状态。
如果发现可疑事务，恢复管理器要么将其回滚——例如第二种情况，要么像崩溃前整个准备阶段最初成功完成时一样将其提交，请参见第三种情况。

> Narayana定期恢复细节

周期性恢复是一个可配置的过程。这带来了灵活性的使用，但如果你想要运行它必要使用适当的设置。我们建议查看Narayana文档的故障恢复章节。

恢复运行周期性(默认情况下每两分钟一次)——可以通过设置系统属性RecoveryEnvironmentBean.periodicRecoveryPeriod来更改恢复周期。
启动后，它将遍历所有已注册的恢复模块(参见Narayana codebase com.arjuna.ats.arjuna.recovery.RecoveryModule)，并按以下顺序运行:在所有恢复模块上调用periodicWorkFirstPass方法，等待由RecoveryEnvironmentBean.recoveryBackoffPeriod定义的时间，在所有恢复模块上调用方法RecoveryEnvironmentBean.recoveryBackoffPeriod.
当您想要运行标准的JTA XA事务(JTS不同，您可以检查Narayana代码库中的配置示例)时，您需要配置XARecoveryModule。然后，XARecoveryModule需要配置XAResourceOrphanFilters，它在仅在资源端可用时管理完成有疑问的事务(第二种情况表示这种场景)。

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

要配置属性的值，您需要使用以下方法之一来定义它
通过系统属性-参见快速入门例子中的pom.xml。
我们可以看到属性是在JVM参数行传递的。
通过使用描述符文件jbossts-properties.xml。
这通常是使用Narayana的独立应用程序中配置的主要来源。您可以看到示例jbossts-properties.xml，并注意到作为独立的应用程序也不例外。
描述符必须位于Narayana的类路径中，以便能够访问它。
通过调用bean setter方法。
这是一种编程方法，通常主要在托管环境中使用，因为它们和WildFly一样是应用服务器。

系统属性的使用优先于描述符jbossts-properties.xml的使用。
setter方法的程序调用的使用优先于系统属性的使用。
narayana-idlj-jts.jar 使用的默认设置可以在<https://github.com/jbosstm/narayana/blob/master/ArjunaJTS/narayana-jts-idlj/src/main/resources/jbossts-properties.xml>中找到.当您没有定义任何属性文件时，这些是(特定bean中的设置组合)默认值。
有关配置的更多细节，请查看Narayana.io文档。

如果您想使用编程方法并调用bean setter，您需要首先获得bean实例。这通常是通过调用PropertyManager的静态方法来完成的。根据您想要配置的内容，有各种各样的配置。
与我们有关的是:
arjPropertyManager for CoreEnvironmentBean etc.
recoveryPropertyManager for RecoveryEnvironmentBean
jdbcPropertyManager for JDBCEnvironementBean

我们将在 jdbc transactional driver quickstart 示例的 the recovery utility 类中研究可编程的功能，其中在代码中重置了XAResourceRecovery的属性控制值。

如果您想了解jbossts-properties.xml中的系统属性或条目的确切名称，依靠经验法则是获取bean的简短类名，并在末尾添加点和属性名。
例如，假设您想要重新定义周期性恢复周期的时间段。然后您需要访问RecoveryEnvironmentBean，查找变量的名称——它是periodicRecoveryPeriod。通过使用经验法则，将使用名称RecoveryEnvironmentBean.periodicRecoveryPeriod用于重新定义默认为2分钟的值。

一些bean使用@PropertyPrefix注解，它提供了对属性进行设置的其他命名方式。对于periodicRecoveryPeriod，我们可以使用system属性名com.arjuna.ats.arjuna.recovery.periodicRecoveryPeriod 以同样的方式重置它。

注意:一个有趣的链接可能是与Tomcat集成的设置，它使用的是编程方法，请参阅NarayanaJtaServletContextListener。

考虑XA恢复
我希望你对事务恢复过程的设置和工作有方式有了更好的理解。
XARecoveryModule负责处理2PC XA事务的恢复。
该模块负责提交未完成的事务，并通过运行注册的XAResourceOrphanFilter来处理成为孤立的分支事务。

正如您所看到的，我们在jbossts-properties.xml描述符中配置了两个recoverymodule——XARecoveryModule和AtomicActionRecoveryModule。
AtomicActionRecoveryModule负责从对象存储中加载资源，如果它是可序列化的，并且作为整体保存在Narayana事务日志中，那么它可以在恢复期间被反序列化并立即使用。
这种情况并不常见。当XAResource不是可序列化的(这很难实现，例如对于数据库，我们需要一个连接来做任何工作)，Narayana提供恢复过程启动所需的资源。这需要一个类(一个代码和一个设置)，它可以为恢复目的提供XAResource。为了获得资源，我们需要一个连接(到数据库、到jms代理……)。XARecoveryModule使用两个接口的对象来获取这些信息(获取用于恢复的XAResources)。
这些接口是

XAResourceRecoveryHelper
XAResourceRecovery

然后，两个接口都包含检索资源的方法(XAResourceRecoveryHelper.getXAResources()， xaresourcerecover . getxaresource())。然后，XARecoveryModule请求所有接收到的XAResources查找有问题的事务(通过调用XAResource.recovery()) (又名: 资源定位事务)。
然后将找到的可疑事务与Narayana事务日志存储中的事务配对。如果找到匹配项，就可以调用XAResource.commit()。
也许你想知道这两种接口在大多数情况下是相同的——哪一种是正确的——但是它们的使用是不同的。XAResourceRecoveryHelper被设计成(且仅可用)以编程方式使用。要添加帮助器而不是其他的，您需要调用XARecoveryModule.addXAResourceRecoveryHelper()。您甚至可以通过方法调用xarecoverymodule.remoaresourcerecoveryhelper来取消helper的注册。
XAResourceRecovery不是直接在代码中配置的，而是通过属性com.arjuna.ats.jta.recovery.XAResourceRecovery配置的。这对于动态更改是不可行的，因为在正常情况下不可能重置它—即使您试图通过调用JTAEnvironmentBean.setXaResourceRecoveryClassNames()来更改值。

> 运行恢复管理器
我们已经解释了如何配置恢复属性，但是我们没有指出一个重要的事实——在独立的应用程序中没有自动启动恢复管理器。您需要手动启动它。

很好的一点是，这非常容易(如果您不使用ORB)，只需要调用

      RecoveryManager manager = RecoveryManager.manager();
      manager.initialize()

这将运行一个间接恢复管理器(RecoveryManager.INDIRECT_MANAGEMENT)，它生成一个线程，该线程定期运行恢复进程。如果您认为需要在需要的时间内运行周期性恢复(然后不使用周期性超时值)，您可以使用direct management并调用来手动运行它

      RecoveryManager manager = RecoveryManager.manager(RecoveryManager.DIRECT_MANAGEMENT);
      manager.initialize();
      manager.scan();

要停止恢复管理器工作，请使用terminate调用。

      manager.terminate();

总结
这篇博客文章试图介绍Narayana中的事务恢复过程。
我们的目标是为XA事务提供必要的恢复设置，并展示如何在应用程序中启动恢复管理器。
