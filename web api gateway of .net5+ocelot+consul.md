# .net web api 网关入门
1. dotNet 5
2. ocelot + consul

## ocelot 增加consul服务发现网关访问提示404问题

ocelot 及 consul 配置经反复对比未发现问题，去除服务发现，恢复下游列表定义的方式网关是可用的（列表方式虽然可用，但负载均衡在站点服务不可用时仍然发送请求过去了，没有达到自动切换的效果），推断问题在ocelot注入端。
 1. 原注入ocelot方式
 ```csharp
     services
          .AddOcelot(new ConfigurationBuilder()
               .AddJsonFile("ocelot.json", optional: false, reloadOnChange: true)
               .Build())
           .AddConsul();
 ```
 2. 需调整为
 ```csharp
      services
           .AddOcelot()
           .AddConsul();
 ```
 ocelot.json 放在 program中注入：
 ```csharp
     Host.CreateDefaultBuilder(args)
         .ConfigureAppConfiguration((hostingContext, config) =>
         {
             config.SetBasePath(hostingContext.HostingEnvironment.ContentRootPath)
             .AddJsonFile("ocelot.json")
             .AddEnvironmentVariables();
         })
         .ConfigureWebHostDefaults(webBuilder =>
         {
             webBuilder.UseStartup<Startup>();
         });
 ```
    
 Then runs perfect as expected.

### ocelot & consul nuget package
1. Ocelot
2. Ocelot.Provider.Consul

### ocelot config
```json
{
  "Routes": [
    {
      "DownstreamPathTemplate": "/api/{controller}/{action}",
      "UpstreamPathTemplate": "/yourPrefix/{controller}/{action}",
      "UpstreamHttpMethod": [
        "Get",
        "POST"
      ],
      "DownstreamScheme": "http",
      "UseServiceDiscovery": true,
      "ReRouteIsCaseSensitive": false,
      "ServiceName": "YourServiceName",
      "LoadBalancerOptions": {
        "Type": "RoundRobin"
      }
    },

    {
      "DownstreamPathTemplate": "/yourPrefix/{everything}",
      "UpstreamPathTemplate": "/yourPrefix/{everything}",
      "UpstreamHttpMethod": [
        "Get",
        "POST"
      ],
      "DownstreamScheme": "http",
      "UseServiceDiscovery": true,
      "ReRouteIsCaseSensitive": false,
      "ServiceName": "YourServiceName",
      "LoadBalancerOptions": {
        "Type": "RoundRobin"
      }
    }
  ],
  "GlobalConfiguration": { 
    "ServiceDiscoveryProvider": {
      "Scheme": "http",
      "Host": "localhost",
      "Port": 8500,
      "Type": "Consul"
    }
  }
}
```
### consul services config
```json
{
  "services": [
    {
      "id": "webapi1",
      "name": "YourServiceName",
      "tags": [ "WebApi" ],
      "address": "127.0.0.1",
      "port": 51012,
      "checks": [
        {
          "id": "AbcCheck1",
          "name": "Abc_Check",
          "http": "http://127.0.0.1:51012/weatherforecast",
          "interval": "10s",
          "tls_skip_verify": false,
          "method": "GET",
          "timeout": "1s"
        }
      ]
    },

    {
      "id": "webapi2",
      "name": "YourServiceName",
      "tags": [ "WebApi" ],
      "address": "127.0.0.1",
      "port": 51013,
      "checks": [
        {
          "id": "AbcCheck2",
          "name": "Abc1_Check",
          "http": "http://127.0.0.1:51013/weatherforecast",
          "interval": "10s",
          "tls_skip_verify": false,
          "method": "GET",
          "timeout": "1s"
        }
      ]
    }
  ]
}

```
## Consul, Important: Run Consul as Windows Service
1. 下载安装consul [https://www.consul.io/](https://www.consul.io/).
ps: 下载很慢，超过1个小时

**2. sc 创建服务（创建后到服务中更改启动方式为“自动(延迟启动)”）**
```json
sc.exe create "Consul" binPath= "D:\Consul\consul.exe agent -server -ui -bootstrap-expect=1 -data-dir=D:\Consul -node=consulNode1 -client=0.0.0.0 -bind=YourLocalIP -datacenter=dc1 -config-dir=D:\Consul\consulServices.json" start= auto
```
注:(1)bootstrap-expect=1, windows 单机
(2)不需要“&”带入命令行，否则windows service 启动异常。
(3)延迟自动启动，该服务不能在系统重启过程启动（可能和consul.exe系统启动过程未识别到有关）

![consul健康检查](https://img-blog.csdnimg.cn/20201228102607182.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlbmxhaXNoaXdv,size_16,color_FFFFFF,t_70#pic_center)
### Postman 测试

1. 下游服务全部可用时，可以轮流发送。
2. 下游服务部分不可用时，仍然可以全部发送到可用的服务（不加consul时，ocelot就是服务不可用也发送过去了）

### .net 5  ocelot consul web api gateway 入门总结，欢迎转载:[https://blog.csdn.net/wenlaishiwo/article/details/111830852](https://editor.csdn.net/md?articleId=111830852)
### 参考资源
 (1) [https://www.cnblogs.com/zhaobingwang/p/12424708.html](https://www.cnblogs.com/zhaobingwang/p/12424708.html)
 (2) [https://blog.csdn.net/anqi4868/article/details/101800352](https://blog.csdn.net/anqi4868/article/details/101800352)
 (3) [https://www.cnblogs.com/shanyou/p/6286207.html](https://www.cnblogs.com/shanyou/p/6286207.html)
 (4) [https://learn.hashicorp.com/tutorials/consul/windows-agent](https://learn.hashicorp.com/tutorials/consul/windows-agent)
 (5) [https://ocelot.readthedocs.io/en/latest/index.html](https://ocelot.readthedocs.io/en/latest/index.html)
