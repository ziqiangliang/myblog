## 什么是 Gravitino 
引用官方文档 `Apache Gravitino  是一个高性能、地理分布式、联邦式元数据湖。它能够直接管理不同数据源、类型及区域的元数据，同时为用户提供数据和 AI 资产的统一元数据访问能力。` 从上面这句话可以看出最主要的就是 **统一元数据访问**。

## 核心价值：统一元数据治理
### 行业痛点
做过大数据开发的同学可能都知道目前数据源很多：关系型数据库以及非关系型数据库等等，例如要想在 Spark 引擎里去查询这些数据源则需要注册不同的 catalog 去查询元数据
### 解决方案
Gravitino 的作用就是把不同数据源管理起来，由 Gravitino 去查询不同的元数据，相当于一层代理
将元数据抽象为 3 层: catalog, schema, (table,topic等):
![image.png](https://upload-images.jianshu.io/upload_images/11859806-39409ba46d958450.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 核心功能
### 统一结构化数据访问
 举个例子：
你们公司现在有个 hive 数仓，数仓下面有 ods, dwd, dws 库。。，库下面有表。你现在就可以将这个 hive 数仓注册成为一个 catalog 名字就叫 hive , 下面的库就是 schema: ods,dws, 库下面的表就是 table: user_info_1d_a 等。那你完全就可以把一个表描述成: `hive.ods.user_info_1d_a`
你的 sql 语句就可以这么写:
```
select * from hive.ods.user_info_1d_a;
```
### 统一非结构化数据访问
#### 文件集
由于 2022 年末 GPT 3 的突出表现，AI 又又火了，各大公司内部都开始疯狂投入精力到 AI 上来，但是 AI 训练需要数据，这些数据大多都是非结构化的，需要被管理，`文件集`的出现就是为了管理这些非结构化的文件。同样文件集也是被抽象为三层结构 `catalog.schema.fileset` 方便大家管理训练文件
#### GVFS 
为了更好的去使用文件集， Gravitino 定义一个文件系统叫 GVFS：它是一个虚拟层，它通过虚拟路径管理文件集中的文件和目录，不用关心他是存储在 HDFS 还是在 s3 上。您可以通过以下方式访问文件或文件夹：
```
gvfs://fileset/${catalog_name}/${schema_name}/${fileset_name}/sub_dir/
```
### 与传统对比
| 场景                | 传统方式                  | Gravitino方案            |
|---------------------|--------------------------|--------------------------|
| 跨数据源查询         | 维护多个Catalog连接       | 统一SQL语法访问          |
| AI训练数据管理       | 手动同步HDFS/S3路径       | GVFS虚拟路径自动映射     |
## 统一权限管理
通过统一元数据实现统一权限管理似乎是手到擒来，事实上的确如此。有了统一元数据的基础，Gravitino 可以将权限都管理起来，目前 Gravitino 的权限模型是 RABC 模型，和大数据常用组件 Ranger 中的 policy 的灵活性相比还是弱了些。

# Gravitino 的核心架构
![image.png](https://upload-images.jianshu.io/upload_images/11859806-63198c94bd5f2a9e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- 第一层是数据应用层，包括数据平台、数据应用组件以及引擎，通过统一处理和治理，都通过这一层接入。
- 第二层是接口层，目前 Gravitino 提供了 REST 统一接口，对外提供元数据的访问能力，并且还提供了 Thrift 和 JDBC 接口，供引擎进行具体接入。
- 第三层是 Gravitino 的核心组件，其数据分类基于 Catalog 进行管理。不同数据源具有不同的 Catalog。整个层级架构基于 MetalLake Catalog Schema 和 Table 的概念。Schema 是我们通常理解中的数据库概念。对于 FileSet，文件管理也基于 Catalog。例如，Hive 是一个 HiveCatalog。Fileset 代表一个独立的文件系统的目录， 也可以理解为一个文件集，则对应 FilesetCatalog
- 最底层是连接层，它连接了不同的数据源。例如，对于 Hive 表，其背后是 Hive MetaStore；对于 Iceberg 表，则是提供的统一 REST Catalog。此外，Gravitino 内部还维护了一套自己的存储系统。不过，需要注意的是，Gravitino 本身并不存储元数据，这部分主要用于维护 Gravitino 内部数据，如 Catalog 和 Meter Lake 等信息。


