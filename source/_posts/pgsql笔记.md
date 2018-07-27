---
title: pgsql笔记
date: 2018-03-18 21:19:08
type: "tags"
tags: [pgsql]
---

> 最近项目中用到pgsql，遇到些麻烦事，记录一下。

<!--more-->

### pgsql array 类型

不知道大家对于pgsql的Array类型是否用的频繁，我在用Mybatis做持久层的时候，``ARRAY``类型对应的是``java.sql.Array``。

这时候问题就来了，在写业务代码的时候发现java类型不能直接转换成Array的实现类，这就头痛了。

翻看了JDK的文档，``Connection``接口中有：

| Modifier and Type| Method | Description|
| :-------- | --------:|  --------:|
|Array|	createArrayOf(String typeName, Object[] elements) |	Factory method for creating Array objects.|


于是乎，我们就可以利用Mybatis的``TypeHandler``来解决这个问题了。具体代码如下

```
/**
 * 自定义Mybatis ARRAY类型处理器
 *
 * @author 7le
 */
@MappedTypes(Object.class)
@MappedJdbcTypes(JdbcType.ARRAY)
public class ArrayTypeHandler extends BaseTypeHandler<Object[]> {

    private static final String TYPE_NAME_VARCHAR = "varchar";
    private static final String TYPE_NAME_INTEGER = "integer";
    private static final String TYPE_NAME_BOOLEAN = "boolean";
    private static final String TYPE_NAME_NUMERIC = "numeric";
    private static final String TYPE_NAME_BIGINT = "bigint";

    @Override
    public void setNonNullParameter(PreparedStatement ps, int i, Object[] parameter, JdbcType jdbcType) throws SQLException {
        String typeName = null;
        if (parameter instanceof Integer[]) {
            typeName = TYPE_NAME_INTEGER;
        } else if (parameter instanceof String[]) {
            typeName = TYPE_NAME_VARCHAR;
        } else if (parameter instanceof Boolean[]) {
            typeName = TYPE_NAME_BOOLEAN;
        } else if (parameter instanceof Double[]) {
            typeName = TYPE_NAME_NUMERIC;
        } else if (parameter instanceof Long[]) {
            typeName = TYPE_NAME_BIGINT;
        } else {
            typeName = TYPE_NAME_VARCHAR;
        }

        Connection conn = ps.getConnection();
        Array array = conn.createArrayOf(typeName, parameter);
        ps.setArray(i, array);

    }

    @Override
    public Object[] getNullableResult(ResultSet resultSet, String s) throws SQLException {
        return getArray(resultSet.getArray(s));
    }

    @Override
    public Object[] getNullableResult(ResultSet resultSet, int i) throws SQLException {
        return getArray(resultSet.getArray(i));
    }

    @Override
    public Object[] getNullableResult(CallableStatement callableStatement, int i) throws SQLException {
        return getArray(callableStatement.getArray(i));
    }

    private Object[] getArray(Array array) {
        if (array == null) {
            return null;
        }
        try {
            return (Object[]) array.getArray();
        } catch (Exception e) {
        }
        return null;
    }
}
```

然后在``application.yml``中加上
```
mybatis:
  type-handlers-package: xxx.xxx.handler   #自定义handler的路径
```

或者
```
@Bean(name = "sessionFactory")
public SqlSessionFactory sessionFactory(@Qualifier("dataSource") DataSource dataSource) throws Exception {
    SqlSessionFactoryBean bean = new SqlSessionFactoryBean();
    bean.setDataSource(dataSource);
    bean.setMapperLocations(new PathMatchingResourcePatternResolver().getResources(PgsqlSourceConfig.MAPPER_LOCATION));
    bean.getObject().getConfiguration().getTypeHandlerRegistry().register(ArrayTypeHandler.class);
    return bean.getObject();
}
```

如果不是springboot的话，用spring的配置方式就可以了。

[Github](https://github.com/7le) 不要吝啬你的star ^.^
[更多精彩 戳我](https://7le.top)
