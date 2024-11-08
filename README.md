

# 








提供会员注册服务，用户必须注册成会员才能享受应用提供的服务，如浏览和发布新闻， 但有些服务又需要指定角色的会员才能操作，如所有会员都可以浏览新闻，只有管理员(admin)角色的会员才可以发布新闻。


有 2 个公开的 api:


1. CheckName：判断用户名是否可用；
2. Register：根据用户名注册会员；



## 1 声明接口，创建基于 .Net Core 6\.0 的类库项目，命名为 Register.IServices




### 1\.1 添加 jimu 引用






```
Install-Package  Jimu
```




### 1\.2 创建 dto 类






```
using System;
using System.Collections.Generic;
using System.Text;

namespace Register.IServices
{
    public class Member
    {
        public Guid Id { get; set; }
        public string Name { get; set; }
        public string NickName { get; set; }
        public string Pwd { get; set; }

    }
}
```




### 1\.3 声明公开的微服务接口



微服务的定义规则：


1. 必须继承 IJimuService 接口
2. 声明路由属性 \[JimuServiceRoute()]
3. 方法添加属性 \[JimuService()]，该方法才会注册成公开的微服务


Jimu 支持异步方法， 如下面的 Register


下面的两个方法，未设置 EnableAuthorization \= true(默认为 false),都可以匿名访问





```
using System;
using System.Threading.Tasks;
using Jimu;

namespace Register.IServices
{
    [JimuServiceRoute("/api/v1/register")]
    public interface IRegisterService : IJimuService
    {
        [JimuService(CreatedBy = "grissom", CreatedDate = "2018-07-17", Comment = "check member name whether is valid")]
        bool CheckName(string name);
        [JimuService(CreatedBy = "grissom", CreatedDate = "2018-07-17", Comment = "register member")]
        Task<bool> Register(string name, string nickname, string pwd);

    }
}
```




## 2 实现接口，创建基于 .Net Core 6\.0 的类库项目，命名为 Register.Services




### 2\.1 添加对接口项目 Register.IServices 的引用


:[FlowerCloud机场](https://hanlianfangzhi.com)

### 2\.2 实现接口






```
using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;
using Register.IServices;

namespace Register.Services
{
    public class RegisterService : IRegisterService
    {
        static List _membersDb = new List();

        public bool CheckName(string name)
        {
            return !_membersDb.Any(x => x.Name == name);
        }

        public Task<bool> Register(string name, string nickname, string pwd)
        {
            if (!CheckName(name))
            {
                return Task.FromResult(false);
            }
            _membersDb.Add(new Member { Id = Guid.NewGuid(), Name = name, NickName = nickname, Pwd = pwd });
            return Task.FromResult(true);

        }
    }
}
```




## 3 微服务的宿主服务器，创建基于 .Net Core 6\.0 的控制台应用， 命名为 Register.Server




### 3\.1 添加对项目： Register.Services 的引用




### 3\.2 添加 jimu.server 和 Jimu.Common.Discovery.ConsulIntegration 引用






```
Install-Package  Jimu.Server
Install-Package  Jimu.Common.Discovery.ConsulIntegration
```




### 3\.3 启动 jimu 服务代码






```
using Jimu.Server;
using System;

namespace Register.Server
{
    class Program
    {
        static void Main(string[] args)
        {
            var builder = new ServiceHostServerBuilder(new Autofac.ContainerBuilder())
             .UseLog4netLogger() // 使用 log4net 记录日志
             .LoadServices("Register.IServices", "Register.Services") // 加载服务
             .UseDotNettyForTransfer("127.0.0.1", 8001) // DotNetty 监听 8001 端口进行通讯
             .UseConsulForDiscovery("127.0.0.1", 8500, "JimuService", $"127.0.0.1:8001") // 使用 consul, "JimiService" 指定注册服务时，key 带上的前缀，相当于服务分组，$"127.0.0.1:8001" 指定服务宿主的访问地址
             .UseJoseJwtForOAuth(new Jimu.Server.OAuth.JwtAuthorizationOptions
             {
                 SecretKey = "123456", 
             }); // 使用 jwt 进行鉴权，这里只是验证 token,所以只需验证的密钥， 生产 token 在 Auth.Server 服务
            using (var host = builder.Build())
            {
                host.Run(); // 启动服务
                while (true)
                {
                    Console.ReadKey();
                }
            }
        }
    }
}
```


 








