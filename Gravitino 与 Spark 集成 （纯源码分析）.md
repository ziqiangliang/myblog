## 背景
在上一期介绍了 Gravitino 的概念以及解决了什么问题，现在介绍下 Gravitino 如何与 Spark 集成，毕竟 Spark 的市占率是非常巨大的。
## Spark multi-catalog 介绍
### 背景
 > Spark **2.x** 推出的 `DataSourceV2` API 主要用于与外部数据存储进行数据读写集成，但缺少对表元数据（如创建、修改、删除表）的操作支持，导致 CTAS 等高阶操作行为不可控、易出错，且 Spark 的 Catalog API 也不完善，无法支持 `多 catalog` 或 `CTAS` 所需的原子性、分区配置等功能，因此 Catalog Plugin 的提出就是为了解决这些问题，提供一套标准的 catalog API 来实现对表的创建、修改、加载和删除操作，保证 Spark 对各类数据源的高阶操作行为一致且可管理。

### CatalogPlugin Interface
CatalogPlugin 在 Spark 代码中是一个 Interface，代码如下。
```
/**
 * A marker interface to provide a catalog implementation for Spark.
 * <p>
 * Implementations can provide catalog functions by implementing additional interfaces for tables,
 * views, and functions.
 * <p>
 * Catalog implementations must implement this marker interface to be loaded by
 * {@link Catalogs#load(String, SQLConf)}. The loader will instantiate catalog classes using the
 * required public no-arg constructor. After creating an instance, it will be configured by calling
 * {@link #initialize(String, CaseInsensitiveStringMap)}.
 * <p>
 * Catalog implementations are registered to a name by adding a configuration option to Spark:
 * {@code spark.sql.catalog.catalog-name=com.example.YourCatalogClass}. All configuration properties
 * in the Spark configuration that share the catalog name prefix,
 * {@code spark.sql.catalog.catalog-name.(key)=(value)} will be passed in the case insensitive
 * string map of options in initialization with the prefix removed.
 * {@code name}, is also passed and is the catalog's name; in this case, "catalog-name".
 *
 * @since 3.0.0
 */
@Evolving
public interface CatalogPlugin {
  /**
   * Called to initialize configuration.
   * <p>
   * This method is called once, just after the provider is instantiated.
   *
   * @param name the name used to identify and load this catalog
   * @param options a case-insensitive string map of configuration
   */
  void initialize(String name, CaseInsensitiveStringMap options);

  /**
   * Called to get this catalog's name.
   * <p>
   * This method is only called after {@link #initialize(String, CaseInsensitiveStringMap)} is
   * called to pass the catalog's name.
   */
  String name();

  /**
   * Return a default namespace for the catalog.
   * <p>
   * When this catalog is set as the current catalog, the namespace returned by this method will be
   * set as the current namespace.
   * <p>
   * The namespace returned by this method is not required to exist.
   *
   * @return a multi-part namespace
   */
  default String[] defaultNamespace() {
    return new String[0];
  }
}
```
从代码中我们可以获得几点有用的信息：

1. 自定义 catalog 必须实现这个 interface
2. 然后通过 `Catalog#load(String, SQLConf)` 进行加载，加载时会调用具体 Catalog 的无参构造函数方法进行初始化
3. 初始化之后会调用 `CatalogPlugin` 中的 `initialize` 方法进行初始化
4. 使用 `CatalogPlugin` 需要添加如下配置，其中第二个配置就是我们传递给 CatalogPlugin 的 initialize 方法的参数
```
spark.sql.catalog.catalog-name=com.example.YourCatalogClass
spark.sql.catalog.catalog-name.(key)=(value)
```
### TableCatalog
这是一个接口继承了 `CatalogPlugin` ,  Gravitino 与 Spark 集成它是主角，下面是它的定义：
```
/**
 * Catalog methods for working with Tables.
 * <p>
 * TableCatalog implementations may be case sensitive or case insensitive. Spark will pass
 * {@link Identifier table identifiers} without modification. Field names passed to
 * {@link #alterTable(Identifier, TableChange...)} will be normalized to match the case used in the
 * table schema when updating, renaming, or dropping existing columns when catalyst analysis is case
 * insensitive.
 *
 * @since 3.0.0
 */
@Evolving
public interface TableCatalog extends CatalogPlugin {
}
```
下面使它的主要方法：
![TableCatalog的方法](https://upload-images.jianshu.io/upload_images/11859806-9077da2494c89af4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
都是关于表操作的方法，大家可能也比较熟悉

## Gravitino 实现的 spark-connector
下面我们来进入主题，首先我们来第一个类  `org.apache.gravitino.spark.connector.plugin.GravitinoSparkPlugin`
```
/** The entrypoint for Apache Gravitino Spark connector. */
public class GravitinoSparkPlugin implements SparkPlugin {

  @Override
  public DriverPlugin driverPlugin() {
    return new GravitinoDriverPlugin();
  }

  @Override
  public ExecutorPlugin executorPlugin() {
    return null;
  }
}
```
这个类就是 Gravitino  Spark Connector 的入口类，它实现了 `SparkPlugin` 接口: 这是 Spark 3.0 加入的能力，可以在不改变 Spark 源码的情况下对 Spark 进行增强和扩展。仔细看 `GravitinoSparkPlugin` 只实现了 `driverPlugin()` 方法, 为什么呢？因为 SQL 解析是在 Driver 端去做的，在生成 Logical Plan 和 Physical Plan 时会调用 Catalog 进行元数据解析与管理。那下面我们的重头戏就在这个 `GravitinoDriverPlugin` 。
###GravitinoDriverPlugin
我们来看下这个类的签名：
```
...
import org.apache.spark.api.plugin.DriverPlugin;
...
/**
 * GravitinoDriverPlugin creates GravitinoCatalogManager to fetch catalogs from Apache Gravitino and
 * register Gravitino catalogs to Apache Spark.
 */
public class GravitinoDriverPlugin implements DriverPlugin {
     ...
}
```
它实现了 `DriverPlugin` ，同属于 Spark 3.0 之后的插件体系，会在 spark driver 初始化的进行实例化，创建完之后会调用它定义的 init 方法，下面是初始化 DriverPlugin 代码：
```
...
    //这里的 p 就是 SparkPlugin 实现类的实例
    val driverPlugin = p.driverPlugin()
    if (driverPlugin != null) {
      val name = p.getClass().getName()
      val ctx = new PluginContextImpl(name, sc.env.rpcEnv, sc.env.metricsSystem, sc.conf,
        sc.env.executorId, resources)
      // driverPlugin 构建完之后就紧接着调用 init 方法
      val extraConf = driverPlugin.init(sc, ctx)
      if (extraConf != null) {
        extraConf.asScala.foreach { case (k, v) =>
          sc.conf.set(s"${PluginContainer.EXTRA_CONF_PREFIX}$name.$k", v)
        }
      }
      logInfo(s"Initialized driver component for plugin $name.")
      Some((p.getClass().getName(), driverPlugin, ctx))
    } else {
      None
    }
..
```

### GravitinoDriverPlugin#init
我们来看下代码：
```java
public Map<String, String> init(SparkContext sc, PluginContext pluginContext) {
    SparkConf conf = sc.conf();
    //gravitino 地址
    String gravitinoUri = conf.get(GravitinoSparkConfig.GRAVITINO_URI);
    //gravitino  元数据湖
    String metalake = conf.get(GravitinoSparkConfig.GRAVITINO_METALAKE);
    //校验
    Preconditions.checkArgument(
        StringUtils.isNotBlank(gravitinoUri),
        String.format(
            "%s:%s, should not be empty", GravitinoSparkConfig.GRAVITINO_URI, gravitinoUri));
    Preconditions.checkArgument(
        StringUtils.isNotBlank(metalake),
        String.format(
            "%s:%s, should not be empty", GravitinoSparkConfig.GRAVITINO_METALAKE, metalake));
    //暂时不关心
    this.enableIcebergSupport =
        conf.getBoolean(GravitinoSparkConfig.GRAVITINO_ENABLE_ICEBERG_SUPPORT, false);
    if (enableIcebergSupport) {
      gravitinoDriverExtensions.addAll(gravitinoIcebergExtensions);
    }

   //创建 GravitinoCatalogManager
    this.catalogManager =
        GravitinoCatalogManager.create(
            () -> createGravitinoClient(gravitinoUri, metalake, conf, sc.sparkUser()));
    //从 gravitino 服务端加载关系型 catalogs
    catalogManager.loadRelationalCatalogs();
   
    //这步是重点，向 spark 注册 catalog 
    registerGravitinoCatalogs(conf, catalogManager.getCatalogs());
    registerSqlExtensions(conf);
    return Collections.emptyMap();
  }
```
下面我先来看创建 GravitinoCatalogManager 实例以及从 gravitino 服务端加载关系型 catalogs 的逻辑。
1. 创建 GravitinoCatalogManager
先来看下 `GravitinoCatalogManager##create` 代码:
```
//这里可以看到这就是一个单例,但是没有做线程安全保护
private static GravitinoCatalogManager gravitinoCatalogManager;
//缓存 gravitino catalogs 
private final Cache<String, Catalog> gravitinoCatalogs;
//gravitino 客户端用来和 gravitino 沟通 
private final GravitinoClient gravitinoClient;
//clientBuilder 参数就是创建 Gravitino 客户端的实现
public static GravitinoCatalogManager create(Supplier<GravitinoClient> clientBuilder) {
    //单例实现：判断是否已经创建
    Preconditions.checkState(
        gravitinoCatalogManager == null, "Should not create duplicate GravitinoCatalogManager");
    //若没创建直接创建
    gravitinoCatalogManager = new GravitinoCatalogManager(clientBuilder);
    return gravitinoCatalogManager;
  }

//构造器
private GravitinoCatalogManager(Supplier<GravitinoClient> clientBuilder) {
    this.gravitinoClient = clientBuilder.get();
    // Will not evict catalog by default
    this.gravitinoCatalogs = Caffeine.newBuilder().build();
  }
```
2. 加载关系型的 `catalogs`(GravitinoCatalogManager#getGravitinoCatalogInfo)
```
public void loadRelationalCatalogs() {
    //现获取当前元数据湖下的所有 catalogs
    Catalog[] catalogs = gravitinoClient.listCatalogsInfo();
    Arrays.stream(catalogs)
         //只需要类型为 RELATIONAL 的 catalog
        .filter(catalog -> Catalog.Type.RELATIONAL.equals(catalog.type()))
         //放入缓存中
        .forEach(catalog -> gravitinoCatalogs.put(catalog.name(), catalog));
  }
```
我们再来回顾下 GravitinoDriverPlugin#init 方法：

```java
public Map<String, String> init(SparkContext sc, PluginContext pluginContext) {
    SparkConf conf = sc.conf();
    //gravitino 地址
    String gravitinoUri = conf.get(GravitinoSparkConfig.GRAVITINO_URI);
    //gravitino  元数据湖
    String metalake = conf.get(GravitinoSparkConfig.GRAVITINO_METALAKE);
    //校验
    Preconditions.checkArgument(
        StringUtils.isNotBlank(gravitinoUri),
        String.format(
            "%s:%s, should not be empty", GravitinoSparkConfig.GRAVITINO_URI, gravitinoUri));
    Preconditions.checkArgument(
        StringUtils.isNotBlank(metalake),
        String.format(
            "%s:%s, should not be empty", GravitinoSparkConfig.GRAVITINO_METALAKE, metalake));
    //暂时不关心
    this.enableIcebergSupport =
        conf.getBoolean(GravitinoSparkConfig.GRAVITINO_ENABLE_ICEBERG_SUPPORT, false);
    if (enableIcebergSupport) {
      gravitinoDriverExtensions.addAll(gravitinoIcebergExtensions);
    }

   //创建 GravitinoCatalogManager
    this.catalogManager =
        GravitinoCatalogManager.create(
            () -> createGravitinoClient(gravitinoUri, metalake, conf, sc.sparkUser()));
    //从 gravitino 服务端加载关系型 catalogs
    catalogManager.loadRelationalCatalogs();
   
    //这步是重点，向 spark 注册 catalog  **我们讲到这里了！！！！**
    registerGravitinoCatalogs(conf, catalogManager.getCatalogs());
    registerSqlExtensions(conf);
    return Collections.emptyMap();
  }
```
我们来看下 registerGravitinoCatalogs 方法：
```java
private void registerGravitinoCatalogs(
      SparkConf sparkConf, Map<String, Catalog> gravitinoCatalogs) {
    gravitinoCatalogs
        .entrySet()
        .forEach(
            entry -> {
              String catalogName = entry.getKey();
              Catalog gravitinoCatalog = entry.getValue();
              String provider = gravitinoCatalog.provider();
              if ("lakehouse-iceberg".equals(provider.toLowerCase(Locale.ROOT))
                  && enableIcebergSupport == false) {
                return;
              }
              try {
                //这里才是重点注册的逻辑主要在这
                registerCatalog(sparkConf, catalogName, provider);
              } catch (Exception e) {
                LOG.warn("Register catalog {} failed.", catalogName, e);
              }
            });
  }
```
我们紧接着来看 registerCatalog 方法：
```java
  private void registerCatalog(SparkConf sparkConf, String catalogName, String provider) {
    if (StringUtils.isBlank(provider)) {
      LOG.warn("Skip registering {} because catalog provider is empty.", catalogName);
      return;
    }

    //根据 provider 获取 catalogClassName 
    String catalogClassName = CatalogNameAdaptor.getCatalogName(provider);
    if (StringUtils.isBlank(catalogClassName)) {
      LOG.warn("Skip registering {} because {} is not supported yet.", catalogName, provider);
      return;
    }
     
    
     //这里比较简单就是组装 spark.sql.catalog.catalog-name=com.example.YourCatalogClass 然后设置到 sparkConf 中,看不懂的可以参考我上面第二节提到的 spark multi catalog
    String sparkCatalogConfigName = "spark.sql.catalog." + catalogName;
    Preconditions.checkArgument(
        !sparkConf.contains(sparkCatalogConfigName),
        catalogName + " is already registered to SparkCatalogManager");
    sparkConf.set(sparkCatalogConfigName, catalogClassName);
    
LOG.info("Register {} catalog to Spark catalog manager.", catalogName);
  }
```
我们来看下根据 provider 获取 catalog class name:
```

public static String getCatalogName(String provider) {
   //这个获取的是大版本，我们假设成 3
    int majorVersion = VersionUtils$.MODULE$.majorVersion(sparkVersion());
   //这个是小版本版本，我们假设成 3
    int minorVersion = VersionUtils$.MODULE$.minorVersion(sparkVersion());
    /
    return getCatalogName(provider, majorVersion, minorVersion);
  }
  
  private static String sparkVersion() {
    return package$.MODULE$.SPARK_VERSION();
  }
  
  //假设 provider 是 hive , majorVersion 是 3 ，minorVersion 是 3
  private static String getCatalogName(String provider, int majorVersion, int minorVersion) {
    if (provider.startsWith("jdbc")) {
      return jdbcCatalogNames.get(String.format("%d.%d", majorVersion, minorVersion));
    }
    //因为 provider 是 hive 所以走到这
    String key =
        String.format("%s-%d.%d", provider.toLowerCase(Locale.ROOT), majorVersion, minorVersion);
   //key 是 hive-3.3, 获取的是  org.apache.gravitino.spark.connector.hive.GravitinoHiveCatalogSpark33
    return catalogNames.get(key);
  }

  
private static final Map<String, String> catalogNames =
      ImmutableMap.of(
          "hive-3.3",
          "org.apache.gravitino.spark.connector.hive.GravitinoHiveCatalogSpark33",
          "hive-3.4",
          "org.apache.gravitino.spark.connector.hive.GravitinoHiveCatalogSpark34",
          "hive-3.5",
          "org.apache.gravitino.spark.connector.hive.GravitinoHiveCatalogSpark35",
          "lakehouse-iceberg-3.3",
          "org.apache.gravitino.spark.connector.iceberg.GravitinoIcebergCatalogSpark33",
          "lakehouse-iceberg-3.4",
          "org.apache.gravitino.spark.connector.iceberg.GravitinoIcebergCatalogSpark34",
          "lakehouse-iceberg-3.5",
          "org.apache.gravitino.spark.connector.iceberg.GravitinoIcebergCatalogSpark35",
          "lakehouse-paimon-3.3",
          "org.apache.gravitino.spark.connector.paimon.GravitinoPaimonCatalogSpark33",
          "lakehouse-paimon-3.4",
          "org.apache.gravitino.spark.connector.paimon.GravitinoPaimonCatalogSpark34",
          "lakehouse-paimon-3.5",
          "org.apache.gravitino.spark.connector.paimon.GravitinoPaimonCatalogSpark35");
```
到这里spark 启动初始化的逻辑已经讲完，后续就是实现 Spark 的 `CatalogPlugin` 的实现类初始化了，在第二小节也说了会在解析 SQL 转成 LogicalPlan| PhysicalPlan 用到哪个 catalog 就会初始化哪个 catalog 实例。
下面我们来看看 `GravitinoHiveCatalogSpark33` 这个类，它继承了 `GravitinoHiveCatalog` 而 `GravitinoHiveCatalog` 继承了 `BaseCatalog`  它实现了 Spark 的 `TableCatalog` 接口，现在我们主要来看下初始化方法它在 `BaseCatalog` 里：
```
// The Gravitino catalog client to do schema operations.
  protected Catalog gravitinoCatalogClient;
public void initialize(String name, CaseInsensitiveStringMap options) {
    this.catalogName = name;
   //Gravitino Catalog 实例用来操作元数据
    this.gravitinoCatalogClient = gravitinoCatalogManager.getGravitinoCatalogInfo(name);
    String provider = gravitinoCatalogClient.provider();
    Preconditions.checkArgument(
        StringUtils.isNotBlank(provider), name + " catalog provider is empty");
   
   //这个是 Spark 侧的 Catalog 实例，做真正的读写 IO
   this.sparkCatalog =
        createAndInitSparkCatalog(name, options, gravitinoCatalogClient.properties());
   //spark 和gravitino  属性转换
    this.propertiesConverter = getPropertiesConverter();
    
    //spark transform 与 gravitino partition 、distribution 以及 sort order 转换
    this.sparkTransformConverter = getSparkTransformConverter();
    //类型转换
    this.sparkTypeConverter = getSparkTypeConverter();
    //spark 和 gravitino 表变更转换
    this.sparkTableChangeConverter = getSparkTableChangeConverter(sparkTypeConverter);
  }
```
说到这里基本要讲的都讲完了，我们来做个总结：
1. GravitinoDriverPlugin 初始化逻辑
- 从 SparkConf 中读取 Gravitino 服务地址和元数据湖名称（GRAVITINO_URI 和 GRAVITINO_METALAKE）。

- 创建 GravitinoCatalogManager 单例，负责和 Gravitino 服务端通信。

- 通过 GravitinoCatalogManager 加载 Gravitino 上所有类型为 RELATIONAL 的 Catalog 信息。

- 遍历所有 Catalog，根据其 Provider 通过版本适配选择对应的 Spark Catalog 实现类（如 GravitinoHiveCatalogSpark33）。

- 在 SparkConf 中动态注册这些 Catalog，实现多 Catalog 支持。
2. Catalog 的具体实现
- 每个 Catalog 继承 TableCatalog（间接继承 CatalogPlugin），实现元数据管理接口。

- BaseCatalog 类中实现 initialize，会拿到 Gravitino Catalog 信息，完成 Catalog 客户端和 Spark Catalog 实例的初始化。

- Spark Catalog 负责具体的读写数据操作，而 Gravitino Catalog 负责元数据管理。

我们再来看一下操作表的逻辑，我们选一个改表吧：
```
public Table alterTable(Identifier ident, TableChange... changes) throws NoSuchTableException {
   //现将 spark 的 TableChange 转换成 gravitino 的
    org.apache.gravitino.rel.TableChange[] gravitinoTableChanges =
        Arrays.stream(changes)
            .map(sparkTableChangeConverter::toGravitinoTableChange)
            .toArray(org.apache.gravitino.rel.TableChange[]::new);
    try {
     //这就是负责 IO 的 spark 侧 catalog 这里主要是将缓存置为失效，因为修改了嘛
      sparkCatalog.invalidateTable(ident);
     //调用 gravitino 修改元数据
      org.apache.gravitino.rel.Table gravitinoTable =
          gravitinoCatalogClient
              .asTableCatalog()
              .alterTable(
                  NameIdentifier.of(getDatabase(ident), ident.name()), gravitinoTableChanges);
      org.apache.spark.sql.connector.catalog.Table sparkTable = loadSparkTable(ident);
     //返回 spark 定义的 table 类
      return createSparkTable(
          ident,
          gravitinoTable,
          sparkTable,
          sparkCatalog,
          propertiesConverter,
          sparkTransformConverter,
          sparkTypeConverter);
    } catch (org.apache.gravitino.exceptions.NoSuchTableException e) {
      throw new NoSuchTableException(ident);
    }
  }
```
还是很简单的。


## 总结
本文涵盖的知识点略广，主要涉及到 spark 3.0 以上的 plugin 体系以及 multi catalog 能力，如果知道并理解了这个就会很容易理解这篇文章了。祝大家生活愉快！
