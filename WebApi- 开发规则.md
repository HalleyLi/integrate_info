

###  WebApi-develop-usage-rules

@(开发规则)


## 概述

WebApi的定义必须符合RESTful规则。

RESTful是Representational State Transfer的缩写。理解它就要先理解：资源（Resource）、表现层、状态转换三个概念。

* 资源就是一个实体、一个具体信息，可以是文本、图片、服务等等，可以用一个URI指向它、每个资源都有一个特定的URI，唯一的，访问这个资源只需要知道URI是什么就可以访问了。
* 表现层，就是指资源可以有多种表现形式，可以是JSON格式、XML格式、二进制、HTML等等。通过HTTP头的Accept和Content-Type字段指定。
* 状态转换，通过HTTPS协议的METHOD指定针对某一个资源的什么操作。GET：获取，POST：创建，PUT/PATCH：更新，DELETE：删除。

总结出来就是：每一个URI代表一个资源，通过表现层约定资源的展现形式，通过HTTP METHOD指定对资源的何种操作（状态转换）。

## 正确用法：

*   URI仅标识资源的路径（位置）
*   以复数（名词）进行资源命名。路径表示资源唯一资源时采用单数，比如通过ID获取资源。
*   明确的使用小写字母、数字以及下划线(_)来命名资源和属性。下划线主要是用来区分多个单词组成的资源或属性。严禁使用大写字母描述字段。
*   资源的路径从父到子依次定义。如：`/resource/{resource_id}/sub_resource/{sub_resource_id}/sub_resource_property`。
*   使用内容协商（Content-Type）来区分资源的表示格式。比如：JSON、XML等。此限定适用于请求和返回两种操作。必须指定Accept。
*   通过HTTP动词（HTTP METHOD）表示对资源的具体操作。

    * GET：从服务器取出资源（一项或多项）。
    * POST：在服务器新建一个资源。
    * PUT：在服务器更新资源（客户端提供改变后的完整资源，全部更新）。
    * PATCH：在服务器更新资源（客户端提供改变的属性，部分更新）。
    * DELETE：从服务器删除资源。
    * HEAD：获取资源的元数据。
    * OPTIONS：获取信息，关于资源的哪些属性是客户端可以改变的。

    **示例：**

    * 创建一个总行，POST, `/subbranchs`
    * 更新更新总行编号是1的记录，PUT, `/subbranch/1`
    * 更新总行信息的状态，PATCH，`/subbranch/1/state`。
    * 删除总行编号是1的记录，DELETE, `/subbranch/1` 
    * 查询、搜索总行，GET, `/subbranchs?name=上海银行&address=控江路`

    * 获取最新的账务年月，GET, `/billing_month/lasted`
    * 获取最新的账务年月，标识。GET, `/billing_month/lasted/key`。

    * 获取所有变更的抄表任务ID列表，GET, `/pm_reading/tasks/updated_ids?meter_reader=123`

*   编码尽可能采用UTF-8
*   日期格式采用UTC Long的形式表示。
*   使用Query String进行资源的过滤。比如分页、搜索等。
*   仅仅使用Content Body来传输数据，比如创建资源时，资源的众多属性内容应该通过Content Body传入。
*   文档尽可能描述清楚返回数据每个字段的默认值。便于统一判断字段是否存在。
*   POST操作（创建资源）应尽可能的返回新建的资源。PUT/PATCH（更新）操作返回更新后的完整资源。
*   返回多个资源时尽量使用属性名包裹。比如：

    ```json
      {
          code: 0,
          data: {
              count: 100,
              items: [
                  { ... },
                  { ... }
              ]
          }
      }
    ```

*   返回状态码为非2XX时，返回的Content Body必须携带详细错误。如下：

      返回的状态码必须明确，除此之外ContentBody中明确携带错误信息。

    ```json
      {
          code: 101231,      // 针对返回状态码的补充。具体业务错误号。
          message: '',          // 错误描述
          data: {
              server_time: 12312031232103,  // 服务器时间
              host_id: 1201,                       // 服务实例ID
              error: 'ResouceNotFound',   // 错误类型 
          }
      }
    ```

### 状态码定义

应该为每个请求返回适当的状态码，常用状态码：

* 200 ok  - 成功返回状态，对应，GET,PUT,PATCH,DELETE。
* 201 created  - 成功创建。
* 304 not modified   - HTTP缓存有效。
* 400 bad request   - 请求格式错误。
* 401 unauthorized   - 未授权。
* 403 forbidden   - 鉴权成功，但是该用户没有权限。
* 404 not found - 请求的资源不存在
* 405 method not allowed - 该http方法不被允许。
* 410 gone - 这个url对应的资源现在不可用。
* 415 unsupported media type - 请求类型错误。
* 422 unprocessable entity - 校验错误时用。
* 429 too many request - 请求过多。
* 500 internal server error - 通用错误响应。
* 503 service unavailable - 服务当前无法处理请求。


## 常见错误

在URI中包含动词。比如：

* 更新资源，POST, `headoffice/1/update`。错误的使用了Http动词，URI中不应该出现动词，应该使用PUT，去除update。
* 创建资源，PUT, `headoffice/create`。 错误的使用了HTTP动词，资源没有使用复数。
* 获取资源，GET, `headoffice/get/{type}`。URI中出现动词（get），资源没有使用复数。
* 获取资源，GET, `headoffice/get/{id}`。URI中出现了动词（get）。
* 获取资源，GET,relationObject/{customerId}, relationObject/{objectId}/{type},会有冲突

