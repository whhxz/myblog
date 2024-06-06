---
title: 一次mybatis映射未获取到值
date: 2024-06-06 15:29:30
categories:
tags: ['开发异常']
---

一次项目依赖版本变动，导致mybatis返回的字段变为空值。
查询sql如下:
```xml
<select id="findFreshActivity" parameterType="int" resultMap="freshActivity">
    select id,
    activity_name activityName
    from fresh_activity
    <where>
        id = #{0}
    </where>
</select>
```
映射如下：
```xml
<resultMap id="freshActivity" type="com.feiniu.common.po.FreshActivity">
    <id column="id" property="id" jdbcType="INTEGER"/>
    <result column="activity_name" property="activityName" jdbcType="VARCHAR"/>
</resultMap>
```
发现返回的对象里面**id**有值，**activityName**为null。
<!-- more -->
旧版本使用**3.4.0**
```xml
<dependency>
    <groupId>org.mybatis</groupId>
    <artifactId>mybatis</artifactId>
    <version>3.4.0</version>
</dependency>
```
新版本使用**3.5.6**。
按照正常情况说，上面sql写了别名，但是映射里面又不是用的别名，这样写不规范。但是在使用旧版本时无异常，数据都正常返回，这次改动版本后，却没有返回数据。

原理如下：
```java
//版本3.5.6
//org.apache.ibatis.executor.resultset.DefaultResultSetHandler#createAutomaticMappings
private List<UnMappedColumnAutoMapping> createAutomaticMappings(ResultSetWrapper rsw, ResultMap resultMap, MetaObjectmetaObject, String columnPrefix) throws SQLException {
  final String mapKey = resultMap.getId() + ":" + columnPrefix;
  List<UnMappedColumnAutoMapping> autoMapping = autoMappingsCache.get(mapKey);
  if (autoMapping == null) {
    autoMapping = new ArrayList<>();
    final List<String> unmappedColumnNames = rsw.getUnmappedColumnNames(resultMap, columnPrefix);
    for (String columnName : unmappedColumnNames) {
      String propertyName = columnName;
      if (columnPrefix != null && !columnPrefix.isEmpty()) {
        // When columnPrefix is specified,
        // ignore columns without the prefix.
        if (columnName.toUpperCase(Locale.ENGLISH).startsWith(columnPrefix)) {
          propertyName = columnName.substring(columnPrefix.length());
        } else {
          continue;
        }
      }
      final String property = metaObject.findProperty(propertyName, configuration.isMapUnderscoreToCamelCase());
      if (property != null && metaObject.hasSetter(property)) {
        //如果配置了映射，不处理，后续由applyPropertyMappings处理
        if (resultMap.getMappedProperties().contains(property)) {
          continue;
        }
        final Class<?> propertyType = metaObject.getSetterType(property);
        if (typeHandlerRegistry.hasTypeHandler(propertyType, rsw.getJdbcType(columnName))) {
          final TypeHandler<?> typeHandler = rsw.getTypeHandler(propertyType, columnName);
          autoMapping.add(new UnMappedColumnAutoMapping(columnName, property, typeHandler, propertyType.isPrimitive()));
        } else {
          configuration.getAutoMappingUnknownColumnBehavior()
              .doAction(mappedStatement, columnName, property, propertyType);
        }
      } else {
        configuration.getAutoMappingUnknownColumnBehavior()
            .doAction(mappedStatement, columnName, (property != null) ? property : propertyName, null);
      }
    }
    autoMappingsCache.put(mapKey, autoMapping);
  }
  return autoMapping;
}
```
```java
//版本3.4.0
private List<UnMappedColumnAutoMapping> createAutomaticMappings(ResultSetWrapper rsw, ResultMap resultMap, MetaObjectmetaObject, String columnPrefix) throws SQLException {
  final String mapKey = resultMap.getId() + ":" + columnPrefix;
  List<UnMappedColumnAutoMapping> autoMapping = autoMappingsCache.get(mapKey);
  if (autoMapping == null) {
    autoMapping = new ArrayList<UnMappedColumnAutoMapping>();
    final List<String> unmappedColumnNames = rsw.getUnmappedColumnNames(resultMap, columnPrefix);
    for (String columnName : unmappedColumnNames) {
      String propertyName = columnName;
      if (columnPrefix != null && !columnPrefix.isEmpty()) {
        // When columnPrefix is specified,
        // ignore columns without the prefix.
        if (columnName.toUpperCase(Locale.ENGLISH).startsWith(columnPrefix)) {
          propertyName = columnName.substring(columnPrefix.length());
        } else {
          continue;
        }
      }
      final String property = metaObject.findProperty(propertyName, configuration.isMapUnderscoreToCamelCase());
      //此处没有上述代码，如果匹配到了类里面属性直接赋值
      if (property != null && metaObject.hasSetter(property)) {
        final Class<?> propertyType = metaObject.getSetterType(property);
        if (typeHandlerRegistry.hasTypeHandler(propertyType, rsw.getJdbcType(columnName))) {
          final TypeHandler<?> typeHandler = rsw.getTypeHandler(propertyType, columnName);
          autoMapping.add(new UnMappedColumnAutoMapping(columnName, property, typeHandler, propertyType.isPrimitive()));
        } else {
          configuration.getAutoMappingUnknownColumnBehavior()
                  .doAction(mappedStatement, columnName, property, propertyType);
        }
      } else{
        configuration.getAutoMappingUnknownColumnBehavior()
                .doAction(mappedStatement, columnName, (property != null) ? property : propertyName, null);
      }
    }
    autoMappingsCache.put(mapKey, autoMapping);
  }
  return autoMapping;
}
```
根本原理就是新版本里面加了一个判断，如果设置了映射就会走映射。