# Insql
[![Build status](https://ci.appveyor.com/api/projects/status/92f8ydwwu5nile9q?svg=true)](https://ci.appveyor.com/project/rainrcn/insql)
![](https://img.shields.io/github/license/rainrcn/insql.svg?style=flat)

Insql 是一个轻量级的.NET ORM类库. 对象映射基于Dapper, Sql配置灵感来自于Mybatis.

[中文](https://github.com/rainrcn/insql/blob/master/README.zh_cn.md) | [English](https://github.com/rainrcn/insql/blob/master/README.md)

# Packages

| Package  |  Nuget Stable  |  Downloads  |
| ------------ | ------------ | ------------ |
| [Insql](https://www.nuget.org/packages/Insql/)  | [![Insql](https://img.shields.io/nuget/v/Insql.svg?style=flat)](https://www.nuget.org/packages/Insql/)  |  [![Insql](https://img.shields.io/nuget/dt/Insql.svg?style=flat)](https://www.nuget.org/packages/Insql/)  |
| [Insql.MySql](https://www.nuget.org/packages/Insql.MySql/)  | [![Insql.MySql](https://img.shields.io/nuget/v/Insql.MySql.svg?style=flat)](https://www.nuget.org/packages/Insql.MySql/)  |  [![Insql.MySql](https://img.shields.io/nuget/dt/Insql.MySql.svg?style=flat)](https://www.nuget.org/packages/Insql.MySql/)  |
| [Insql.Oracle](https://www.nuget.org/packages/Insql.Oracle/)  | [![Insql.Oracle](https://img.shields.io/nuget/v/Insql.Oracle.svg?style=flat)](https://www.nuget.org/packages/Insql.Oracle/)  |  [![Insql.Oracle](https://img.shields.io/nuget/dt/Insql.Oracle.svg?style=flat)](https://www.nuget.org/packages/Insql.Oracle/)  |
| [Insql.PostgreSql](https://www.nuget.org/packages/Insql.PostgreSql/)  | [![Insql.PostgreSql](https://img.shields.io/nuget/v/Insql.PostgreSql.svg?style=flat)](https://www.nuget.org/packages/Insql.PostgreSql/)  |  [![Insql.PostgreSql](https://img.shields.io/nuget/dt/Insql.PostgreSql.svg?style=flat)](https://www.nuget.org/packages/Insql.PostgreSql/)  |
| [Insql.Sqlite](https://www.nuget.org/packages/Insql.Sqlite/)  | [![Insql.Sqlite](https://img.shields.io/nuget/v/Insql.Sqlite.svg?style=flat)](https://www.nuget.org/packages/Insql.Sqlite/)  |  [![Insql.Sqlite](https://img.shields.io/nuget/dt/Insql.Sqlite.svg?style=flat)](https://www.nuget.org/packages/Insql.Sqlite/)  |

# 功能
- **支持 DoNet Core 2.0+ && DotNet Framework 4.6.1+**
- **支持依赖注入系统**
- **MyBatis sql xml 语法**
- **多数据库支持**
- **灵活扩展性**
- **使用简单**

# 基本用法
### 添加 Insql
```c#
public void ConfigureServices(IServiceCollection services)
{
    services.AddInsql();

    services.AddInsqlDbContext<UserDbContext>(options =>
    {
      //options.UseSqlServer(this.Configuration.GetConnectionString("sqlserver"));
      options.UseSqlite(this.Configuration.GetConnectionString("sqlite"));
    });
}
```
### 创建 DbContext
```c#
public class UserDbContext : Insql.DbContext  
{
    public UserDbContext(Insql.DbContextOptions<UserDbContext> options) 
		: base(options)
    {
    }

    public IEnumerable<UserInfo> GetUserList(string userName)
    {
        //sqlId = "GetUserList"
        //sqlParam is PlainObject or IDictionary<string,object>
        return this.Query<UserInfo>(nameof(GetUserList), new { userName, userGender = Gender.W });
    }

    public void InsertUser(UserInfo info)
    {
        var userId = this.ExecuteScalar<int>(nameof(InsertUser),info);

        info.UserId = userId;
    }

    public void UpdateUserSelective(UserInfo info)
    {
        this.Execute(nameof(UpdateUserSelective), info);
    }
}
	
//user model
public class UserInfo
{
    public int UserId { get; set; }

    public string UserName { get; set; }

    public Gender? UserGender { get; set; }
}

public enum Gender
{
    M,
    W
}
```
### 创建 DbContext.insql.xml
_创建 `UserDbContext.insql.xml` 文件并且修改这个文件的属性为`嵌入式文件`类型 . `insql type` 与 `UserDbContext` 类型对应._
```xml
<insql type="Example.Domain.Contexts.UserDbContext,Example.Domain" >
  
    <sql id="selectUserColumns">
      select user_id as UserId,user_name as UserName,user_gender as UserGender from user_info
    </sql>

    <select id="GetUserList">
      <include refid="selectUserColumns" />
      <where>
        <if test="userName != null">
          <bind name="likeUserName" value="'%' + userName + '%'" />
          user_name like @likeUserName
        </if>
        <if test="userGender != null and userGender != 'M' ">
          and user_gender = @userGender
        </if>
      </where>
      order by  user_id
    </select>

    <insert id="InsertUser">
      insert into user_info (user_name,user_gender) values (@UserName,@UserGender);
      select last_insert_rowid() from user_info;
    </insert>

    <update id="UpdateUserSelective">
      update user_info
      <set>
        <if test="UserName != null">
          user_name=@UserName,
        </if>
        <if test="UserGender != null">
          user_gender=@UserGender
        </if>
      </set>
      where user_id = @UserId
    </update>
	
</insql>
```
 
### 使用 DbContext
_使用 `UserDbContext` 在Domain Service中或者Web Controller中_
```c#
public class ValuesController : ControllerBase
{
    private readonly UserDbContext userDbContext;

    public ValuesController(UserDbContext userDbContext)
    {
        this.userDbContext = userDbContext;
    }

    [HttpGet]
    public ActionResult<IEnumerable<string>> Get()
    {
        //可以这样使用事务
        this.userDbContext.DoWithTransaction(() =>
        {
            var userInfo = new Domain.UserInfo
            {
                UserName = "loveW",
                UserGender = Domain.Gender.M
            };

            this.userDbContext.InsertUser(userInfo);

            this.userDbContext.UpdateUserSelective(new Domain.UserInfo
            {
                UserId = userInfo.UserId,
                UserName = "loveWWW",
            });
        });

        var list = this.userDbContext.GetUserList("love");
	//todo return
    }
}
```
# 说明文档
详细说明文档请看 : [说明文档](https://github.com/rainrcn/insql/blob/master/doc/doc.zh_cn.md)
