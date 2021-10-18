---
title: Mybatis源码解析 Configuration初始化
date: 2021-06-16 16:12
categories: Mybatis源码解析
---
## 前言
本文分享Mybatis的Configuration初始化流程，[配置参考](https://mybatis.org/mybatis-3/zh/configuration.html)。
重点分析XML资源文件解析和功能实现。
# 一、加载配置
Mybatis加载配置文件分为两个步骤：

1. 将文件转换为资源
```java
String resource = "mybatis-config.xml";
// 使用类加载器加载配置文件获取InputStream
InputStream inputStream = inputStream = Resources.getResourceAsStream(resource);
```

2. 将资源解析为Configuration对象
```java
// 正式开始解析
XMLConfigBuilder parser = new XMLConfigBuilder(inputStream, null, null);
return build(parser.parse());
```
# 二、解析Config
解析Configuration的由**XMLConfigBuilder**实现。
## 1. XMLConfigBuilder
![](https://cdn.nlark.com/yuque/__puml/a57785b938ac31bbebf37b3c520d7749.svg#lake_card_v2=eyJ0eXBlIjoicHVtbCIsImNvZGUiOiJAc3RhcnR1bWxcblxuYWJzdHJhY3QgY2xhc3MgQmFzZUJ1aWxkZXJcbmNsYXNzIFhNTENvbmZpZ0J1aWxkZXIgZXh0ZW5kcyBCYXNlQnVpbGRlclxuXG5hYnN0cmFjdCBjbGFzcyBCYXNlQnVpbGRlcntcblx0IyBDb25maWd1cmF0aW9uIGNvbmZpZ3VyYXRpb25cblx0IyBUeXBlQWxpYXNSZWdpc3RyeSB0eXBlQWxpYXNSZWdpc3RyeVxuXHQjIFR5cGVIYW5kbGVyUmVnaXN0cnkgdHlwZUhhbmRsZXJSZWdpc3RyeVxufVxubm90ZSBsZWZ0IG9mIEJhc2VCdWlsZGVyOjpjb25maWd1cmF0aW9uXG4gIOmcgOimgeWwhlhNTOaWh-S7tuS4reeahOmFjee9ruaYoOWwhOWIsOivpeexu-S4rVxuZW5kIG5vdGVcbm5vdGUgbGVmdCBvZiBCYXNlQnVpbGRlcjo6dHlwZUFsaWFzUmVnaXN0cnlcbiAg5Yir5ZCN5rOo5YaM5Zmo77yM55So5LqO5Zyo6YWN572u5Lit6L-b6KGM57yp5YaZ44CCXG5lbmQgbm90ZVxubm90ZSBsZWZ0IG9mIEJhc2VCdWlsZGVyOjp0eXBlSGFuZGxlclJlZ2lzdHJ5XG4gIOexu-Wei-WkhOeQhuWZqO-8jOeUqOS6juWwhkphdmHnsbvlnosgPC0tPiBKREJD57G75Z6L77yM55u45LqS6L2s5o2i44CCXG5lbmQgbm90ZVxuXG5jbGFzcyBYTUxDb25maWdCdWlsZGVye1xuXHQrIENvbmZpZ3VyYXRpb24gcGFyc2UoKVxuXHQtIHZvaWQgcGFyc2VDb25maWd1cmF0aW9uKFhOb2RlIHJvb3QpXG59XG5ub3RlIGxlZnQgb2YgWE1MQ29uZmlnQnVpbGRlcjo6cGFyc2VcbiAg6Kej5p6Q6YWN572u77yM6L-U5ZueQ29uZmlndXJhdGlvblxuZW5kIG5vdGVcbm5vdGUgbGVmdCBvZiBYTUxDb25maWdCdWlsZGVyOjpwYXJzZUNvbmZpZ3VyYXRpb25cbiAg6Kej5p6Q6YWN572u77yM6YWN572u5pig5bCE5YiwQ29uZmlndXJhdGlvblxuZW5kIG5vdGVcbkBlbmR1bWwiLCJ1cmwiOiJodHRwczovL2Nkbi5ubGFyay5jb20veXVxdWUvX19wdW1sL2E1Nzc4NWI5MzhhYzMxYmJlYmYzN2IzYzUyMGQ3NzQ5LnN2ZyIsImlkIjoiQ2x2NDkiLCJtYXJnaW4iOnsidG9wIjp0cnVlLCJib3R0b20iOnRydWV9LCJjYXJkIjoiZGlhZ3JhbSJ9)TypeAliasRegistry：在Configuration中默认创建，[配置参考](https://mybatis.org/mybatis-3/zh/configuration.html#typeAliases)。
TypeHandlerRegistry：在Configuration中默认创建，[配置参考](https://mybatis.org/mybatis-3/zh/configuration.html#typeHandlers)。
​

实际处理XML中**Configuration**节点由**parseConfiguration(...)**方法，我们从这个方法开始。
## 2. 属性（properties）
[配置参考](https://mybatis.org/mybatis-3/zh/configuration.html#properties)，方便编写配置，进行动态替换，配置如下：
```xml
<properties resource="config.properties">
  <property name="driver" value="com.mysql.cj.jdbc.Driver"/>
  <property name="url"    value="jdbc:mysql://localhost:3306/mybatis-test?characterEncoding=utf8&amp;zeroDateTimeBehavior=convertToNull&amp;useSSL=false&amp;useJDBCCompliantTimezoneShift=true&amp;useLegacyDatetimeCode=false&amp;serverTimezone=GMT%2B8&amp;allowMultiQueries=true&amp;allowPublicKeyRetrieval=true"/>
</properties>
```
```java
# config.properties
username = root
password = 123456
```
配置属性有三种方法：

1. 直接在properties节点中定义property。
1. properties标签resource属性。
1. properties标签url属性。
```java
// root.evalNode会将占位符解析成属性
propertiesElement(root.evalNode("properties"));

private void propertiesElement(XNode context) throws Exception {
    if (context != null) {
        // 在properties节点中定义的属性
        Properties defaults = context.getChildrenAsProperties();
        // 配置在资源文件中。
        String resource = context.getStringAttribute("resource");
        // 从网络中获取资源文件
        String url = context.getStringAttribute("url");
        if (resource != null && url != null) {
            throw new BuilderException("The properties element cannot specify both a URL and a resource based property file reference.  Please specify one or the other.");
        }
        if (resource != null) {
            defaults.putAll(Resources.getResourceAsProperties(resource));
        } else if (url != null) {
            defaults.putAll(Resources.getUrlAsProperties(url));
        }
        Properties vars = configuration.getVariables();
        if (vars != null) {
            defaults.putAll(vars);
        }
        // 注意，evalNode调用时会尝试从variables中读取变量
        parser.setVariables(defaults);
        // 设置变量
        configuration.setVariables(defaults);
    }
}
```
实际上在调用**evalNode()**方法时已经用到了**properties**：
```java
// 比如getStringAttribute, attributes = parseAttributes(node);
public String getStringAttribute(String name, Supplier<String> defSupplier) {
    String value = attributes.getProperty(name);
    return value == null ? defSupplier.get() : value;
}

// 解析属性
private Properties parseAttributes(Node n) {
    Properties attributes = new Properties();
    NamedNodeMap attributeNodes = n.getAttributes();
    if (attributeNodes != null) {
        for (int i = 0; i < attributeNodes.getLength(); i++) {
            Node attribute = attributeNodes.item(i);
            // variables
            String value = PropertyParser.parse(attribute.getNodeValue(), variables);
            attributes.put(attribute.getNodeName(), value);
        }
    }
    return attributes;
}

```
## 3. 设置（settings）
[配置参考](https://mybatis.org/mybatis-3/zh/configuration.html#settings)，设置启动 / 关闭功能，配置如下：
```xml
<settings>
  <setting name="cacheEnabled"        value="true"/>
  <setting name="defaultExecutorType" value="SIMPLE"/>
</settings>
```
设置属性太多，以这两个举例：
```java
// 解析XML配置
Properties settings = settingsAsProperties(root.evalNode("settings"));
// 设置VFS的实现
loadCustomVfs(settings);
// 设置自定义日志
loadCustomLogImpl(settings);
// ...
// 配置设置默认属性
settingsElement(settings);

private Properties settingsAsProperties(XNode context) {
    if (context == null) {
        return new Properties();
    }
    // 读取配置信息
    Properties props = context.getChildrenAsProperties();
    // Check that all settings are known to the configuration class
    
    // 验证 Configuration中是否存在setting标签中set name属性方法。
    MetaClass metaConfig = MetaClass.forClass(Configuration.class, localReflectorFactory);
    for (Object key : props.keySet()) {
        if (!metaConfig.hasSetter(String.valueOf(key))) {
            throw new BuilderException("The setting " + key + " is not known.  Make sure you spelled it correctly (case sensitive).");
        }
    }
    return props;
}
```
## 4. 类型别名（typeAliases）
[配置参考](https://mybatis.org/mybatis-3/zh/configuration.html#typeAliases)，设置缩写的别名，配置如下：
```xml
<typeAliases>
  <typeAlias alias="student" type="com.example.entity.StudentEntity"/>
  <package name="com.example.entity"/>
</typeAliases>
```
XMLConfigBuilder UML图中展示了**Configuration**类中有一个成员变量：**TypeAliasRegistry**，我相信大家已经想到了Mybatis是如何做的？
但需要一点注意，别名注册有两种形式：

1. package：该包名下Java Bean都会注册别名（具体请看配置参考）。
1. typeAlias：单个类注册别民。

Mybatis已经帮我注册了一些常用的别名，如：int、long...。
```java
typeAliasesElement(root.evalNode("typeAliases"));

private void typeAliasesElement(XNode parent) {
    if (parent != null) {
        for (XNode child : parent.getChildren()) {
            // 把整个包名下所有对象注册别名
            if ("package".equals(child.getName())) {
                String typeAliasPackage = child.getStringAttribute("name");
                configuration.getTypeAliasRegistry().registerAliases(typeAliasPackage);
            } else {
                // 单个对象注册
                String alias = child.getStringAttribute("alias");
                String type = child.getStringAttribute("type");
                try {
                    Class<?> clazz = Resources.classForName(type);
                    if (alias == null) {
                        typeAliasRegistry.registerAlias(clazz);
                    } else {
                        typeAliasRegistry.registerAlias(alias, clazz);
                    }
                } catch (ClassNotFoundException e) {
                    throw new BuilderException("Error registering typeAlias for '" + alias + "'. Cause: " + e, e);
                }
            }
        }
    }
}

```
## 5. 类型处理器（typeHandlers）
[配置参考](https://mybatis.org/mybatis-3/zh/configuration.html#typeHandlers)，TypeHandler相对复杂，用于Statement参数映射、ResultSet返回处理。将Java Type 与 JDBC Type相互转换，配置如下：
```xml
<typeHandlers>
  <typeHandler handler="com.example.handler.ExampleTypeHandler"/>
</typeHandlers>
```
```xml
@MappedJdbcTypes(JdbcType.VARCHAR)
public class ExampleTypeHandler extends BaseTypeHandler<String> {
    @Override
    public void setNonNullParameter(PreparedStatement ps, int i, String parameter, JdbcType jdbcType) throws SQLException {
        ps.setString(i, parameter);
    }

    @Override
    public String getNullableResult(ResultSet rs, String columnName) throws SQLException {
        return rs.getString(columnName);
    }

    @Override
    public String getNullableResult(ResultSet rs, int columnIndex) throws SQLException {
        return rs.getString(columnIndex);
    }

    @Override
    public String getNullableResult(CallableStatement cs, int columnIndex) throws SQLException {
        return cs.getString(columnIndex);
    }
}
```
和**类型别名**大体相同，同样支持两种方式，注册与**类型别名**相差无几，所以在此省略。
## 6. 对象工厂（objectFactory）
[配置参考](https://mybatis.org/mybatis-3/zh/configuration.html#objectFactory)，**DefaultResultSetHandler**会在处理**结果集**时使用对象工厂创建返回对象，配置如下：
```xml
<objectFactory type="com.example.handler.ExampleObjectFactory">
  <property name="someProperty" value="100"/>
</objectFactory>
```
```java
public class ExampleObjectFactory extends DefaultObjectFactory {
    @Override
    public Object create(Class type) {
        return super.create(type);
    }
    @Override
    public void setProperties(Properties properties) {
        System.out.println(properties);
        super.setProperties(properties);
    }
    @Override
    public <T> boolean isCollection(Class<T> type) {
        return Collection.class.isAssignableFrom(type);
    }
}
// print
// -------------------------------
{someProperty=100}
// -------------------------------
```
**ObjectFactory**比较简单，默认实现为**DefaultObjectFactory**。
```java
private void objectFactoryElement(XNode context) throws Exception {
    if (context != null) {
        String type = context.getStringAttribute("type");
        Properties properties = context.getChildrenAsProperties();
        // 根据类型，创建ExampleObjectFactory
        ObjectFactory factory = (ObjectFactory) resolveClass(type).getDeclaredConstructor().newInstance();
        // 调用set方法
        factory.setProperties(properties);
        // 设置到configuration中
        configuration.setObjectFactory(factory);
    }
}
```
## 7. 插件（plugins）
[配置参考](https://mybatis.org/mybatis-3/zh/configuration.html#plugins)，Mybatis plugin提供的功能非常强大，可以实现相当丰富的功能。如：PageHelper（分页工具），强烈建议参考文档学习，对于一些特定的场景有奇效。配置如下：
```xml
<plugins>
  <plugin interceptor="com.github.pagehelper.PageInterceptor">
  </plugin>
</plugins>
```
以PageHelper为例，注意plugin节点中可以和**objectFactory**一样配置**property**标签，我在这里并没有体现。
```java
private void pluginElement(XNode parent) throws Exception {
    if (parent != null) {
        for (XNode child : parent.getChildren()) {
            String interceptor = child.getStringAttribute("interceptor");
            // plugin -> property
            Properties properties = child.getChildrenAsProperties();
            // 创建plugin 拦截器
            Interceptor interceptorInstance = (Interceptor) resolveClass(interceptor).getDeclaredConstructor().newInstance();
            interceptorInstance.setProperties(properties);
            // 添加到configuration拦截器链中
            configuration.addInterceptor(interceptorInstance);
        }
    }
}
```
## 8. environments（环境配置）  
[配置参考](https://mybatis.org/mybatis-3/zh/configuration.html#environments)，提供三个功能：

1. 配置事务管理器

提供事务管理，比如：**Executor**提供**commit**、**rollback**等事务方法，实际上由**Transaction**执行。

2. 配置数据源

配置数据源，可选择具体数据库连接池（这里默认Mybatis自带）。

3. 支持切换环境

发布不同版本，切换不同的环境。
配置如下：
```xml
<environments default="development">
  <environment id="development">
    <!-- 这里选择JBDC事务管理器 -->
    <transactionManager type="JDBC"/>
    <dataSource type="POOLED">
      <property name="driver" value="${driver}"/>
      <property name="url" value="${url}"/>
      <property name="username" value="${username}"/>
      <property name="password" value="${password}"/>
    </dataSource>
  </environment>
</environments>
```
根据设置默认环境，选择读取哪一个环境。
创建事务管理工厂、数据源工厂，设值进configuration成员中。
```java
private void environmentsElement(XNode context) throws Exception {
    if (context != null) {
        if (environment == null) {
            // 获取当前环境名称
            environment = context.getStringAttribute("default");
        }
        for (XNode child : context.getChildren()) {
            String id = child.getStringAttribute("id");
            // 判断当前默认环境 和 environmentId是否相同 
            if (isSpecifiedEnvironment(id)) {
                // 解析事务管理，创建TransactionFactory实例
                TransactionFactory txFactory = transactionManagerElement(child.evalNode("transactionManager"));
                // 解析数据源
                DataSourceFactory dsFactory = dataSourceElement(child.evalNode("dataSource"));
                DataSource dataSource = dsFactory.getDataSource();
                // 构建环境
                Environment.Builder environmentBuilder = new Environment.Builder(id)
                    .transactionFactory(txFactory)
                    .dataSource(dataSource);
                // 赋值
                configuration.setEnvironment(environmentBuilder.build());
            }
        }
    }
}
```
## 9. databaseIdProvider（数据库厂商标识）    
[配置参考](https://mybatis.org/mybatis-3/zh/configuration.html#environments)，根据不同数据库，可以选择不同的MappedStatement，不是很好用。配置如下：
```xml
<databaseIdProvider type="DB_VENDOR">
  <property name="MySQL" value="mysql"/>
  <property name="Oracle" value="oracle"/>
</databaseIdProvider>

<select id="selectOne" resultType="com.example.entity.StudentEntity" databaseId="mysql">
  select id, name from student
</select>
<select id="selectOne" resultType="com.example.entity.StudentEntity" databaseId="oracle">
  select id, name from student
</select>
```
只能设置单个数据源databaseId，如果想用多数据源，得创建多个SqlSessionFactory做切换。代码如下：
```java
databaseIdProviderElement(root.evalNode("databaseIdProvider"));
private void databaseIdProviderElement(XNode context) throws Exception {
    DatabaseIdProvider databaseIdProvider = null;
    if (context != null) {
        String type = context.getStringAttribute("type");
        // awful patch to keep backward compatibility
        if ("VENDOR".equals(type)) {
            type = "DB_VENDOR";
        }
        Properties properties = context.getChildrenAsProperties();
        // 创建VendorDatabaseIdProvider对象
        databaseIdProvider = (DatabaseIdProvider) resolveClass(type).getDeclaredConstructor().newInstance();
        databaseIdProvider.setProperties(properties);
    }
    Environment environment = configuration.getEnvironment();
    if (environment != null && databaseIdProvider != null) {
        // 设置当前数据源 对于databaseId
        String databaseId = databaseIdProvider.getDatabaseId(environment.getDataSource());
        configuration.setDatabaseId(databaseId);
    }
  }
```
## 10. mappers（映射器）
[配置参考](https://mybatis.org/mybatis-3/zh/configuration.html#mappers)，告诉Mybatis去哪里寻找Mapper文件进行注册。配置如下：
```xml
<mappers>
  <mapper resource="StudentMapper.xml"/>
  <!-- 官网用例 -->
  <mapper url="file:///var/mappers/BlogMapper.xml"/>
  <mapper class="org.mybatis.builder.BlogMapper"/>
  <package name="org.mybatis.builder"/>
</mappers>
```
## 11. autoMapping（自动映射）
可以区分为两大类：

1. XML：resource（资源路径）、url（URL）。
1. Annotation：class（类路径）、package（包名下）。
```java
mapperElement(root.evalNode("mappers"));
private void mapperElement(XNode parent) throws Exception {
    if (parent != null) {
        for (XNode child : parent.getChildren()) {
            // 1. 注解模式：把整个包名下注册Mapper
            if ("package".equals(child.getName())) {
                String mapperPackage = child.getStringAttribute("name");
                configuration.addMappers(mapperPackage);
            } else {
                String resource = child.getStringAttribute("resource");
                String url = child.getStringAttribute("url");
                String mapperClass = child.getStringAttribute("class");
                // 2. XML 资源目录
                if (resource != null && url == null && mapperClass == null) {
                    ErrorContext.instance().resource(resource);
                    InputStream inputStream = Resources.getResourceAsStream(resource);
                    XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, resource, configuration.getSqlFragments());
                    mapperParser.parse();
                } 
                // 3. XML URL
                else if (resource == null && url != null && mapperClass == null) {
                    ErrorContext.instance().resource(url);
                    InputStream inputStream = Resources.getUrlAsStream(url);
                    XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, url, configuration.getSqlFragments());
                    mapperParser.parse();
                } 
                // 4. 注解 单个
                else if (resource == null && url == null && mapperClass != null) {
                    Class<?> mapperInterface = Resources.classForName(mapperClass);
                    configuration.addMapper(mapperInterface);
                } else {
                    throw new BuilderException("A mapper element may only specify a url, resource or class, but not more than one.");
                }
            }
        }
    }
}
```
到此处，整个MybatisConfig全部完成工作，下面开始解析Mapper。
# 三、解析Mapper
## 1. XMLMapperBuilder
![](https://cdn.nlark.com/yuque/__puml/9ddf552a8aec9d85f16a7cbbdc9cdb64.svg#lake_card_v2=eyJ0eXBlIjoicHVtbCIsImNvZGUiOiJAc3RhcnR1bWxcblxuYWJzdHJhY3QgY2xhc3MgQmFzZUJ1aWxkZXJcbmNsYXNzIFhNTE1hcHBlckJ1aWxkZXIgZXh0ZW5kcyBCYXNlQnVpbGRlciBcbmNsYXNzIE1hcHBlckJ1aWxkZXJBc3Npc3RhbnRcbm5vdGUgbGVmdDogTWFwcGVy5p6E5bu65Yqp5omL77yM55So5LqO5LiA5Lqb5aSE55CGTWFwcGVy5qCH562-55qE5bel5L2c77yM5bCG5aSE55CG57uT5p6c6K6-572u6L-bQ29uZmlndXJhdGlvbuS4reOAglxuXG5CYXNlQnVpbGRlciA8fC0tIE1hcHBlckJ1aWxkZXJBc3Npc3RhbnRcblhNTE1hcHBlckJ1aWxkZXIgKi0tIE1hcHBlckJ1aWxkZXJBc3Npc3RhbnRcblxuYWJzdHJhY3QgY2xhc3MgQmFzZUJ1aWxkZXJ7XG5cdCMgQ29uZmlndXJhdGlvbiBjb25maWd1cmF0aW9uXG5cdCMgVHlwZUFsaWFzUmVnaXN0cnkgdHlwZUFsaWFzUmVnaXN0cnlcblx0IyBUeXBlSGFuZGxlclJlZ2lzdHJ5IHR5cGVIYW5kbGVyUmVnaXN0cnlcbn1cblxuY2xhc3MgWE1MTWFwcGVyQnVpbGRlcntcblx0KyB2b2lkIHBhcnNlKClcblx0LSB2b2lkIGNvbmZpZ3VyYXRpb25FbGVtZW50KFhOb2RlIHJvb3QpXG59XG5ub3RlIGxlZnQgb2YgQmFzZUJ1aWxkZXI6OmNvbmZpZ3VyYXRpb25cbiAg6ZyA6KaB5bCGWE1M5paH5Lu25Lit55qE6YWN572u5pig5bCE5Yiw6K-l57G75LitXG5lbmQgbm90ZVxubm90ZSBsZWZ0IG9mIEJhc2VCdWlsZGVyOjp0eXBlQWxpYXNSZWdpc3RyeVxuICDliKvlkI3ms6jlhozlmajvvIznlKjkuo7lnKjphY3nva7kuK3ov5vooYznvKnlhpnjgIJcbmVuZCBub3RlXG5ub3RlIGxlZnQgb2YgQmFzZUJ1aWxkZXI6OnR5cGVIYW5kbGVyUmVnaXN0cnlcbiAg57G75Z6L5aSE55CG5Zmo77yM55So5LqO5bCGSmF2Yeexu-WeiyA8LS0-IEpEQkPnsbvlnovvvIznm7jkupLovazmjaLjgIJcbmVuZCBub3RlXG5ub3RlIGxlZnQgb2YgWE1MTWFwcGVyQnVpbGRlcjo6cGFyc2VcbiAg6Kej5p6Q6YWN572uXG5lbmQgbm90ZVxubm90ZSBsZWZ0IG9mIFhNTE1hcHBlckJ1aWxkZXI6OmNvbmZpZ3VyYXRpb25FbGVtZW50XG4gIOino-aekOmFjee9ru-8jOmFjee9ruaYoOWwhOWIsENvbmZpZ3VyYXRpb25cbmVuZCBub3RlXG5AZW5kdW1sIiwidXJsIjoiaHR0cHM6Ly9jZG4ubmxhcmsuY29tL3l1cXVlL19fcHVtbC85ZGRmNTUyYThhZWM5ZDg1ZjE2YTdjYmJkYzljZGI2NC5zdmciLCJpZCI6Im5VNm9FIiwibWFyZ2luIjp7InRvcCI6dHJ1ZSwiYm90dG9tIjp0cnVlfSwiY2FyZCI6ImRpYWdyYW0ifQ==)**parse**方法与XMLConfigBuilder有所不同。代码如下：
```java
public void parse() {
    // resource 是否加载过
    if (!configuration.isResourceLoaded(resource)) {
        // 解析Mapper标签
        configurationElement(parser.evalNode("/mapper"));
        // 添加到加载后集合
        configuration.addLoadedResource(resource);
        // 主要责任将Mapper加载到configuration，Mapper注册器中。
        bindMapperForNamespace();
    }
	
    // 错误重新解析，有些先后初始化顺序会导致解析失败，此时在重新解析一遍。
    parsePendingResultMaps();
    parsePendingCacheRefs();
    parsePendingStatements();
}
```
## 2. cache（缓存）
[配置参考](https://mybatis.org/mybatis-3/zh/sqlmap-xml.html#cache)，Mybatis一级缓存默认开启，二级缓存需要手动配置开启。
区别：
一级缓存：作用域Statement，多个Statement不共享。
二级缓存：作用域namespace，同一个namespace中Statement共享同一个缓存。有缓存淘汰机制，支持定制化操作。
配置如下：
```xml
<cache/>
```
二级缓存使用条件较为苛刻，一般不启动，以最简单的默认配置进行分析。
```java
cacheElement(context.evalNode("cache"));
private void cacheElement(XNode context) {
    if (context != null) {
        // 获取缓存类型，这里以默认为例：PERPETUAL
        String type = context.getStringAttribute("type", "PERPETUAL");
        // 从别名注册器中找到该名称对应的类型
        Class<? extends Cache> typeClass = typeAliasRegistry.resolveAlias(type);
        // 获取缓存获取策略， 默认LRU
        String eviction = context.getStringAttribute("eviction", "LRU");
        Class<? extends Cache> evictionClass = typeAliasRegistry.resolveAlias(eviction);
        // 刷新间隔
        Long flushInterval = context.getLongAttribute("flushInterval");
        // 引用数目
        Integer size = context.getIntAttribute("size");
        // 只读
        boolean readWrite = !context.getBooleanAttribute("readOnly", false);
        boolean blocking = context.getBooleanAttribute("blocking", false);
        Properties props = context.getChildrenAsProperties();
        // 使用助手创建缓存对象，并放到Configuration中，同时放到currentCache成员变量中，方便解析Statement使用。
        builderAssistant.useNewCache(typeClass, evictionClass, flushInterval, size, readWrite, blocking, props);
    }
}
```
## 3. cache-ref（缓存引用）
[配置参考](https://mybatis.org/mybatis-3/zh/sqlmap-xml.html#cache-ref)，使多个namespace共用一个cache。有几个要注意的地方：

1. cache配置会覆盖cache-ref（由于执行顺序cache-ref总是被先执行）
1. 如果使用cache-ref，如果引用Mapper在此时还没有初始化，解析cache-ref、解析Statement将在后续的错误尝试中执行。

配置如下：
```xml
<cache-ref namespace="com.example.mapper.TestMapper"/>
```
涉及到缓存依赖问题，如果依赖的缓存还没有创建怎么办？
Mybatis选择在后续处理，而不是JVM / SpringBean 加载类时进行交叉加载。
```java
cacheRefElement(context.evalNode("cache-ref"));
private void cacheRefElement(XNode context) {
    if (context != null) {
        // 添加configuration
        configuration.addCacheRef(builderAssistant.getCurrentNamespace(), context.getStringAttribute("namespace"));
        CacheRefResolver cacheRefResolver = new CacheRefResolver(builderAssistant, context.getStringAttribute("namespace"));
        try {
            // 解析cache-ref
            cacheRefResolver.resolveCacheRef();
        } catch (IncompleteElementException e) {
            configuration.addIncompleteCacheRef(cacheRefResolver);
        }
    }
}

// 跳过了一些方法
public Cache resolveCacheRef() {
    if (namespace == null) {
        throw new BuilderException("cache-ref element requires a namespace attribute.");
    }
    try {
        // 未解决缓存引用 true，需要后续处理
        unresolvedCacheRef = true;
        Cache cache = configuration.getCache(namespace);
        if (cache == null) {
            throw new IncompleteElementException("No cache for namespace '" + namespace + "' could be found.");
        }
        // 这个字段用于后续statement处理缓存
        currentCache = cache;
         // 缓存引用已存在，必须要错误处理
        unresolvedCacheRef = false;
        return cache;
    } catch (IllegalArgumentException e) {
        throw new IncompleteElementException("No cache for namespace '" + namespace + "' could be found.", e);
    }
  
}
```
## 4. resultMap（结果映射）
[配置参考](https://mybatis.org/mybatis-3/zh/sqlmap-xml.html#Result_Maps)，查询返回结果集进行映射。支持复杂结果处理，有四种方式：

1. constructor： 调用构造方法
1. association： 一对一
1. collection: 一对多
1. discriminator： 根据匹配字段的value，使用结果值来决定使用哪个 resultMapresult
### 1. 支持功能
默认的子节点：
```xml
<id     property="id"   column="id"/>
<result property="name" column="name"/>
```
**id**：一个 ID 结果；标记出作为 ID 的结果可以帮助提高整体性能
**result**：注入到字段或 JavaBean 属性的普通结果
#### 1.  constructor
```xml
<resultMap id="map" type="com.example.entity.StudentEntity">
  <constructor>
    <idArg column="id"   name="id"   javaType="long"   />
    <arg   column="name" name="name" javaType="string" />
  </constructor>
</resultMap>

```
constructor：拥有idArg、arg两个子节点，
在对结果集进行映射后，会调用构造方法进行创建对象。
需要注意一点：如果使用了该功能，java一般情况下反射是无法获取到形参名称，有两种解决方案：

1. 添加 @Param 注解
```java
public StudentEntity(@Param("id") Long id, @Param("name") String name) {
    this.id = id;
    this.name = name;
}

```

2. 开启编译选项**-parameter**
```xml
<plugins>
  <plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-compiler-plugin</artifactId>
    <configuration>
      <source>1.8</source>
      <target>1.8/</target>
      <compilerArgs>
        <arg>-parameters</arg>
      </compilerArgs>
    </configuration>
  </plugin>
</plugins>
```
#### 2. association
江湖人称：一对一关联关系，可嵌套。
```xml
<association property="child" javaType="com.example.entity.T4Entity">
  <id     property="id"       column="id5"/>
  <result property="username" column="name5"/>
</association>
<!ELEMENT association (constructor?,id*,result*,association*,collection*, discriminator?)>
```
#### 3. collection
江湖人称：一对多关联关系，可嵌套。
```xml
<collection property="childList" ofType="com.example.entity.T5Entity">
  <result property="id"        column="id5"/>
  <result property="username"  column="name5"/>
</collection>
```
#### 4.discriminator
监听器：根据字段的值case匹配的value返回不同的对象。
```xml
<discriminator javaType="long" column="id">
  <case value="3" resultType="com.example.entity.T1Entity"/>
  <case value="2" resultType="com.example.entity.T2Entity"/>
  <case value="1" resultMap ="map" />
</discriminator>
```
### 2.功能解析
#### 1. 解析resultMap
```java
  private ResultMap resultMapElement(XNode resultMapNode, List<ResultMapping> additionalResultMappings, Class<?> enclosingType) {
    ErrorContext.instance().activity("processing " + resultMapNode.getValueBasedIdentifier());
    // 获取标注的type名称
    String type = resultMapNode.getStringAttribute("type",
        resultMapNode.getStringAttribute("ofType",
            resultMapNode.getStringAttribute("resultType",
                resultMapNode.getStringAttribute("javaType"))));
    // 根据type名称获取class对象，因为可以缩写，需要别名注册器查询。
    Class<?> typeClass = resolveClass(type);
    if (typeClass == null) {
      // 处理没填resultMap
      typeClass = inheritEnclosingType(resultMapNode, enclosingType);
    }
    // 鉴定器，用来选择返回值的类型
    Discriminator discriminator = null;
    // 核心类，返回映射。resultSetHandler就是用这个来把结果集转换成指定的类型
    List<ResultMapping> resultMappings = new ArrayList<>(additionalResultMappings);
    // 开始解析子节点
    List<XNode> resultChildren = resultMapNode.getChildren();
    for (XNode resultChild : resultChildren) {
      // 解析构造方法
      if ("constructor".equals(resultChild.getName())) {
        processConstructorElement(resultChild, typeClass, resultMappings);
      }
      // 解析鉴定器
      else if ("discriminator".equals(resultChild.getName())) {
        discriminator = processDiscriminatorElement(resultChild, typeClass, resultMappings);
      }
      // id、result、collection
      else {
        List<ResultFlag> flags = new ArrayList<>();
        if ("id".equals(resultChild.getName())) {
          flags.add(ResultFlag.ID);
        }
        //构建resultMapping,添加到集合中
        resultMappings.add(buildResultMappingFromContext(resultChild, typeClass, flags));
      }
    }
    String id = resultMapNode.getStringAttribute("id",
            resultMapNode.getValueBasedIdentifier());
    // 有没有继承
    String extend = resultMapNode.getStringAttribute("extends");
    // 有没有自动映射
    Boolean autoMapping = resultMapNode.getBooleanAttribute("autoMapping");
    ResultMapResolver resultMapResolver = new ResultMapResolver(builderAssistant, id, typeClass, extend, discriminator, resultMappings, autoMapping);
    try {
      return resultMapResolver.resolve();
    } catch (IncompleteElementException e) {
      configuration.addIncompleteResultMap(resultMapResolver);
      throw e;
    }
  }
```
#### 2. 处理constructor
```java
  private void processConstructorElement(XNode resultChild, Class<?> resultType, List<ResultMapping> resultMappings) {
    List<XNode> argChildren = resultChild.getChildren();
    // 遍历constructor下元素
    for (XNode argChild : argChildren) {
      List<ResultFlag> flags = new ArrayList<>();
      flags.add(ResultFlag.CONSTRUCTOR);
      if ("idArg".equals(argChild.getName())) {
        flags.add(ResultFlag.ID);
      }
      // 添加到集合中
      resultMappings.add(buildResultMappingFromContext(argChild, resultType, flags));
    }
  }
```
#### 3. 处理 / 构建鉴别器
```java
  private Discriminator processDiscriminatorElement(XNode context, Class<?> resultType, List<ResultMapping> resultMappings) {
    String column = context.getStringAttribute("column");
    String javaType = context.getStringAttribute("javaType");
    String jdbcType = context.getStringAttribute("jdbcType");
    String typeHandler = context.getStringAttribute("typeHandler");
    
    // resolveClass 基本都是使用别名注册器获取
    Class<?> javaTypeClass = resolveClass(javaType);
    Class<? extends TypeHandler<?>> typeHandlerClass = resolveClass(typeHandler);
      
    JdbcType jdbcTypeEnum = resolveJdbcType(jdbcType);
    Map<String, String> discriminatorMap = new HashMap<>();
      
    // 解析case元素
    // case元素同样可以有id 、result
    for (XNode caseChild : context.getChildren()) {
      String value = caseChild.getStringAttribute("value");
      // 如果case元素的属性是resultMap, 会再次调用resultMapElement方法（注意这里会递归调用，这里很重要！！！）
      // 这个方法请看4
      String resultMap = caseChild.getStringAttribute("resultMap", processNestedResultMappings(caseChild, resultMappings, resultType));
      discriminatorMap.put(value, resultMap);
    }
    // 构建一个鉴别器
    return builderAssistant.buildDiscriminator(resultType, column, javaTypeClass, jdbcTypeEnum, typeHandlerClass, discriminatorMap);
  }
```
#### 4. 处理 id / result / collection
```java
private ResultMapping buildResultMappingFromContext(XNode context, Class<?> resultType, List<ResultFlag> flags) {
    String property;
    if (flags.contains(ResultFlag.CONSTRUCTOR)) {
        property = context.getStringAttribute("name");
    } else {
        property = context.getStringAttribute("property");
    }
    String column = context.getStringAttribute("column");
    String javaType = context.getStringAttribute("javaType");
    String jdbcType = context.getStringAttribute("jdbcType");
    String nestedSelect = context.getStringAttribute("select");
    // 如果元素属性同样使用了resultMap, 也会再次调用resultMapElement方法（注意这里会递归调用，这里很重要！！！）
    String nestedResultMap = context.getStringAttribute("resultMap", () -> processNestedResultMappings(context, Collections.emptyList(), resultType));
    String notNullColumn = context.getStringAttribute("notNullColumn");
    String columnPrefix = context.getStringAttribute("columnPrefix");
    String typeHandler = context.getStringAttribute("typeHandler");
    String resultSet = context.getStringAttribute("resultSet");
    String foreignColumn = context.getStringAttribute("foreignColumn");
    boolean lazy = "lazy".equals(context.getStringAttribute("fetchType", configuration.isLazyLoadingEnabled() ? "lazy" : "eager"));

    // 别名注册器获取
    Class<?> javaTypeClass = resolveClass(javaType);
    Class<? extends TypeHandler<?>> typeHandlerClass = resolveClass(typeHandler);

    JdbcType jdbcTypeEnum = resolveJdbcType(jdbcType);
    return builderAssistant.buildResultMapping(resultType, property, column, javaTypeClass, jdbcTypeEnum, nestedSelect, nestedResultMap, notNullColumn, columnPrefix, typeHandlerClass, flags, resultSet, foreignColumn, lazy);
}

// 处理嵌套返回映射
private String processNestedResultMappings(XNode context, List<ResultMapping> resultMappings, Class<?> enclosingType) {
    if (Arrays.asList("association", "collection", "case").contains(context.getName())
        && context.getStringAttribute("select") == null) {
        validateCollection(context, enclosingType);
        // 这里会递归调用
        ResultMap resultMap = resultMapElement(context, resultMappings, enclosingType);
        return resultMap.getId();
    }
    return null;
}
```
#### 5. 构建ResultMapping
不必太过研究，只需要有哪些成员有个印象。
```java
public ResultMapping buildResultMapping(
    Class<?> resultType,
    String property,
    String column,
    Class<?> javaType,
    JdbcType jdbcType,
    // 需要select执行 MappedStatement
    String nestedSelect,
    // 需要注意一下，resultSetHandler解析需要嵌套执行
    String nestedResultMap,
    String notNullColumn,
    String columnPrefix,
    Class<? extends TypeHandler<?>> typeHandler,
    List<ResultFlag> flags,
    String resultSet,
    String foreignColumn,
    boolean lazy) {
    Class<?> javaTypeClass = resolveResultJavaType(resultType, property, javaType);
    TypeHandler<?> typeHandlerInstance = resolveTypeHandler(javaTypeClass, typeHandler);
    List<ResultMapping> composites;
    if ((nestedSelect == null || nestedSelect.isEmpty()) && (foreignColumn == null || foreignColumn.isEmpty())) {
        composites = Collections.emptyList();
    } else {
        composites = parseCompositeColumnName(column);
    }
    return new ResultMapping.Builder(configuration, property, column, javaTypeClass)
        .jdbcType(jdbcType)
        .nestedQueryId(applyCurrentNamespace(nestedSelect, true))
        .nestedResultMapId(applyCurrentNamespace(nestedResultMap, true))
        .resultSet(resultSet)
        .typeHandler(typeHandlerInstance)
        .flags(flags == null ? new ArrayList<>() : flags)
        .composites(composites)
        .notNullColumns(parseMultipleColumnNames(notNullColumn))
        .columnPrefix(columnPrefix)
        .foreignColumn(foreignColumn)
        .lazy(lazy)
        .build();
}
```
### 3. 发现问题
当**constructor**元素和**discriminator case**元素中**resultMap**属性同时使用，会导致创建**resultMap**, 获取构造方法名称没有类型导致NPE。
已经提交PR到[mybatis github](https://github.com/mybatis/mybatis-3)，目前还没有同意。
[https://github.com/mybatis/mybatis-3/pull/2353](https://github.com/mybatis/mybatis-3/pull/2353)
## 5. sql
[配置参考](https://mybatis.org/mybatis-3/zh/sqlmap-xml.html#sql)，可被其它语句引用的可重用语句块。
```java
<sql id="userColumns"> ${alias}.id,${alias}.username,${alias}.password </sql>
```
一般使用都是存放数据库的所有字段：
```xml
<sql id="baseColumn">
  t1.id, t1.account, t1.nick_name, t1.phone, t1.email, t1.pwd_reset_time, t1.avatar_name, t1.avatar_path, t1.status
</sql>
```
代码如下：
```java
// 存放当前数据库厂商sql
private final Map<String, XNode> sqlFragments;

private void sqlElement(List<XNode> list) {
    if (configuration.getDatabaseId() != null) {
        sqlElement(list, configuration.getDatabaseId());
    }
    sqlElement(list, null);
}

private void sqlElement(List<XNode> list, String requiredDatabaseId) {
    for (XNode context : list) {
        // 隔离数据库厂商
        String databaseId = context.getStringAttribute("databaseId");
        String id = context.getStringAttribute("id");
        // id = namespace + "." + sql.id
        id = builderAssistant.applyCurrentNamespace(id, false);
        // 判断当前数据库厂商和 sql属性databaseId 是否匹配
        if (databaseIdMatchesCurrent(id, databaseId, requiredDatabaseId)) {
            // 存放到sqlFragments
            sqlFragments.put(id, context);
        }
    }
}
```
# 四、解析Statement
[配置参考](https://mybatis.org/mybatis-3/zh/sqlmap-xml.html#select)，解析Statement是在Mapper中解析的，由于篇幅过长，并且Statement很重要所有单独开一个小结。
**Statement**：select、insert、update、delete四大标签，对于执行器来说只有**Query**和**Update**方法。具体的配置就不在阐述了。
## 1. 解析CRUD元素
```java
buildStatementFromContext(context.evalNodes("select|insert|update|delete"));
```
## 2. 解析Statement节点
```java
public void parseStatementNode() {
    String id = context.getStringAttribute("id");
    String databaseId = context.getStringAttribute("databaseId");

    if (!databaseIdMatchesCurrent(id, databaseId, this.requiredDatabaseId)) {
        return;
    }

    String nodeName = context.getNode().getNodeName();
    // 区分 C、R、U、D 类型
    SqlCommandType sqlCommandType = SqlCommandType.valueOf(nodeName.toUpperCase(Locale.ENGLISH));
    boolean isSelect = sqlCommandType == SqlCommandType.SELECT;
    // 是否刷新缓存，默认select不需要刷新，其他都需要
    boolean flushCache = context.getBooleanAttribute("flushCache", !isSelect);
    // 是否使用cache
    boolean useCache = context.getBooleanAttribute("useCache", isSelect);
    // 一对多，collection 结果的笛卡尔积是否有序，可以减少缓存的对象。
    boolean resultOrdered = context.getBooleanAttribute("resultOrdered", false);

    // Include Fragments before parsing
    XMLIncludeTransformer includeParser = new XMLIncludeTransformer(configuration, builderAssistant);
    includeParser.applyIncludes(context.getNode());


    // 参数类型
    String parameterType = context.getStringAttribute("parameterType");
    Class<?> parameterTypeClass = resolveClass(parameterType);

    String lang = context.getStringAttribute("lang");
    LanguageDriver langDriver = getLanguageDriver(lang);

    // selectKey支持，可以查询某些数据，再set到insert / update语句中。
    // 解析完成后需要删除 <selectKey> 元素，不参与sqlSource解析
    // Parse selectKey after includes and remove them.
    processSelectKeyNodes(id, parameterTypeClass, langDriver);

    // Parse the SQL (pre: <selectKey> and <include> were parsed and removed)
    KeyGenerator keyGenerator;
    String keyStatementId = id + SelectKeyGenerator.SELECT_KEY_SUFFIX;
    keyStatementId = builderAssistant.applyCurrentNamespace(keyStatementId, true);
    if (configuration.hasKeyGenerator(keyStatementId)) {
        keyGenerator = configuration.getKeyGenerator(keyStatementId);
    } else {
        keyGenerator = context.getBooleanAttribute("useGeneratedKeys",
                                                   configuration.isUseGeneratedKeys() && SqlCommandType.INSERT.equals(sqlCommandType))
            ? Jdbc3KeyGenerator.INSTANCE : NoKeyGenerator.INSTANCE;
    }

    // 重要！, 用于解析预处理sql, 参数。这里不会验证sql是否写对了。 
    SqlSource sqlSource = langDriver.createSqlSource(configuration, context, parameterTypeClass);
    // 类型，直接sql、预处理、存储过程
    StatementType statementType = StatementType.valueOf(context.getStringAttribute("statementType", StatementType.PREPARED.toString()));
    Integer fetchSize = context.getIntAttribute("fetchSize");
    Integer timeout = context.getIntAttribute("timeout");
    // 马上弃用了
    String parameterMap = context.getStringAttribute("parameterMap");
    String resultType = context.getStringAttribute("resultType");
    Class<?> resultTypeClass = resolveClass(resultType);
    String resultMap = context.getStringAttribute("resultMap");
    String resultSetType = context.getStringAttribute("resultSetType");
    ResultSetType resultSetTypeEnum = resolveResultSetType(resultSetType);
    if (resultSetTypeEnum == null) {
        resultSetTypeEnum = configuration.getDefaultResultSetType();
    }

    String keyProperty = context.getStringAttribute("keyProperty");
    String keyColumn = context.getStringAttribute("keyColumn");
    String resultSets = context.getStringAttribute("resultSets");

    // 使用助手，构建MappedStatement，添加到configuration中。
    builderAssistant.addMappedStatement(id, sqlSource, statementType, sqlCommandType,
                                        fetchSize, timeout, parameterMap, parameterTypeClass, resultMap, resultTypeClass,
                                        resultSetTypeEnum, flushCache, useCache, resultOrdered,
                                        keyGenerator, keyProperty, keyColumn, databaseId, langDriver, resultSets);
}

// 只需要注意一下cache
public MappedStatement addMappedStatement(
    String id,
    SqlSource sqlSource,
    StatementType statementType,
    SqlCommandType sqlCommandType,
    Integer fetchSize,
    Integer timeout,
    String parameterMap,
    Class<?> parameterType,
    String resultMap,
    Class<?> resultType,
    ResultSetType resultSetType,
    boolean flushCache,
    boolean useCache,
    boolean resultOrdered,
    KeyGenerator keyGenerator,
    String keyProperty,
    String keyColumn,
    String databaseId,
    LanguageDriver lang,
    String resultSets) {

    if (unresolvedCacheRef) {
        throw new IncompleteElementException("Cache-ref not yet resolved");
    }

    id = applyCurrentNamespace(id, false);
    boolean isSelect = sqlCommandType == SqlCommandType.SELECT;

    MappedStatement.Builder statementBuilder = new MappedStatement.Builder(configuration, id, sqlSource, sqlCommandType)
        .resource(resource)
        .fetchSize(fetchSize)
        .timeout(timeout)
        .statementType(statementType)
        .keyGenerator(keyGenerator)
        .keyProperty(keyProperty)
        .keyColumn(keyColumn)
        .databaseId(databaseId)
        .lang(lang)
        .resultOrdered(resultOrdered)
        .resultSets(resultSets)
        .resultMaps(getStatementResultMaps(resultMap, resultType, id))
        .resultSetType(resultSetType)
        .flushCacheRequired(valueOrDefault(flushCache, !isSelect))
        .useCache(valueOrDefault(useCache, isSelect))
        // 这个currentCache就是当前namespace的缓存，上文提到过
        // currentCache是整个namespace里的所有statement共享
        .cache(currentCache);
    
    // 弃用了，会在后续的版本中删除
    ParameterMap statementParameterMap = getStatementParameterMap(parameterMap, parameterType, id);
    if (statementParameterMap != null) {
        statementBuilder.parameterMap(statementParameterMap);
    }

    MappedStatement statement = statementBuilder.build();
    configuration.addMappedStatement(statement);
    return statement;
}
```
# 五、总结
照着官网走了一遍解析配置的流程，总体下来写的不是很满意，太粗糙了。  
但是大部分常用的功能还是研究了，resultMap交叉解析这一块花了大功夫研究，遇到了很多问题，还碰见了Mybatis的BUG。  
语言表达能力、写作能力有待训练，下面着重研究缓存和结果集解析功能。   
后续可能会修改
