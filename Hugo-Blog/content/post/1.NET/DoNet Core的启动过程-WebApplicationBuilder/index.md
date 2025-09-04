---
title: DoNet Core的启动过程-WebApplicationBuilder
description: 深入解析.NET 6及以上版本中WebApplicationBuilder的启动过程，包括主机配置、应用程序配置、服务注册和中间件设置的完整流程。
date: 2024-05-21
slug: Net/2024-05-21
# image: 2.jpg
categories:
    - .NET Core
    - ASP.NET Core
    - .NET
tags:
    - WebApplicationBuilder
    - .NET 6
    - ASP.NET Core
    - 启动过程
    - 依赖注入
---

## 1. 前言

在.NET 6开始做ASP.NET Core的开发时，我们首先要了解的是应用程序的启动过程。WebApplication和WebApplicationBuilder类是启动过程中不可或缺的核心组件。WebApplicationBuilder负责引导启动，这与之前.NET Core版本中分别使用Program和Startup类的方式有所不同。.NET 6及以上版本直接在Program类中进行引导启动，而WebApplication在Run之前，需要完成四个关键步骤：**主机配置**、**应用程序配置**、**服务注册**、**中间件设置**。

## 2. WebApplicationBuilder概述

### 2.1 什么是WebApplicationBuilder

WebApplicationBuilder是.NET 6引入的新类，它简化了ASP.NET Core应用程序的配置和启动过程。它将之前分散在Program.cs和Startup.cs中的配置逻辑统一到一个地方，使代码更加简洁和易于维护。

### 2.2 创建WebApplicationBuilder实例

使用`WebApplicationBuilder.CreateDefault(args)`方法可以创建一个WebApplicationBuilder的实例：

```csharp
var builder = WebApplicationBuilder.CreateDefault(args);
```

该方法会默认加载一些常见的配置和服务：
- 环境变量配置
- appsettings.json配置文件
- 用户机密配置（开发环境）
- 命令行参数
- 默认日志提供程序
- IIS集成（如果适用）

## 3. 主机配置

### 3.1 配置主机属性

WebApplicationBuilder的`Host`属性是一个IHostBuilder实例，可以用它来配置主机相关的设置：

```csharp
// 配置主机配置
builder.Host.ConfigureHostConfiguration(configHost => 
{ 
    // 添加环境变量前缀
    configHost.AddEnvironmentVariables("MYAPP_"); 
    // 添加自定义配置文件
    configHost.AddJsonFile("hostsettings.json", optional: true);
});

// 配置日志系统
builder.Host.ConfigureLogging(logging => 
{ 
    logging.ClearProviders();
    logging.AddConsole();
    logging.AddDebug();
    logging.SetMinimumLevel(LogLevel.Information);
});
```

### 3.2 配置Web主机

还可以通过`WebHost`属性配置Web相关的设置：

```csharp
builder.WebHost.ConfigureKestrel(options =>
{
    options.ListenLocalhost(5000);
    options.ListenLocalhost(5001, listenOptions =>
    {
        listenOptions.UseHttps();
    });
});

builder.WebHost.UseContentRoot(Directory.GetCurrentDirectory());
builder.WebHost.UseWebRoot("wwwroot");
```

## 4. 应用程序配置

### 4.1 配置系统

通过`builder.Configuration`可以访问应用程序的配置系统：

```csharp
// 添加额外的配置源
builder.Configuration.AddJsonFile("appsettings.json", optional: true, reloadOnChange: true);
builder.Configuration.AddJsonFile($"appsettings.{builder.Environment.EnvironmentName}.json", optional: true);
builder.Configuration.AddEnvironmentVariables();
builder.Configuration.AddCommandLine(args);

// 如果是开发环境，添加用户机密
if (builder.Environment.IsDevelopment())
{
    builder.Configuration.AddUserSecrets<Program>();
}
```

### 4.2 环境配置

可以根据不同环境进行特定配置：

```csharp
if (builder.Environment.IsDevelopment())
{
    // 开发环境特定配置
    builder.Configuration.AddUserSecrets<Program>();
}
else if (builder.Environment.IsProduction())
{
    // 生产环境特定配置
    builder.Configuration.AddAzureKeyVault(/* 配置参数 */);
}
```

## 5. 服务注册

### 5.1 基础服务注册

使用`builder.Services`来访问服务容器并注册服务：

```csharp
// 注册控制器服务
builder.Services.AddControllers();

// 注册API探索服务
builder.Services.AddEndpointsApiExplorer();

// 注册Swagger服务
builder.Services.AddSwaggerGen(c =>
{
    c.SwaggerDoc("v1", new OpenApiInfo { Title = "My API", Version = "v1" });
});
```

### 5.2 自定义服务注册

```csharp
// 注册自定义服务
builder.Services.AddScoped<IUserService, UserService>();
builder.Services.AddSingleton<IConfiguration>(builder.Configuration);
builder.Services.AddTransient<IEmailService, EmailService>();

// 注册数据库上下文
builder.Services.AddDbContext<ApplicationDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));

// 注册身份验证
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        // JWT配置
    });
```

### 5.3 配置选项模式

```csharp
// 配置选项
builder.Services.Configure<DatabaseOptions>(builder.Configuration.GetSection("Database"));
builder.Services.Configure<EmailOptions>(builder.Configuration.GetSection("Email"));
```

## 6. 构建WebApplication

完成所有必要的配置和服务注册后，使用`Build()`方法构建WebApplication实例：

```csharp
var app = builder.Build();
```

这个WebApplication实例包含了所有已配置的中间件、服务、路由等，并准备运行。

## 7. 中间件配置

### 7.1 开发环境中间件

```csharp
if (app.Environment.IsDevelopment()) 
{ 
    app.UseDeveloperExceptionPage(); 
    app.UseSwagger(); 
    app.UseSwaggerUI(c =>
    {
        c.SwaggerEndpoint("/swagger/v1/swagger.json", "My API V1");
    }); 
}
else
{
    app.UseExceptionHandler("/Error");
    app.UseHsts();
}
```

### 7.2 通用中间件配置

```csharp
// HTTPS重定向
app.UseHttpsRedirection();

// 静态文件
app.UseStaticFiles();

// 路由
app.UseRouting();

// 跨域
app.UseCors();

// 身份验证
app.UseAuthentication();

// 授权
app.UseAuthorization();

// 映射控制器
app.MapControllers();

// 映射最小API
app.MapGet("/", () => "Hello World!");
```

### 7.3 自定义中间件

```csharp
// 添加自定义中间件
app.UseMiddleware<RequestLoggingMiddleware>();

// 或者使用内联中间件
app.Use(async (context, next) =>
{
    // 请求处理前的逻辑
    await next.Invoke();
    // 请求处理后的逻辑
});
```

## 8. 运行应用程序

### 8.1 启动应用程序

```csharp
// 同步运行（阻塞当前线程）
app.Run();

// 或者异步运行
// await app.RunAsync();
```

### 8.2 配置启动和关闭事件

```csharp
// 配置应用程序生命周期事件
app.Lifetime.ApplicationStarted.Register(() =>
{
    Console.WriteLine("应用程序已启动");
});

app.Lifetime.ApplicationStopping.Register(() =>
{
    Console.WriteLine("应用程序正在停止");
});

app.Lifetime.ApplicationStopped.Register(() =>
{
    Console.WriteLine("应用程序已停止");
});

app.Run();
```

## 9. 完整示例

以下是一个完整的Program.cs示例：

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.OpenApi.Models;

var builder = WebApplicationBuilder.CreateDefault(args);

// 配置服务
builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen(c =>
{
    c.SwaggerDoc("v1", new OpenApiInfo { Title = "My API", Version = "v1" });
});

// 配置数据库
builder.Services.AddDbContext<ApplicationDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));

// 配置CORS
builder.Services.AddCors(options =>
{
    options.AddPolicy("AllowAll", policy =>
    {
        policy.AllowAnyOrigin()
              .AllowAnyMethod()
              .AllowAnyHeader();
    });
});

// 构建应用程序
var app = builder.Build();

// 配置中间件管道
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();
app.UseCors("AllowAll");
app.UseAuthorization();
app.MapControllers();

// 运行应用程序
app.Run();
```

## 10. 最佳实践

### 10.1 配置管理
- 使用强类型配置选项
- 根据环境分离配置文件
- 敏感信息使用用户机密或密钥管理服务

### 10.2 服务注册
- 遵循依赖注入的最佳实践
- 合理选择服务生命周期（Singleton、Scoped、Transient）
- 使用接口抽象具体实现

### 10.3 中间件顺序
- 异常处理中间件应该最先注册
- 身份验证在授权之前
- 路由中间件的位置很重要

## 总结

WebApplicationBuilder为ASP.NET Core应用程序提供了一种更加简洁和统一的配置方式。它将主机配置、应用程序配置、服务注册和中间件设置整合到一个流畅的API中，使得应用程序的启动过程更加清晰和易于管理。

通过WebApplicationBuilder，开发者可以：
- 更轻松地配置应用程序的各个方面
- 享受更好的开发体验和代码可读性
- 利用.NET 6+的新特性和性能改进
- 构建更加现代化和高效的Web应用程序

掌握WebApplicationBuilder的使用方法对于.NET 6+的ASP.NET Core开发至关重要，它是构建现代Web应用程序的基础。