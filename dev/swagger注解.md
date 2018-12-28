##### 常用到的注解有：

- Api
- ApiModel
- ApiModelProperty
- ApiOperation
- ApiParam
- ApiResponse
- ApiResponses
- ResponseHeader

###### 1. api标记

Api 用在类上，说明该类的作用。可以标记一个Controller类做为swagger 文档资源，使用方式：

```
@Api(value = "/user", description = "Operations about user")
```

与Controller注解并列使用。 属性配置：

| 属性名称       | 备注                                             |
| -------------- | ------------------------------------------------ |
| value          | url的路径值                                      |
| tags           | 如果设置这个值、value的值会被覆盖                |
| description    | 对api资源的描述                                  |
| basePath       | 基本路径可以不配置                               |
| position       | 如果配置多个Api 想改变显示的顺序位置             |
| produces       | For example, "application/json, application/xml" |
| consumes       | For example, "application/json, application/xml" |
| protocols      | Possible values: http, https, ws, wss.           |
| authorizations | 高级特性认证时配置                               |
| hidden         | 配置为true 将在文档中隐藏                        |

在SpringMvc中的配置如下：

```
@Controller
@RequestMapping(value = "/api/pet", produces = {APPLICATION_JSON_VALUE, APPLICATION_XML_VALUE})
@Api(value = "/pet", description = "Operations about pets")
public class PetController {

}
```

###### 2. ApiOperation标记

ApiOperation：用在方法上，说明方法的作用，每一个url资源的定义,使用方式：

```
@ApiOperation(
          value = "Find purchase order by ID",
          notes = "For valid response try integer IDs with value <= 5 or > 10. Other values will generated exceptions",
          response = Order,
          tags = {"Pet Store"})
```

与Controller中的方法并列使用。
 属性配置：

| 属性名称          | 备注                                                         |
| ----------------- | ------------------------------------------------------------ |
| value             | url的路径值                                                  |
| tags              | 如果设置这个值、value的值会被覆盖                            |
| description       | 对api资源的描述                                              |
| basePath          | 基本路径可以不配置                                           |
| position          | 如果配置多个Api 想改变显示的顺序位置                         |
| produces          | For example, "application/json, application/xml"             |
| consumes          | For example, "application/json, application/xml"             |
| protocols         | Possible values: http, https, ws, wss.                       |
| authorizations    | 高级特性认证时配置                                           |
| hidden            | 配置为true 将在文档中隐藏                                    |
| response          | 返回的对象                                                   |
| responseContainer | 这些对象是有效的 "List", "Set" or "Map".，其他无效           |
| httpMethod        | "GET", "HEAD", "POST", "PUT", "DELETE", "OPTIONS" and "PATCH" |
| code              | http的状态码 默认 200                                        |
| extensions        | 扩展属性                                                     |

在SpringMvc中的配置如下：

```
@RequestMapping(value = "/order/{orderId}", method = GET)
  @ApiOperation(
      value = "Find purchase order by ID",
      notes = "For valid response try integer IDs with value <= 5 or > 10. Other values will generated exceptions",
      response = Order.class,
      tags = { "Pet Store" })
   public ResponseEntity<Order> getOrderById(@PathVariable("orderId") String orderId)
      throws NotFoundException {
    Order order = storeData.get(Long.valueOf(orderId));
    if (null != order) {
      return ok(order);
    } else {
      throw new NotFoundException(404, "Order not found");
    }
  }
```

###### 3. ApiParam标记

ApiParam请求属性,使用方式：

```
public ResponseEntity<User> createUser(@RequestBody @ApiParam(value = "Created user object", required = true)  User user)
```

与Controller中的方法并列使用。

属性配置：

| 属性名称        | 备注         |
| --------------- | ------------ |
| name            | 属性名称     |
| value           | 属性值       |
| defaultValue    | 默认属性值   |
| allowableValues | 可以不配置   |
| required        | 是否属性必填 |
| access          | 不过多描述   |
| allowMultiple   | 默认为false  |
| hidden          | 隐藏该属性   |
| example         | 举例子       |

在SpringMvc中的配置如下：

```
 public ResponseEntity<Order> getOrderById(
      @ApiParam(value = "ID of pet that needs to be fetched", allowableValues = "range[1,5]", required = true)
      @PathVariable("orderId") String orderId)
```

###### 4. ApiResponse

ApiResponse：响应配置，使用方式：

```
@ApiResponse(code = 400, message = "Invalid user supplied")
```

与Controller中的方法并列使用。 属性配置：

| 属性名称          | 备注                             |
| ----------------- | -------------------------------- |
| code              | http的状态码                     |
| message           | 描述                             |
| response          | 默认响应类 Void                  |
| reference         | 参考ApiOperation中配置           |
| responseHeaders   | 参考 ResponseHeader 属性配置说明 |
| responseContainer | 参考ApiOperation中配置           |

在SpringMvc中的配置如下：

```
 @RequestMapping(value = "/order", method = POST)
  @ApiOperation(value = "Place an order for a pet", response = Order.class)
  @ApiResponses({ @ApiResponse(code = 400, message = "Invalid Order") })
  public ResponseEntity<String> placeOrder(
      @ApiParam(value = "order placed for purchasing the pet", required = true) Order order) {
    storeData.add(order);
    return ok("");
  }
```

###### 5. ApiResponses

ApiResponses：响应集配置，使用方式：

```
 @ApiResponses({ @ApiResponse(code = 400, message = "Invalid Order") })
```

与Controller中的方法并列使用。 属性配置：

| 属性名称 | 备注                |
| -------- | ------------------- |
| value    | 多个ApiResponse配置 |

在SpringMvc中的配置如下：

```
 @RequestMapping(value = "/order", method = POST)
  @ApiOperation(value = "Place an order for a pet", response = Order.class)
  @ApiResponses({ @ApiResponse(code = 400, message = "Invalid Order") })
  public ResponseEntity<String> placeOrder(
      @ApiParam(value = "order placed for purchasing the pet", required = true) Order order) {
    storeData.add(order);
    return ok("");
  }
```

###### 6. ResponseHeader

> 响应头设置，使用方法

```
@ResponseHeader(name="head1",description="response head conf")
```

与Controller中的方法并列使用。 属性配置：

| 属性名称          | 备注                   |
| ----------------- | ---------------------- |
| name              | 响应头名称             |
| description       | 头描述                 |
| response          | 默认响应类 Void        |
| responseContainer | 参考ApiOperation中配置 |

在SpringMvc中的配置如下：

```
@ApiModel(description = "群组")
```

###### 7. 其他

- @ApiImplicitParams：用在方法上包含一组参数说明；
- @ApiImplicitParam：用在@ApiImplicitParams注解中，指定一个请求参数的各个方面 
  - paramType：参数放在哪个地方
  - name：参数代表的含义
  - value：参数名称
  - dataType： 参数类型，有String/int，无用
  - required ： 是否必要
  - defaultValue：参数的默认值
- @ApiResponses：用于表示一组响应；
- @ApiResponse：用在@ApiResponses中，一般用于表达一个错误的响应信息； 
  - code： 响应码(int型)，可自定义
  - message：状态码对应的响应信息
- @ApiModel：描述一个Model的信息（这种一般用在post创建的时候，使用@RequestBody这样的场景，请求参数无法使用@ApiImplicitParam注解进行描述的时候；
- @ApiModelProperty：描述一个model的属性。



