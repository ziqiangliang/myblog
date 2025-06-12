> 这是我在排查一个开源项目 [Gravitino](https://github.com/apache/gravitino) 线上问题时的实战记录。起初问题表现得并不明显，只是偶尔数据库连接出错，经过深入观察，发现数据库活跃连接数（active connections）一直不降，最终定位到 SqlSession 未关闭的严重连接泄漏。

---

## 🧩 问题背景和初期症状

在项目刚上线时，偶尔收到如下数据库通讯异常：

```

CommunicationsException: The last packet successfully received from the server was ...

```

但这时并未意识到这是连接泄漏的问题，日志偶尔出现且量不大，数据库压力不大。

经过一段时间观察，我们开始关注数据库连接池状态，发现：

- 活跃连接数（numActive）不断累积，且即使业务访问停止也不下降；
- 空闲连接数（numIdle）非常少，基本为 0；

这表明连接池里的连接被“借出”后未归还，导致连接池耗尽。

---

## 🔍 初步分析

我们使用了 Apache DBCP2 作为连接池，配置了如下参数：

```java
dataSource.setTestOnBorrow(true);
dataSource.setValidationQuery("SELECT 1");

```

在借出连接时验证连接有效，**理论上应该不会出现“死连接”问题**。

通过观察数据连接池的监控得知：

*   `numActive`: 持续增长，不下降

*   `numIdle`: 逐渐为 0

这意味着连接被借出后，**没有归还**。

* * *



## 🧠 关键排查手段：连接追踪记录器

虽然尝试了 `Arthas` 和线程 dump，但都无法明确知道“谁借用了连接”。

于是我们实现了一个自定义的连接追踪器 `TrackingDataSource`：

```
public class TrackingDataSource extends DelegatingDataSource {
    private final ConcurrentHashMap<Connection, StackTraceElement[]> checkoutMap = new ConcurrentHashMap<>();

    @Override
    public Connection getConnection() throws SQLException {
        Connection conn = super.getConnection();
        checkoutMap.put(conn, Thread.currentThread().getStackTrace());
        return Proxy.newProxyInstance(...); // 包装 Connection，拦截 close
    }

    public void logLeakedConnections() {
        for (Map.Entry<Connection, StackTraceElement[]> entry : checkoutMap.entrySet()) {
            log.warn("Leaked connection acquired at:");
            for (StackTraceElement ste : entry.getValue()) {
                log.warn("  at " + ste);
            }
        }
    }
}

```

再配合一个定时任务每分钟打印：

```
Executors.newSingleThreadScheduledExecutor().scheduleAtFixedRate(
    () -> trackingDataSource.logLeakedConnections(),
    1, 1, TimeUnit.MINUTES
);

```

**就是它，让我们看到了是哪些地方借走连接而未归还！**

* * *

## 🕵️♂️ 真相：SqlSession 未关闭！

定位后发现问题代码在：

```
org.apache.gravitino.storage.relational.utils.SessionUtils

```

以下两个静态方法未关闭 `SqlSession`：

```
public static <T, R> R doWithoutCommitAndFetchResult(Class<T> mapperClazz, Function<T, R> func) {
  T mapper = SqlSessions.getMapper(mapperClazz);
  return func.apply(mapper); // ❌ SqlSession 未关闭
}

public static <T> void doWithoutCommit(Class<T> mapperClazz, Consumer<T> consumer) {
  T mapper = SqlSessions.getMapper(mapperClazz);
  consumer.accept(mapper);  // ❌ SqlSession 未关闭
}

```

这两个方法调用了 `SqlSessions.getMapper(...)`，实际内部打开了 `SqlSession`，放入 `ThreadLocal`，但没有关闭，**连接自然也就不会释放**。

* * *

## ✅ 正确示例：显式关闭资源

在该类中另一方法就是正确的做法：

```
public static <T> void doWithCommit(Class<T> mapperClazz, Consumer<T> consumer) {
  try (SqlSession session = SqlSessions.getSqlSession()) {
    try {
      T mapper = SqlSessions.getMapper(mapperClazz);
      consumer.accept(mapper);
      SqlSessions.commitAndCloseSqlSession();
    } catch (Throwable t) {
      SqlSessions.rollbackAndCloseSqlSession();
      throw t;
    }
  }
}

```

使用 `try-with-resources` + 显式事务处理，可以保证无论正常或异常，连接都能释放。

* * *

## 🧪 如何重现该问题

1.  启动使用 Gravitino 的服务，使用连接池（例如 DBCP）并设置连接数上限；

2.  反复调用如 `GET /metalakes/{metalake}/objects/{type}/{fullName}/tags`（内部使用 `doWithoutCommitAndFetchResult`）的接口；

3.  查看连接池状态，`active` 持续升高，`idle` 不回升；

4.  最终触发 `CommunicationsException` 或连接获取阻塞。

* * *

## 💡 修复方案与建议

我们临时对 `SessionUtils` 增加关闭逻辑，例如：

```
public static <T, R> R doWithoutCommitAndFetchResult(Class<T> mapperClazz, Function<T, R> func) {
  try (SqlSession session = SqlSessions.getSqlSession()) {
    try {
      T mapper = session.getMapper(mapperClazz);
      return func.apply(mapper);
    } catch(Thrownable t){
    throw t;
}finally{
  SqlSessions.closeSqlSession();
}
  }
}

```

并已在 GitHub 上提了 [Issue](https://github.com/apache/gravitino/issues) 建议修复，避免更多用户踩坑。

* * *

## 📝 小结 & 建议

*   ❗ **连接泄露比连接超时更可怕**，因为它不会抛异常，却能悄无声息拖垮系统；

*   🛡 `SqlSession` 是典型的非线程安全对象，一定要显式关闭；

*   🔍 ThreadLocal + 连接池是一对“危险组合”，要格外注意清理逻辑；

*   🧰 工具推荐：`Arthas`、`JDK Thread Dump`、`DBCP metrics` 都是排查利器；

*   📚 开源项目使用时，也要保持审慎，审查其事务控制和资源释放设计。

* * *

如果你也遇到类似问题，希望这篇排查思路对你有所帮助。欢迎留言交流！
