LiteDB 是一个轻量级的嵌入式 NoSQL 数据库，其设计理念与 MongoDB 类似，但它是完全使用 C# 开发的，因此与 C# 应用程序的集成非常顺畅。与 SQLite 相比，LiteDB 提供了 NoSQL（即键值对）的数据存储方式，并且是一个开源且免费的项目。它适用于桌面、移动以及 Web 应用程序。

### 安装 LiteDB 包

你可以通过 NuGet 包管理器来安装 LiteDB：

```csharp
Install-Package LiteDB
```

### 定义数据模型

定义一个 `FileModel` 类来表示文件信息：

```csharp
using LiteDB;  
  
namespace YourNamespace  
{  
    [BsonCollection("files")] // 可以指定集合名称  
    public class FileModel  
    {  
        [BsonId] // 标记为主键  
        public int Id { get; set; }  
        
        public string Path { get; set; }  
    }  
}
```

### 数据访问服务类

创建一个 `FileServices` 类来封装对 LiteDB 的数据操作：

```csharp
using LiteDB;

namespace UploadLogiData.LiteSql
{
    /// <summary>
    /// Logitech数据操作类
    /// </summary>
    public class FileServices
    {
        /// <summary>
        /// 定义数据库名称,文件在当前程序目录
        /// </summary>
        readonly string database = @"LogiDB.db";

        /// <summary>
        /// 打开一个表
        /// </summary>
        /// <returns></returns>
        public ILiteCollection<FileModel> GetCollection(LiteDatabase db)
        {
            return db.GetCollection<FileModel>("Files");
        }

        /// <summary>
        /// 插入数据(初始化)
        /// </summary>
        public void Initialize()
        {
            // 打开数据库 (如果不存在则创建)
            using (var db = new LiteDatabase(database))
            {
                var cg = GetCollection(db);
                cg.Delete(1);
                FileModel file = new FileModel
                {
                    Id = 1,
                    Path = @"C:\Users\ch190006\Desktop\Loginet\logs",
                };
                cg.Insert(file);
            }
        }

        /// <summary>
        /// 根据主键查询出配置单条数据
        /// </summary>
        /// <returns></returns>
        public FileModel GetSingle(int id)
        {
            using (var db = new LiteDatabase(database))
            {
                var cg = GetCollection(db);
                return cg.FindOne(p1 => p1.Id == id);
            }
        }

        /// <summary>
        /// 更新
        /// </summary>
        /// <param name="value">更新字段</param>
        public void UpdatePath(string value)
        {
            using (var db = new LiteDatabase(database))
            {
                var cg = GetCollection(db);
                // //查询到数据
                var data = cg.FindOne(p1 => p1.Id == 1);
                data.Path = value;
                // 当中数据库中查找到数据后，比如上面的data，可以直接修改后再更新。
                cg.Update(data);
            }
        }

    }
}

```

### 使用示例

```csharp

using System;  
  
namespace YourNamespace  
{  
    class Program  
    {  
        static void Main(string[] args)  
        {  
             FileServices file = new FileServices();
             file.Initialize();
             file.UpdatePath(textLog.Text);
             file.GetSingle(1).Path;
        }  
    }  
}
```

### 使用场景

* **桌面应用程序**：LiteDB 非常适合用于桌面应用程序，因为它是一个嵌入式数据库，可以轻松与应用程序一起打包和分发。它不需要单独的数据库服务器，简化了部署和配置
* **移动应用程序**：由于 LiteDB 的轻量级和嵌入式特性，它也适用于移动应用程序。开发者可以在移动设备上存储和检索数据，而无需依赖远程服务器。
* **小型 Web 应用程序**：对于需要轻量级数据存储解决方案的小型 Web 应用程序，LiteDB 是一个不错的选择。它易于设置和使用，且性能良好。
* **原型设计和测试**：在开发早期阶段或进行原型设计时，使用 LiteDB 可以快速搭建数据存储和检索功能，而无需投入大量时间设置和维护复杂的数据库系统。
* **一个账户/用户**一个数据库的数据存储
* **本地缓存**：在某些情况下，开发者可能希望将数据缓存在本地以提高性能或减少网络延迟。LiteDB 可以作为这种本地缓存的解决方案。