---
title:  Code Generator
date:   2020-03-26 21:55:55 +0800
categories:
  - utils
tags:
  - generator
---

本文主要介绍 [Code Generator](https://github.com/vazmin/code-generator) 的目的、背景及实现细节。

`Code Generator` 能根据对象表结构和模板生成代码文件。


### 目的

* 减少书写繁琐重复的代码
* 灵活生成不同风格的项目代码

### 背景

开发一个新表增删改查的功能，如果需要一行一行的敲或复制原有的代码变更类名、字段名等等，那简直是个漫长的噩梦。

[mybatis/generator](https://github.com/mybatis/generator)提供了很不错的功能，但那远远不够。我们还需要生成Service,Controller,前端的代码,i18n文件等等，而且还要输出不同代码风格的代码。一般的项目之间的代码风格多多少少都不太一样，不同公司的代码风格就更大了。又或者我们写的不只是java项目。当然`mybatis/generator`也提供了插件，用起来还是挺笨重。

所以我觉得<b>配置+模板</b>的形式比较理想，比较灵活。


## 应用视角

### 查询表和列架构

从jdbc连接中获取`DatabaseMetaData`, `catalog`, `schema`
```java
DatabaseMetaData databaseMetaData = connection.getMetaData();
final String catalog = connection.getCatalog();
final String schema = connection.getSchema();
```

取列结构
```java
ResultSet rs = databaseMetaData.getColumns(tc.getCatalog(), tc.getSchema(),
                tc.getTableName(), "%"); 
while (rs.next()) {
    // 省略代码
    IntrospectedColumn introspectedColumn = new IntrospectedColumn();
    introspectedColumn.setActualTypeName(rs.getString("TYPE_NAME")); 
    introspectedColumn.setActualColumnName(rs.getString("COLUMN_NAME")); 
    introspectedColumn.setRemarks(rs.getString("REMARKS")); 
    // 省略代码
}
```

取表信息
```java
ResultSet rs = databaseMetaData.getTables(tc.getCatalog(), tc.getSchema(),
                    tc.getTableName(), null);
if (rs.next()) {
    String remarks = rs.getString("REMARKS"); 
    String tableType = rs.getString("TABLE_TYPE"); 
    introspectedTable.setRemarks(remarks);
    introspectedTable.setTableType(tableType);

}
```

#### 渲染Freemark模板

[Freemarker](https://freemarker.apache.org/)语法相对简单，看一下就能写。这里随便举两个


输出model

```java
package ${TableNamePackage};

import java.io.Serializable;

/**
* ${table.clearRemark} Bean
*
* Created by ${app.author} on ${genDate}.
*/
public class ${table.upperCamelCaseName} implements Serializable {

<#list columnList as column>
    /** ${column.remarks} */
    private ${column.javaType} ${column.lowerCamelCaseName};
</#list>

<#list columnList as column>
    public ${column.javaType} get${column.upperCamelCaseName}() {
        return ${column.lowerCamelCaseName};
    }

    public void set${column.upperCamelCaseName}(${column.javaType} ${column.lowerCamelCaseName}) {
        this.${column.lowerCamelCaseName} = ${column.lowerCamelCaseName};
    }

</#list>
}
```

输出前端需要的Angular的路由文件

```typescript
import {Routes} from '@angular/router';
import {${table.upperCamelCaseName}Component} from './${table.lowerCaseSubName}.component';
import {${table.upperCamelCaseName}DetailComponent} from './${table.lowerCaseSubName}-detail.component';
import {${table.upperCamelCaseName}EditComponent} from './${table.lowerCaseSubName}-edit.component';
import {InletComponent} from '../../../@theme/components/inlet.component';

export const ${table.lowerCamelCaseName}Routes: Routes = [
    {
        path: '${table.lowerCaseSubName}',
        component: InletComponent,
        children: [
            {
                path: '',
                component: ${table.upperCamelCaseName}Component,
            }, {
                path: 'detail',
                component: ${table.upperCamelCaseName}DetailComponent
            }, {
                path: 'new',
                component: ${table.upperCamelCaseName}EditComponent
            }, {
                path: 'edit',
                component: ${table.upperCamelCaseName}EditComponent
            }
        ]
    },

];

export const routed${table.upperCamelCaseName}Components = [
    ${table.upperCamelCaseName}Component,
    ${table.upperCamelCaseName}DetailComponent,
    ${table.upperCamelCaseName}EditComponent,
];

```


### 约束

* 数据源需支持`JDBC`连接
* 表和表字段应该带有注释


待续......