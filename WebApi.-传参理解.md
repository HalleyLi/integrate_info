

####  webApi-传参理解

@(开发规则)


每一个URL代表一个资源，听过表现层约定资源的展现形式，通过HTTP METHOD 指定对资源的何种操作（状态转换）

在URI中包含动词。比如：

更新资源，POST, 
创建资源，PUT, 
获取资源，GET, 
删除资源，DELETE,

controller层里的方法传参，对直接简单类型参数，以及参数前加FromURL、FromBody的例子如下。
1. 直接从url上获取参数，对于简单类型；

```
 [HttpGet]
    [Route("companyAccount/{companyAccountId}")]
    [ActionName("queryCompanyAccountById")]
    public WapResponse<BMCompanyAccountDto> GetCompanyAccountById(int companyAccountId)
    { }
```

2. 对于一个复杂类型的参数，webApi使用media-type formatter 去从request body 读取值

```
 [HttpPut]
        [Route("companyAccount/{companyAccountId}")]
        [ActionName("modifyCompanyAccount")]
        public WapBoolean ModifyCompanyAccount(int companyAccountId, [FromBody]BMCompanyAccountDto companyaccount)
        {}
```

3. 强制wepApi去从url读取一个复杂类型的话，可以在参数前加FromUrl属性。下面的例子定义一个PageCondition类型，along with a controller方法，然后从url里获取PageCondition.

```
 public class PageCondition
    {
   
        public PageCondition(int start, int length)
        {
            this.Start = start;
            this.Length = length;
        }

       
        public PageCondition(int start, int length, string[] sortField, string[] sortDirection)
        {
            this.Start = start;
            this.Length = length;
            this.SortField = sortField;
            this.SortDirection = sortDirection;
        }
        public int Start { get; set; }

        public int Length { get; set; }

        //public string SearchText { get; set; }

        public string[] SortField { get; set; }

        public string[] SortDirection { get; set; }
   }
```

```
public ValuesController : ApiController
{
    public HttpResponseMessage Get([FromUri] PageCondition condition) { ... }
}
```

4. 强制webApi从request body读取参数，则在参数前添加FromBody属性

```
public HttpResponseMessage Post([FromBody] string name) { ... }
```

在这个例子里，webApi使用一个 media-type formatter 从request body中读取对应name的value.下面是一个客户端请求的例子。

```
POST http://localhost:5076/api/values HTTP/1.1
User-Agent: Fiddler
Host: localhost:5076
Content-Type: application/json
Content-Length: 7

"Alice"
```

当一个参数前有FromBody属性，webApi则使用Content-Type header去选择一个formatter.在这个例子里，content type 是 ‘application/json’,请求体是一个原始的JSON字符串。

大多数情况下，我们只能从消息体力读取一个参数，所以下面这段代码是无法起作用的；

```
// Caution: Will not work!    
public HttpResponseMessage Post([FromBody] int id, [FromBody] string name) { ... }
```

原因是：有一个规定是，请求体被存储在一个只能被读取一次的non-buffered 流

