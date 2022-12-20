**简体中文 | [English](./README.md)**

<p align="center">
    <a href="https://github.com/lyzsk/food-delivery-app/blob/master/LICENSE">
        <img src="https://img.shields.io/github/license/lyzsk/food-delivery-app.svg?style=plastic&logo=github" />
    </a>
    <a href="https://github.com/lyzsk/food-delivery-app/members">
        <img src="https://img.shields.io/github/forks/lyzsk/food-delivery-app.svg?style=plastic&logo=github" />
    </a>
    <a href="https://github.com/lyzsk/food-delivery-app/stargazers">
        <img src="https://img.shields.io/github/stars/lyzsk/food-delivery-app.svg?style=plastic&logo=github" />
    </a>
</p>

# food delivery app

> **_喜欢，或者对你有帮助的话，记得点赞哦_** :star:

# fix bugs

后端 console 里输出的 sql 是:

```
Creating a new SqlSession
SqlSession [org.apache.ibatis.session.defaults.DefaultSqlSession@24a0ba3c] was not registered for synchronization because synchronization is not active
JDBC Connection [com.mysql.cj.jdbc.ConnectionImpl@6e15ef51] will not be managed by Spring
==>  Preparing: UPDATE employee SET status=?, update_time=?, update_user=? WHERE id=?
==> Parameters: 0(Integer), 2022-12-20T21:29:17.878(LocalDateTime), 1(Long), 1605314025255661600(Long)
<==    Updates: 0
```

而 db 里存的 id 是: `1605314025255661569`

然而查前端的 network:

1. `employee` (update 方法 putmapping 是 "/employee") 的 payload 是: `{id: 1605314025255661600, status: 0}`

2. `page?page=1&pageSize=2` (click 更新后 getmapping 是 "/page") 的 response 里是 `{"code":1,"msg":null,"data":{"records":[{"id":1605314025255661569,`

简而言之, 就是 sql 语句执行的 id 和 db 里存的 id 不一致

错误原因:

js 的问题, 损失了精度, 雪花算法生成的 id 是 19 位的, response 里是正确的, 而 js 处理 response 的时候损失了精度, 因为 js 只能保证前 16 位是精确的, 后面的进行了四舍五入 569 -> 600

解决方法:

在给服务端响应 json 数据时把 Long 类型型统一转化为 String 类型

1. 提供对象转换器 `JacksonObjectMapper`, 基于 Jackson 进行 Java 对象到 json 数据的转换

```java
public class JacksonObjectMapper extends ObjectMapper {
    public static final String DEFAULT_DATE_FORMAT = "yyyy-MM-dd";
    public static final String DEFAULT_DATE_TIME_FORMAT = "yyyy-MM-dd HH:mm:ss";
    public static final String DEFAULT_TIME_FORMAT = "HH:mm:ss";

    public JacksonObjectMapper() {
        super();
        //收到未知属性时不报异常
        this.configure(FAIL_ON_UNKNOWN_PROPERTIES, false);

        //反序列化时，属性不存在的兼容处理
        this.getDeserializationConfig()
            .withoutFeatures(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES);

        SimpleModule simpleModule =
            new SimpleModule().addDeserializer(LocalDateTime.class,
                    new LocalDateTimeDeserializer(
                        DateTimeFormatter.ofPattern(DEFAULT_DATE_TIME_FORMAT)))
                .addDeserializer(LocalDate.class, new LocalDateDeserializer(
                    DateTimeFormatter.ofPattern(DEFAULT_DATE_FORMAT)))
                .addDeserializer(LocalTime.class, new LocalTimeDeserializer(
                    DateTimeFormatter.ofPattern(DEFAULT_TIME_FORMAT)))

                .addSerializer(BigInteger.class, ToStringSerializer.instance)
                .addSerializer(Long.class, ToStringSerializer.instance)
                .addSerializer(LocalDateTime.class, new LocalDateTimeSerializer(
                    DateTimeFormatter.ofPattern(DEFAULT_DATE_TIME_FORMAT)))
                .addSerializer(LocalDate.class, new LocalDateSerializer(
                    DateTimeFormatter.ofPattern(DEFAULT_DATE_FORMAT)))
                .addSerializer(LocalTime.class, new LocalTimeSerializer(
                    DateTimeFormatter.ofPattern(DEFAULT_TIME_FORMAT)));

        //注册功能模块 例如，可以添加自定义序列化器和反序列化器
        this.registerModule(simpleModule);
    }
}
```

重点在 `.addSerializer(Long.class, ToStringSerializer.instance)`, 也就是遇到 Long 类型 使用 `ToStringSerializer`, 同理对于 LocalDateTime, LocalDate, LocalTime 的转换, 因为前端 json 里日期是: `[date_time]` 的形式, 使用起来不如 String 方便

2. 在 `WebMvcConfig` 中扩展 SpringMVC 的消息转换器, 在消息转换器中使用 `JacksonObjectMapper`

```java
    /**
     * 扩展MVC框架的消息转换器
     *
     * @param converters
     */
    @Override
    protected void extendMessageConverters(
        List<HttpMessageConverter<?>> converters) {
        log.info("扩展消息转换器...");
        // 创建消息转换器对象
        MappingJackson2HttpMessageConverter messageConverter =
            new MappingJackson2HttpMessageConverter();
        // 引入JacksonObjectMapper
        messageConverter.setObjectMapper(new JacksonObjectMapper());
        // 把配置好的消息转换器引入MVC框架中, 记得加index头插
        converters.add(0, messageConverter);
    }
```

更改后测试:

前端分页查询 `page?page=1&pageSize=2` 的 Response:

```js
{"code":1,"msg":null,"data":{"records":[{"id":"1605314025255661569","name":"赵六","username":"zhaoliu","password":"e10adc3949ba59abbe56e057f20f883e","phone":"18343219876","sex":"0","idNumber":"987612345123456789","status":1,"createTime":"2022-12-20 21:27:44","updateTime":"2022-12-20 21:27:44","createUser":"1","updateUser":"1"},{"id":"1605313868686487553","name":"王五","username":"wangwu","password":"e10adc3949ba59abbe56e057f20f883e","phone":"18398765432","sex":"1","idNumber":"123456789123456789","status":1,"createTime":"2022-12-20 21:27:06","updateTime":"2022-12-20 21:27:06","createUser":"1","updateUser":"1"}],"total":5,"size":2,"current":1,"orders":[],"optimizeCountSql":true,"searchCount":true,"countId":null,"maxLimit":null,"pages":3},"map":{}}
```

可以看到 id 是 String 了, 日期相关的也是自定义的 String 格式了

然后再更新操作:

```
JDBC Connection [com.mysql.cj.jdbc.ConnectionImpl@2228f4e6] will not be managed by Spring
==>  Preparing: UPDATE employee SET status=?, update_time=?, update_user=? WHERE id=?
==> Parameters: 0(Integer), 2022-12-20T22:16:35.783(LocalDateTime), 1(Long), 1605314025255661569(Long)
<==    Updates: 1
```

后端更新成功, 前端 Payload 也返回正确数值: `{id: "1605314025255661569", status: 0}`

# LICENSE

Copyright (c) 2022 lyzsk