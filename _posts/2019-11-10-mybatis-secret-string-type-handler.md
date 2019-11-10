---
title:  "数据库指定字段加密-Mybatis TypeHandler"
date:   2019-11-10 19:00:00 +0800
categories:
  - java
tags:
  - mybatis
  - TypeHandler
---

## 需求
贩卖用户数据都快成为行业传统了。为了加强保护用户个人信息，需要对数据库中记录个人信息的字段进行加密。
即入库时自动加密，查询数据自动解密。

## 方案

1. 定义TypeHandler, 需要加解密的字段指定TypeHandler。
2. DTO层手动处理。
3. 编写拦截器插件，定义加密注解，监听`ParameterHandler`，`ResultSetHandler` 信号对带加密注解的参数进行加密,带加密注解的字段解密。
（尝试过，存在不能容忍的弊端,例如插入对象后，需要加密字段的值被加密，后续代码要使用还得解密。 不推荐使用）
4. sql语句修改，调用mysql提供的函数加解密

本文选用第一种。

## Code

定义字段加解密接口

``` java
/**
 * 字符加密器
 * 相同参数，返回值应该幂等
 */
public interface StringEncryptor {

  /**
   * Encrypt the string.
   */
  String encrypt(String str);

  /**
   * Decrypt the string.
   */
  String decrypt(String encryptedStr);

}
```

定义TypeHandler

```java
/**
 * 字段加解密处理, 无法解密返回原值
 * 需要在xml中指定TypeHandler
 * 返回值ResultMap:
 * <result column="" jdbcType="VARCHAR" property="" typeHandler="secretStringTypeHandler"/>
 * 参数:
 * <select> xxx where email = #{email, typeHandler=secretStringTypeHandler}</select>
 */
public class SecretStringTypeHandler implements TypeHandler<String> {

    private static final Logger log = LoggerFactory.getLogger(SecretStringTypeHandler.class);

    private StringEncryptor stringEncryptor;

    public SecretStringTypeHandler() {
    }

    public SecretStringTypeHandler(StringEncryptor stringEncryptor) {
        this.stringEncryptor = stringEncryptor;
    }

    public StringEncryptor getStringEncryptor() {
        return stringEncryptor;
    }

    public void setStringEncryptor(StringEncryptor stringEncryptor) {
        this.stringEncryptor = stringEncryptor;
    }

    @Override
    public void setParameter(PreparedStatement ps, int i, String parameter, JdbcType jdbcType) throws SQLException {
        ps.setString(i, getEncryptResult(parameter));
    }

    @Override
    public String getResult(ResultSet rs, String columnName) throws SQLException {
        return getDecryptResult(rs.getString(columnName));
    }

    @Override
    public String getResult(ResultSet rs, int columnIndex) throws SQLException {
        return getDecryptResult(rs.getString(columnIndex));
    }

    @Override
    public String getResult(CallableStatement cs, int columnIndex) throws SQLException {
        return cs.getString(columnIndex);
    }

    /**
     * 加密字段
     * @param result 原始值
     * @return 加密值
     */
    private String getEncryptResult(String result) {
        if (StringUtils.isBlank(result)) {
            return result;
        }
        try {
            result = stringEncryptor.encrypt(result);
        } catch (Exception e) {
            log.error("set param fail: {}, {}", result, e.getMessage());
        }
        return result;
    }

    /**
     * 解密字段
     * @param result 加密值
     * @return 原始值
     */
    private String getDecryptResult(String result) {
        if (StringUtils.isBlank(result)) {
            return result;
        }
        try {
            result = stringEncryptor.decrypt(result);
        } catch (Exception e) {
            log.error("get result fail: {}, {}", result, e.getMessage());
        }
        return result;
    }
}
```

注册TypeHandler
```java
/**
 * Mybatis 自定义配置
 */
@org.springframework.context.annotation.Configuration
public class MybatisConfigurationCustomizer implements ConfigurationCustomizer {

    private static final Logger log = LoggerFactory.getLogger(MybatisConfigurationCustomizer.class);

    @Autowired
    private SecretStringTypeHandler secretStringTypeHandler;

    @Bean
    public StringEncryptor stringEncryptor() {
        // 密码应从配置里面获得
        return new AesEncryptor("password");
    }

    @Bean
    public SecretStringTypeHandler secretStringTypeHandler(StringEncryptor stringEncryptor) {
        return new SecretStringTypeHandler(stringEncryptor);
    }

    @Override
    public void customize(Configuration configuration) {
        log.debug("<-- MybatisConfigurationCustomizer ");
        configuration.getTypeAliasRegistry()
            .registerAlias("secretStringTypeHandler", SecretStringTypeHandler.class);
        configuration.getTypeHandlerRegistry()
            .register(secretStringTypeHandler);
    }
}

```

## 使用
返回值ResultMap:
```xml
<result column="email" jdbcType="VARCHAR" property="email" typeHandler="secretStringTypeHandler"/>
```

参数：
```xml
<select> xxx where email = #{email, typeHandler=secretStringTypeHandler}</select>
```

需要注意：

* 实现`StringEncryptor`的加密器应选用幂等的加密算法。
* 加密后的字段数据长度将变长。数据结构需修改。
* 加密字段使用不了模糊查询