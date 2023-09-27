# SSO.Client 使用文档
该工具集成在 .NetFramework 或者 .netcore 的项目中.
## 过滤器
### .netframework
```
filters.Add(new MyHandleErrorAttribute());  //全局错误处理过滤器
filters.Add(new ValidateModelStateAttribute());  //全局model验证过滤器
filters.Add(new SSOAuthorizeAttribute()); //全局sso验证过滤器
filters.Add(new LogRecordAttribute());  //全局日志记录过滤器(远程)
```
### .netcore
```
services.AddControllers(options =>
{
    options.Filters.Add(new MyHandleErrorAttribute());  //全局错误处理过滤器
    options.Filters.Add(new ValidateModelStateAttribute());   //全局model验证过滤器
    options.Filters.Add(new SSOAuthorizeAttribute());  //全局sso验证过滤器
    options.Filters.Add(new LogRecordAttribute()); //全局日志记录过滤器(远程)
});
```
### 排除过滤器
- `AllowAnonymousAttribute`:  //匿名验证过滤器
- `NoneLogRecordAttribute`:   //不记录日志过滤器
### 限流过滤器
- `ThrottlingAttribute` 
## 配置文件
### .netframework
```
<add key="ssoBaseUrl" value=""/>  <!--项目的基本地址-->
<add key="ssoSecretKey" value=""/>  <!--解密秘钥,要和基本地址的秘钥一致-->
<add key="ssoCookieKey" value="c.web.auth"/>  <!--cookie的键名称-->
<add key="ssoCookieTime" value="session"/>  <!--cookie保存的时间(分钟)-->
<add key="messageBaseUrl" value=""/>  <!--message记录地址-->
```
### .netcore
```
{
  "ssoBaseUrl": "http://www.sso.com:8030/",
  "ssoSecretKey": "",
  "ssoCookieKey": "web1.web.auth",
  "ssoCookieTime": "session",
  "messageBaseUrl": "http://www.messagecenter.com:8050/"
}
```
### 安装 和 配置
1. nuget上搜索 SSO.Util.Client 关键词安装
2. 在配置文件中配置好节点
3. 编译错误处理: 如果出现了 Could not load file or assembly 的错误,请更新相应的程序集到最新版本
### 功能一: SSO验证和权限验证
- `[SSOAuthorize]`  : 需要登录才能访问
- `[SSOAuthorize("GetFileDetail")]` : 到数据库查询是否有 GetFileDetail 权限
- `[AllowAnonymous]` :可以匿名访问
- 把所有标记了 `SSOAuthorize("name")` 的 name部分插入数据库,用来在界面配置权限
   ```
      var assembly = Assembly.GetExecutingAssembly();
      var controllers = assembly.GetTypes().Where(w => w.FullName.Contains("FileService.Api.Controllers"));
      var res = SSOAuthorizeAttribute.GetPermissionDescription(controllers);
      SSOClientService sSOClientService = new SSOClientService(ssoBaseUrl, JwtManager.GetAuthorization(HttpContext));
      var category = AppSettings.GetApplicationUrlTrimHttpPrefix(Request);
      var result = sSOClientService.ReplacePermissions(category, res);
   ```
- 在action中访问用户id
   ```
   var userId = User.Identity.Name;
   ```
- 获取authorization
   ```
   var authorization = JwtManager.GetAuthorization()  //netframework
   var authorization = JwtManager.GetAuthorization(httpContext)  //netcore
   ```
- 访问用户其他信息
   ```
   var userData = JwtManager.GetUserData()  //netframework
   var userData = JwtManager.GetUserData(httpContext)  //netcore
   ```
### 功能二: 日志记录和异常记录
- 注册全局错误处理器 `MyHandleErrorAttribute`
- 把日志记录在当前项目的 App_Data\ 文件夹中(使用的是log4net)
   ```
   Log4Net.InfoLog("xx");  //详情日志
   Log4Net.ErrorLog("xx"); //异常日志
   ```
- 使用远程api记录日志(节点 `messageBaseUrl` 的地址)
   ```
   [LogRecord]:  //记录日志
   [NoneLogRecord]: //不记录日志
   ```
### 功能三: model验证和返回值标准化
- 注册全局model验证过滤器 `ValidateModelStateAttribute`
- `ErrorCode` : 返回值枚举
- `ResponseModel` :返回值对象(ContentResult的子类,类型为 application/json)
- 案例:
   ```
   return new ResponseModel<string>(ErrorCode.success, "");  //返回字符串
   return new ResponseModel<string>(ErrorCode.success, json);  //返回序列化好的json字符串
   return new ResponseModel<Object>(ErrorCode.success, obj); //返回可以序列化的对象
   ```
- `ServiceModel` :解析返回值对象
- 案例: 
  ```
  var result = requestHelper.Post(baseUrl, model, null); //接口返回值
  return JsonSerializerHelper.Deserialize<ServiceModel<List<LogModel>>>(result);  //解析返回值
  ```
### 功能四: 轻量版操作 ElasticSearch 工具
- 案例
  ```
  List<string> urls = new List<string>(){ "http://localhost:9200/"};
  ElasticConnection elasticConnection = new ElasticConnection(urls);
  //判断index是否存在
  bool exists = elasticConnection.Head(indexName);
  //创建index,并且设置mapping
  var result = elasticConnection.Put(indexName, mapping);
  if (result.Contains("\"acknowledged\":true")) return true;
  //索引数据
  var result = elasticConnection.Post("person/_doc/1", json);
  if (result.Contains("\"successful\":1")) return true;
  //删除数据
  var result = elasticConnection.Delete("person/_doc/1");
  if (result.Contains("\"successful\":1")) return true;
  //搜索数据
  var result = elasticConnection.Post(indexName, json);
  //或者只取需要的数据
  var result = elasticConnection.Post(indexName+ "/_search?filter_path=hits.total,hits.hits._source,hits.hits._id,hits.hits.highlight", json);
  更新数据最佳实践:
  确保id一致使用索引数据接口
  ```
### 功能五: SqlBatisLite 操作 SQL Server 数据库(ORM)
- 配置文件:  
1. sbl.config : 全局配置文件,SqlBatisLite启动时会自动从项目根目录读取该文件  
   文件 Build Action : None  
   文件 Copy To Output Directory : Copy always 或者 Copy if newer  
   案例:
    ```
   <?xml version="1.0" encoding="utf-8" ?>
      <configuration>
         <connectionstring>
            Server=.;Database=T01;User ID=name;Password=123
         </connectionstring>
         <!--创建table-->
         <create-tables resource="create.xml" assembly="SSO.Data" namespace="Create"/>
         <mappings assembly="SSO.Data" namespace="Mappings"/>
      </configuration>
    ```
   `connectionstring` 节点: 数据库连接字符串配置节点  

   `create-tables` 节点: 项目初始化数据表使用,只会在项目启动时候执行一次  
   - `resource` 属性: 文件名,忽略该属性则使用 assembly和namespace组成的文件夹路径里所有的文件,然后根据文件名升序组合文件里面的sql语句
   - `assembly` 属性: 所在的程序集名称   
   - `namespace`属性: 所在的文件夹名称  
  
   `mappings` 节点: sql映射文件配置节点
   - `assembly` 属性: 所在的程序集名称   
   - `namespace`属性: 所在的文件夹名称  
   
2. create.xml : 初始化table的所有sql语句  
   文件 Build Action : Embedded Resource  
   文件 Copy To Output Directory : Do not copy  
   案例:
   ```
   <?xml version="1.0" encoding="utf-8" ?>
   <sql>
      if not exists(SELECT * FROM information_schema.TABLES WHERE table_name ='Company')
      create table Company (
      Id INT IDENTITY NOT NULL,
      Code NVARCHAR(30) not null,
      Name NVARCHAR(50) not null,
      Description NVARCHAR(512) null,
      [Order] INT null,
      UpdateTime DATETIME null,
      CreateTime DATETIME null,
      primary key (Id),
      unique (Code)
      )
   </sql>
   ```
   单个文件(resource+assembly+namespace) : 取出文件中的sql,在项目启动的时候运行一次
   多个文件(assembly+namespace) : 按照文件名升序,然后取出每个文件中sql拼接在一起,在项目启动的时候运行一次
3. *.xml : sql映射文件  
   文件 Build Action : Embedded Resource  
   文件 Copy To Output Directory : Do not copy  

   文件名和对应的类关联,默认关联规则如下  
   对象名 Company -> company.xml  
   对象名 CompanyNews -> company_news.xml  
   也可以在类上加特性标签 [XmlStatement("company")] [XmlStatement("company_news")]来改变默认规则  
   案例: 
   ```
   <?xml version="1.0" encoding="utf-8" ?>
   <sql>
      <get-by-id>
         select * from Company where Id = @Id
      </get-by-id>
   </sql>
   ```
   其中 sql节点是根节点,可以是任意值 , get-by-id 节点将要在程序中引用
- 定义类  
1. 基类(ModelBase): 由使用者定义,必须继承自 EntityBase 类,一个数据库对应一个基类  
  案例: 
   ```
      public abstract class ModelBase : EntityBase
      {
         static SessionFactory sessionFactory = null;
         static ModelBase()
         {
               sessionFactory = new Configuration().Configure();  //默认加载sbl.config.xml
               sessionFactory.CreateTables();  //用配置好的sql语句创建table
         }
         public ModelBase() : base(sessionFactory) { }
         public IEnumerable<T> GetPageList<T>(ref int count, object t, object replacement) where T : class
         {
               return QueryList<T>("get-page-list", t, replacement, ref count);
         }
      }
      ```
   其中 SessionFactory 是单例模式，在调用 Configure()方法时，默认读取 sbl.config.xml 配置文件,如果有不同的数据库,则在 Configure() 指定不同的全局配置文件
2. 数据访问类: 必须继承自基类(ModelBase)  
   案例:
   ```
   [XmlStatement("company")]
   public class Company : ModelBase
   {
      public Company() { }
      public int Id { get; set; }
      public string Code { get; set; }
      public string Name { get; set; }
      public string Description { get; set; }
      public int Order { get; set; }
      public DateTime? UpdateTime { get; set; }
      public DateTime? CreateTime { get; set; }

      public T GetById<T>(int id) where T : class
      {
         return QueryObject<T>("get-by-id", new { Id = id }, null);
      }
   }
   ```
   默认 Company 类和 company.xml 对应,如果要改变对应关系,需要修改 [XmlStatement("company")] 属性  
   Company类具有以下能力:
   1. 数据插入: new Company().Insert(obj)   //自动查找 company.xml 中的insert节点
   2. 数据修改: new Company().Update(obj)   //自动查找 company.xml 中的update节点
   3. 数据删除: new Company().Delete(new { Ids = ids })  //自动查找company.xml 中的delete节点
   4. 根据id查找: new Company().GetById(id) //自动查找 company.xml 中的get-by-id节点

- 动态sql   
   `isNotEmpty` 指定的属性不为null并且不为""
   - property : 要对比的属性
   - prepend : 前置语句

   `IsEmpty` 指定的属性为null或者为""
   - property : 要对比的属性
   - prepend : 前置语句 

   `isNotNull` 指定的属性不为null
   - property : 要对比的属性
   - prepend : 前置语句 

   `IsNull` 指定的属性为null
   - property : 要对比的属性
   - prepend : 前置语句 

   `isEquals` 指定的属性值等于value:
   - property : 要对比的属性
   - prepend : 前置语句 
   - value : 要对比的值
   
   `isNotEquals` 指定的属性值不等于value
   - property : 要对比的属性
   - prepend : 前置语句 
   - value : 要对比的值

   `IsGreaterThan` 指定的属性值大于value
   - property : 要对比的属性
   - prepend : 前置语句 
   - value : 要对比的值

   `IsGreaterEqual` 指定的属性值大于等于value
   - property : 要对比的属性
   - prepend : 前置语句 
   - value : 要对比的值

   `IsLessThan` 指定的属性值小于value: 
   - property : 要对比的属性
   - prepend : 前置语句 
   - value : 要对比的值

   `IsLessEqual` 指定的属性值小于等于value: 
   - property : 要对比的属性
   - prepend : 前置语句 
   - value : 要对比的值

   `iterate` 迭代属性
   - property : 要迭代的属性
   - prepend : 前置语句 
   - conjunction : 用该语句连接每次的迭代结果
   - open : 每次迭代开始语句
   - close : 每次迭代结束语句

   `include` 包含节点
   - id : 要包含的节点名称,绝对路径 id="company.sql-common" 或者 id="sql-common"
  
   
   
### 其他工具方法  
> var str =  AppSettings.GetValue("key")  //获取配置文件
> `AsymmetricEncryptHelper`:   //非对称加密解密类  
> `Base64SecureURL`: //url安全的base64加密解密  
> `DateTimeExtention`: 获取时间戳  
> `HashEncryptHelper`: 散列算法  
> `HttpRequestHelper`: http请求方法  
> `ImageExtention`: 图片方法  
> `JsonSerializerHelper`: json序列化方法  
> `Log4Net`:日志记录方法  
> `MsQueue`:消息队列方法  
> `RandomExtention`:随机数  
> `StreamExtention`: 流扩展(流的md5,sha256)  
> `StringExtention`: 字符串扩展(md5,sha256)  
> `SymmetricEncryptHelper`: 对称加密算法  
> `JwtManager`: JwtToken 生成类 
> `FileClientService`: 文件服务类

﻿# LogCenter 使用文档
```
文档已经集成在 SSO.Util.Client 中请参考
https://github.com/wangyang602117818/SSO.Util.Client
```
```
https://github.com/wangyang602117818/MessageCenter.Web
```
