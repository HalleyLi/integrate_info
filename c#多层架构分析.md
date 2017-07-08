

##  C#多层架构分析

@(综合)


**架构分析：**

思路：从测试代码层层深入式分析，该单元测试时对Service层进行的测试，即对服务层代码进行正确、错误的测试，保证覆盖率达到80%以上

架构图：


**Controller层**：控制器层，规则：CSM+Bank+Controller
先上一段该层的简单伪代码：

```
[Resource("CSMBankRes")]
[RoutePrefix(CSMConsts.URL_PREFIX_CSP)]
public class CSMBankController:BaseController<ICSMBankService>
{
	[HttpPost]
	[Route("bank")]
	[ActionName("addBank")]
	public WapInt32 CreateBank([FromBody]CSMBankDto bank){
		  var result = Service.CreateBank(companyaccount);
        return new WapInt32(result);
	}
}
```
代码分析：
- 该控制器层继承自BaseController抽象类（可传入多个泛型服务接口），用来获取对应的服务，用于该层相应方法的调用。所有控制器都应该继承当前控制器
- 每个控制器类都顶部都会用Resource资源特性，用于标识每个控制器所对应的资源；每种特性作为一个类，尾部约定为Attribute，类用`[AttributeUsage(AttributeTargets.Class)]`它标识。
- WebApi的定义必须符合RestFul规则：（representational State Transfer）:资源、表现层、状态转换三个概念；有兴趣的可以google搜索了解一下，对应的一些规则不在此叙述；
- `WapInt32`是对返回值进行封装，继承至WapResponse类型；


**Contracts层**：服务接口的定义，规则：I+CSM+Bank+Service
**ServiceI层**：实现Contracts接口方法，规则：CSM+Bank+Service+impl
##### 	- 该层会继承**BaseService:抽象的服务计类*，该类实现了服务需要的基本功能，

```
  public abstract class BaseService
{
    /// <summary>
    /// 根据服务类型获取服务对象实例
    /// </summary>
    /// <typeparam name="TService">服务类型</typeparam>
    /// <returns>返回服务对象实例</returns>
    protected TService GetService<TService>()
    {
        return ServiceFactory.GetService<TService>();
    }
}
```
代码分析：该抽象类中提供了一个受保护的，根据传入泛型服务的类型返回对应的服务对象实例，通过`ServiceFactory`静态工厂对象中的GetService方法实现。该ServiceFactory静态类实现如下：

```
  public static class ServiceFactory
    {
        /// <summary>
        /// 定义服务工厂同步锁对象
        /// </summary>
        private static readonly object _SyncLock = new object();

        /// <summary>
        /// 服务容器对象
        /// </summary>
        private static IUnityContainer serviceContainer = null;

        /// <summary>
        /// 获取指定契约对应的服务
        /// </summary>
        /// <typeparam name="TService">服务契约类型</typeparam>
        /// <returns>返回服务实例</returns>
        public static TService GetService<TService>()
        {
            if (serviceContainer == null)
            {
                lock (_SyncLock)
                {
                    if (serviceContainer == null)
                    {
                        serviceContainer = UnityHelper.GetContainer(Consts.CONFIG_FILE_UNITY_SERVICE, null);
                    }
                }
            }
            return serviceContainer.Resolve<TService>();
        }
    }
```

代码分析：该静态类中定义了一个静态只读的锁对象，在获取指定契约对应的服务中，通过锁住该对象，保证的线程的同步性，一次只能有一个线程对它进行访问，同样的，针对该锁对象的操作并不是只有获取和释放，还有等待和缓冲，这个暂时不加以探讨。针对`UnityHelpler.GetContainer`获取Unity容器相关的方法，本文中不加以讨论，仅提供对应的UnityHelper.cs和`ConfigurationHelper.cs`两个类的方法。

```
/// <summary>
    /// 定义Unity辅助类，用于操作Unity相关内容
    /// </summary>
    public static class UnityHelper
    {
        /// <summary>
        /// 保存Unity容器
        /// </summary>
        private static Dictionary<string, IUnityContainer> _Containers
            = new Dictionary<string, IUnityContainer>();

        /// <summary>
        /// 创建并设置一个Unity对象容器
        /// </summary>
        /// <param name="configFilePath">配置文件路径</param>
        /// <param name="containerName">Unity对象容器名称</param>
        /// <returns>返回Unity容器对象</returns>
        public static IUnityContainer GetContainer(string configFilePath, string containerName)
        {
            return GetContainer(configFilePath, containerName, null);
        }

        /// <summary>
        /// 创建并设置一个Unity对象容器
        /// </summary>
        /// <param name="configFilePath">配置文件路径</param>
        /// <param name="containerName">Unity对象容器名称</param>
        /// <param name="parentContainer">父容器对象</param>
        /// <returns>返回Unity容器对象</returns>
        public static IUnityContainer GetContainer(string configFilePath, string containerName, IUnityContainer parentContainer)
        {
            var _containerName = containerName;
            if (_containerName == null) _containerName = string.Empty;
            string containerKey = configFilePath + "_" + (string.IsNullOrEmpty(_containerName) ? "DEFAULT" : _containerName);
            return GetContainer(containerKey, configFilePath, _containerName, parentContainer);
        }

        /// <summary>
        /// 创建并设置一个Unity对象容器
        /// </summary>
        /// <param name="containerKey">容器</param>
        /// <param name="configFilePath">配置文件路径</param>
        /// <param name="containerName">Unity对象容器名称</param>
        /// <param name="parentContainer">父容器对象</param>
        /// <returns>返回Unity容器对象</returns>
        public static IUnityContainer GetContainer(string containerKey, string configFilePath, string containerName, IUnityContainer parentContainer)
        {
            try
            {
                if (!_Containers.ContainsKey(containerKey))
                {
                    var _containerName = containerName;
                    if (_containerName == null) _containerName = string.Empty;
                    UnityConfigurationSection unitySection =
                        ConfigurationHelper.GetSection<UnityConfigurationSection>(configFilePath, UnityConfigurationSection.SectionName);
                    IUnityContainer container = GetContainer(unitySection, _containerName, parentContainer);
                    _Containers[containerKey] = container;
                }
                return _Containers[containerKey];
            }
            catch (Exception ex)
            {
                LogManager.Get().Throw(ex);
                return null;
            }
        }

        /// <summary>
        /// 创建并设置一个Unity对象容器
        /// </summary>
        /// <param name="section">Unity配置对象</param>
        /// <param name="containerName">Unity容器名称</param>
        /// <param name="parentContainer">父容器对象</param>
        /// <returns>返回Unity容器对象</returns>
        public static IUnityContainer GetContainer(UnityConfigurationSection section, string containerName, IUnityContainer parentContainer)
        {
            if (section == null)
                throw new ArgumentNullException("section");
            IUnityContainer container = null;
            if (parentContainer == null) container = new UnityContainer();
            else container = parentContainer.CreateChildContainer();
            if (string.IsNullOrEmpty(containerName)) section.Configure(container);
            else section.Configure(container, containerName);
            return container;
        }
    }
```

```
/// <summary>
    /// 定义获取.config文件的辅助函数类
    /// </summary>
    public static class ConfigurationHelper
    {
        /// <summary>
        /// 加载指定的配置文件
        /// </summary>
        /// <param name="filePath">配置文件路径</param>
        /// <returns>
        /// 如果指定的配置文件存在返回对应的配置对象，否则返回NULL
        /// </returns>
        /// <exception cref="System.IO.FileNotFoundException">无法打开配置文件。</exception>
        public static Configuration GetConfig(string filePath)
        {
            if (string.IsNullOrEmpty(filePath))
                throw new ArgumentNullException("filePath");
            string fullPath = Path.Combine(AppDomain.CurrentDomain.BaseDirectory, filePath);
            if (!File.Exists(fullPath))
                throw new FileNotFoundException("无法打开配置文件。", filePath);
            var fileMap = new ExeConfigurationFileMap { ExeConfigFilename = fullPath };
            var configuration = ConfigurationManager.OpenMappedExeConfiguration(fileMap, ConfigurationUserLevel.None);
            return configuration;
        }

        /// <summary>
        /// 获取指定的配置块
        /// </summary>
        /// <typeparam name="TSection">配置类型</typeparam>
        /// <param name="filePath">配置文件路径</param>
        /// <param name="sectionName">配置块名称</param>
        /// <returns>返回配置块对象实例</returns>
        public static TSection GetSection<TSection>(string filePath, string sectionName) where TSection : class
        {
            if (string.IsNullOrEmpty(sectionName))
                throw new ArgumentNullException("sectionName");
            Configuration configuration = null;
            if (string.IsNullOrEmpty(filePath))
                configuration = ConfigurationManager.OpenExeConfiguration(ConfigurationUserLevel.None);
            else configuration = GetConfig(filePath);
            if (configuration != null)
                return configuration.GetSection(sectionName) as TSection;
            return null;
        }

        /// <summary>
        /// 获取指定的配置块
        /// </summary>
        /// <typeparam name="TSection">配置类型</typeparam>
        /// <param name="sectionName">配置块名称</param>
        /// <returns>返回配置块对象实例</returns>
        public static TSection GetSection<TSection>(string sectionName) where TSection : class
        {
            if (string.IsNullOrEmpty(sectionName))
                throw new ArgumentNullException("sectionName");
            return ConfigurationManager.GetSection(sectionName) as TSection;
        }
    }
```


###### -  Service层会实现一个public的构造函数，用来获取需要的仓储接口对象，在服务层相关方法中，可调用仓储接口中相关方法。该仓储对象会在下面讨论。接下来贴出该部分构造函数部分伪代码：

```
private ICSMBankRepository _bankRepository=null;
// 构造函数
public CSMBankServiceImpl(ICSMBankRepository repo){
	_bankRepository=repo
}
//service层相关方法

public int CreateBank(CSMBankDto Dto){
	if(Dto==null)
		throw new WapException(CSMStateCode.CODE_ARGUMENT_NULL, "银行信息为空！");
	 var res=this._bankRepository.Create(Dto.ToModel());
	 return res.HasValue? res.Value: -1;
}
```
代码分析：该段代码中引出两个类，一个WapException用于异常抛出相关方法。还有一个CSMStateCode静态类，该类中定义了一些const类型的状态码。

##### - 接下来解析Service中的Repository仓储库，后期可根据需要灵活的切换仓储库，该仓储库也会定义相应的接口`ICSMBankRepository` 该层不做分析，主要分析实现层。先上一段伪代码

```
public  class CSMBankRepository: Repository<ICSMStorage>
{
	//相关方法
	public CSMBank GetLasted(){
		return Storage.SelectLasted();
	}
}
```
代码分析：  该层中会继承一个抽象的对象仓库基类，并传入数据访问器对象类型ICSMStorage（该层实际就是最底层SqlServer操作层）；通过传入的泛型获取相应的获取相应的服务实现供该Rpository层调用，当然该内部使用的策略与Service层的`BaseService`思想一样，通过Unity容易，控制反转的方式拿到对应的具体操作类。
下面贴出一段Repository层代码，不做具体的分析，大家可以自己来分析：

```
 /// <summary>
    /// 定义对象仓库类
    /// </summary>
    /// <typeparam name="TStorage">数据访问器对象类型</typeparam>
    public abstract class Repository<TStorage> : IRepository
        where TStorage : class
    {
        /// <summary>
        /// 构造函数
        /// </summary>
        protected Repository()
        {
            StorageProvider = StorageProviderFactory.GetProvider();
        }

        /// <summary>
        /// 获取或设置数据存储提供器对象
        /// </summary>
        protected IStorageProvider StorageProvider { get; set; }

        /// <summary>
        /// 获取数据访问器对象
        /// </summary>
        protected TStorage Storage
        {
            get { return StorageProvider.GetStorage<TStorage>(); }
        }
    }
```




```
  /// <summary>
    /// 定义数据存储提供器工厂对象，用于创建对应的存储提供器对象实例
    /// </summary>
    public static class StorageProviderFactory
    {
        /// <summary>
        /// 内部缓存的数据存储提供器对象
        /// </summary>
        private static IStorageProvider _CachedProvider = new UnityStorageProvider();

        /// <summary>
        /// 创建存储提供器对象
        /// </summary>
        /// <returns>返回存储提供器对象</returns>
        public static IStorageProvider GetProvider()
        {
            return _CachedProvider;
        }
    }
```

**该出用到的工厂创建对象的思想，后期可关注我发布的关于Sequence序列生成 实现思想中会做详细的分析**

##### -  由上层引出的最后一层Storage层，就是数据库相关的操作层，有对应的接口，定义规则：I+CSM+Bank+Storage; 该层不做过多分析，主要我们来分析一下实现层 ：CSM+Bank+Storage

首先来贴出部分伪代码：

```
public class CSMbankStorage:BaseAccess<CSMBank>,ICSMBankStorage
{
	//构造函数
	public CSMBankStorage() 
	  :base(Consts.CONNECTION_STRING)
	{}

	//具体的实现方法
	public CSMBank Selected{
	 try
            {
                string sqlText = @"SELECT TOP 1 * FROM DEMOORDER BY ID DESC ";
                using (var cmd = Database.GetSqlStringCommand(sqlText))
                {
                    return SelectSingle(cmd);
                }
            }
            catch (Exception ex)
            {
                LogManager.Get().Throw(ex);
                return null;
            }
	}
}
```

代码分析：该Storage基层了GDI数据访问器基类`BaseAccess`，通过传入的泛型实体对象来约束该Storage层的操作范围，以及相应的返回方法等；该层主要是包含对数据库操作ExecuteScalar、ExecuteReader等方法的封装；Storage层通过`base(连接字符串)`和相应封装好的SelectList、SelectSingle、Transact等方法，即可完成大部分的数据库操作；该底层代码，暂不贴出；

##### -  实体层Model，没有什么好说的，需要说的是，我们在使用过程中，尽量在Service，Controller中使用Dto封装，Repository层以内我们使用Model实体，Dto与Model的互转使用到的AutoMapper技术，大家可以自己搜资料了解

#####   感觉自己已经很啰嗦了，整个架构大概就这些，开发中的一些细节，比如单元测试中使用到的Moq技术，Unity容器等，有兴趣的可自己Google搜相应的资料

