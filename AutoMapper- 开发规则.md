

##  AutoMapper-Model-Dto-develop-usage-rules

@(开发规则)


## 概述

DTO与Model的相互转换，以及如何使用Mapper组件简化转换过程。

## 规则

转换方法定义在DTO类型中，通过FromModel, ToModel方法实现DTO和Model之间的互转。

1. FromModel定义从一个或多个Model转换为DTO对象。该方法本身是静态的。

   比如:

   ```
   // 单个Model
   public static StudentDto FromModel(Student model)
   // 多个Model 
   public static SchoollDto FromModel(Student item1, Teacher item2)
   ```

2. ToModel是定义从DTO生成Model对象。

   ```
   public Student ToModel()
   ```

   对于一个DTO可以生成多个Model的情况（DTO本身可以是多个Model的组装），定义多个ToModel方法。
   比如：

   ```
   public Teacher ToModelTeacher();
   public Student ToModelStudent();
   ```

## 具体方案

实现DTO中定义的具体转换方法可以完成。最简单的方式就是手工写赋值语句。

比如：

```
public static StudentDto FromModel(Student entity)
{
	var dto = new StudentDto() {
		Code = entity.Code,	
		Name = entity.Name,
		P1 = entity.P1,
		...
	};
	
	return dto;
}
```

还有就是使用Mapper组件。它可以简化以上步骤。筛选后我们采用AutoMapper组件来完成以上工作。

### 示例1：

对于简单的DTO和Model之间结构完全一致（类型、名称），AutoMapper自动为我们提供了Convention。处理起来相对简单。如下：

```


/// <summary>
/// 返回DTO对象
/// </summary>
/// <param name="model"></param>
/// <returns></returns>
public static BMRegionDto FromModel(BMRegion model)
{
    BMRegionDto dto = Mapper.Map<BMRegion, BMRegionDto>(model);                                             
    return dto;
}

/// <summary>
/// 以当前对象状态生成一个Model对象
/// </summary>
/// <returns></returns>
public BMRegion ToModel()
{
    BMRegion region = Mapper.Map<BMRegionDto, BMRegion>(this);
    return region;
}

```

注意，每个模块的互转，需要在Model/MapperProfile.cs类里注册Mapper规则

	CreateMap<BMSubbranchDto, BMSubbranch>();
	CreateMap<BMSubbranch, BMSubbranchDto>();


##### 在BMRegionServiceImpl层里：

1. 转换DTO->Model对象

   ```
   BMRegionDto dto=new BMRegionDto();
   var model = dto.ToModel();
   ```

2. 转换Model->DTO对象

   ```
   var result = repo.GetById(regionId);
   return result != null ? BMRegionDto.FromModel(result) : null;                     
   ```

### 示例2：

> 注：下面涉及到的整体代码，可参考附件AutoMapperTest.cs(Common.Test/AutoMapperTest.cs)。

结构不一致，需映射的字段少，则可使用map.ForMember方法进行单独映射：    

```
// 定义规则
var map = Mapper.CreateMap<BookDto, Publisher>();
// 单个属性映射规则
map.ForMember(d => d.Name, opt => opt.MapFrom(s => s.Publisher));             

BookDto dto = new BookDto() { Publisher = "出版者" };
Publisher publish = Mapper.Map<BookDto, Publisher>(dto);   
```

需映射的字段比较多时，可使用ConstructUsing的方式一次直接定义好所有的映射规则          

```
// 定义规则
var map = Mapper.CreateMap<BookDto, ContactInfo>();
// 多个字段映射规则
map.ConstructUsing(s => new ContactInfo
{
    Blog = s.FirstAuthorBlog,
    Email = s.FirstAuthorEmail,
    Twitter = s.FirstAuthorTwitter
});

BookDto dto = new BookDto
{
    FirstAuthorEmail = "matt.rogen@abc.com",
    FirstAuthorBlog = "matt.amazon.com",
};

ContactInfo contactInfo = Mapper.Map<BookDto, ContactInfo>(dto);                      
```

映射时，若存在类型转换(String->Int)时，使用ConvertUsing方法

```
public void TestConvertStringToInt()
{
	Mapper.CreateMap<string, int>().ConvertUsing(new Int32TypeConverter());
	Mapper.CreateMap<Source, Destination>();

	// 该方法主要用来检查还有那些规则没有写完
	Mapper.AssertConfigurationIsValid();
	
	var source = new Source  { Value1 = "5" };
	Destination result = Mapper.Map<Source, Destination>(source);
	
	result.Value3.Equals(typeof(Destination));
}

public class Int32TypeConverter : ITypeConverter<string, Int32>
{
    public int Convert(ResolutionContext context)
    {
        return System.Convert.ToInt32(context.SourceValue);
    }
}  
```

