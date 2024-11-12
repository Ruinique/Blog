---
title: Jackson 注解浅用
cover: /date_kim.jpg
date: 2024-02-01 10:00:00
categories: 技术
author: ruinique
---

# 背景
在实际的开发场景中，我们往往要对 `json` 数据进行序列化，但是仅仅依据字段名去序列化，使用 `ObjectMapper` 的 `readValue()` 去解析数据，对于内部有其他数据结构，如 `Map` 这种很多层级的数据而言，就显得力不从心。因此，我们有时需要使用自定义程度更高的序列化，比如 `@JsonAnySetter` 注解，才可以满足我们的需求。

# @JsonSetter 和 @JsonGetter 注解简单介绍

对于具体的字段上，我们有的时候要把对应的下划线的命名方法的字段序列化成驼峰命名的，这个时候我们就要用到 `@JsonProperty` 注解来帮助我们对对应的数据进行序列化了。具体的例子如下：
```java
import lombok.AllArgsConstructor;  
import lombok.Data;  
import lombok.NoArgsConstructor;  
  
@Data  
@NoArgsConstructor  
@AllArgsConstructor  
public class SimpleObject {  
    private Long objectId;  
    private String data;  
}
```
json 样例如下：
```json
{
	"object_id":123,
	"data":"123"
}
```
像上面这样一个对象是遵循驼峰命名法的，如果我们直接用对应的工具进行序列化，会导致序列化不成功。
```java
public class TestJackson {  
    public static void main(String[] args) throws IOException {  
        testJsonProperty();  
    }  
  
    public static void testJsonProperty() throws IOException {  
        String data = "{\n"  
                + "\t\"object_id\":123,\n"  
                + "\t\"data\":\"123\"\n"  
                + "}";  
        SimpleObject object = new ObjectMapper().readValue(data, SimpleObject.class);  
        System.out.println(object);  
    }  
}
```
报错如下
```bash
Exception in thread "main" com.fasterxml.jackson.databind.exc.UnrecognizedPropertyException: Unrecognized field "object_id" (class testJackson.model.SimpleObject), not marked as ignorable (2 known properties: "objectId", "data"])
 at [Source: (String)"{
	"object_id":123,
	"data":"123"
}"
```
那么我们应该使用对应的注解去解决这样的问题。
```java
@Data  
@NoArgsConstructor  
@AllArgsConstructor  
public class SimpleObject {  
    @JsonSetter("object_id")  
    private Long objectId;  
    @JsonSetter("data")  
    private String data;  
}
```
改成这个样子，就可以成功序列化了，可以解决命名出现的问题。`@JsonGetter` 应用的场景则恰好和 `@JsonSetter` 相反。

# @JsonNaming 注解简单介绍

对于上面这种情况，我们其实还有一种更加专门解决小写驼峰和下划线命名之间的办法，使用 `@JsonNaming` 注解去完成对应工作。
`@JsonNaming` 是一个类注解，我们使用 `@JsonNaming` 注解可以更加简单地将整个类的字段都实现小写驼峰和下划线之间的转换，具体效果如下：
```java
@Data  
@NoArgsConstructor  
@AllArgsConstructor  
@JsonNaming(PropertyNamingStrategy.SnakeCaseStrategy.class)  
public class SimpleObject {  
    private Long objectId;  
    private String data;  
}
```
就可以解析这个问题了，除了这里提到的 `SnakeCaseStrategy` 以外，我们还有其他策略：
1. **SnakeCaseStrategy:** 转换属性名为 `snake_case` 形式。例如，`propertyName` 会变成 `property_name`。
2. **LowerCamelCaseStrategy:** 保持属性名为 Java 风格的 `camelCase`，不做转换。例如，`propertyName` 保持不变。
3. **UpperCamelCaseStrategy:** 转换属性名为 `PascalCase`（类似于 `camelCase`，但首字母大写）。例如，`propertyName` 会变成 `PropertyName`。
4. **KebabCaseStrategy:** 转换属性名为 `kebab-case`（单词小写，用连字符连接）。例如，`propertyName` 会变成 `property-name`。
5. **LowerDotCaseStrategy:** 转换属性名为 `lower.dot.case` 形式，其中单词用点分隔，所有字母小写。例如，`propertyName` 会变成 `property.name`。
6. **LowerCaseStrategy:** 转换所有属性名为全小写，不包含任何分隔符。例如，`propertyName` 会变成 `propertyname`。
7. **UpperSnakeCaseStrategy:** 转换属性名为全大写的 `SNAKE_CASE`。例如，`propertyName` 会变成 `PROPERTY_NAME`。
## @JsonProperty 注解简单介绍
`@JsonProperty` 注解则是十分通用的注解，可以用于字段、构造器参数、setter 和 getter 方法。它允许指定 JSON 属性的名称，并且用于控制序列化和反序列化过程。
- 当用于字段时，它指定了 JSON 属性和 Java 字段之间的映射。
- 当用于构造器参数时，它帮助在反序列化时映射 JSON 属性到构造器参数。
- 当用于 setter 和 getter 方法时，它改变了属性在 JSON 中的名称。
我们可以把他当成综合了 `@JsonSetter` 和 `@JsonGetter` 注解来使用。

## @JsonAnySetter 和 @JsonAnyGetter 注解简单介绍

有的时候，我们需要设置未反序列化的属性名和值作为键值存储到 map 中，这时候我们就需要用到这个注解了。
像这样一个对象
```java
@Data  
@AllArgsConstructor  
@NoArgsConstructor  
public class MapObject {  
    private Map<String, Object> map = new HashMap<>();  
    @JsonAnySetter  
    public void add(String key, Object value) {  
        map.put(key, value);  
    }  
}
```
他就可以把之前的 `json` 数据解析成对应的具体内容：
```bash
MapObject(map={data=123, object_id=123})
```
我在业务中遇到过对应的情况：
```java
@Data  
@AllArgsConstructor  
@NoArgsConstructor  
@Builder  
@Slf4j  
public class XxxInfoDto {  
    private Map<String, ChildXxxInfoDto> XxxInfoDtoList;  
}
```
```java
@Data  
@Builder  
@AllArgsConstructor  
@NoArgsConstructor  
public class ChildXxxInfoDto {  
    private String data1;  
    private String data2;  
    private String data3;  
}
```
原始数据如下：
```java
"{"99661341216442":"99,1.3,1.2"}"
```
对于这样的数据，当时对 `@JsonAnySetter` 不熟悉，用了很暴力的方法去逐个解析的，现在看应该可以这么写。
```java
@Data  
@AllArgsConstructor  
@NoArgsConstructor  
@Builder  
@Slf4j  
public class XxxInfoDTO {  
    private Map<String, ChildXxxInfoDTO> XxxInfoDtoList = Maps.newHashMap();  
    @JsonAnySetter  
    public void set(String key, String value) {  
        String[] values = value.split(",");  
        XxxInfoDtoList.put(key, ChildXxxInfoDTO.builder().data1(values[0]).data2(values[1]).data3(values[2]).build());  
    }  
}
```
上面这段代码，可以很简单地完成我们需要的序列化。

# @JsonView 注解的简单介绍

对于 `@JsonView` 注解，往往用于不同场景的策略不同，如下：
```java
@Setter
public class User {
    public interface UserDetail{}
 
    public interface UserInfo extends UserDetail{}
 
    /**
     * 在UserInfo视图上展示userNamw这个字段；
     * 要注意的是UserInfo继承UserDetail，所以展示的时候也会展示UserDetail视图中的字段，也就是password字段
     * @return
     */
    @JsonView(UserInfo.class)
    public String getUserName() {
        return userName;
    }
 
    /**
     * 在UserDetail视图上展示password字段
     * @return
     */
    @JsonView(UserDetail.class)
    public String getPassword() {
        return password;
    }
 
    private String userName;
    private String password;
 
}
```
这样，我们在序列化的时候，如果是 `UserDetail` 的时候，我们就只看到 `UserDetail` 的 `password` 字段，而在 `UserInfo` 就可以看到两者了。