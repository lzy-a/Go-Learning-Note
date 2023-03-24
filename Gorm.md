# Gorm

### 表名

GORM 使用结构体名的 `蛇形命名` 作为表名。对于结构体 `User`，根据约定，其表名为 `users`。（什么天才设计）

这里有两种方式去修改表名：第一种就是去掉这个默认设置；第二种就是在保留默认设置的基础上通过重新设定表名来替换。

先说如何通过重新设定表名来替换，可以实现 `Tabler` 接口来更改默认表名，例如：

```go
Copytype Tabler interface {
    TableName() string
}

// TableName 会将 User 的表名重写为 `user_new_name`
func (User) TableName() string {
  return "user_new_name"
}
```

通过去掉默认配置上面已经有提，配置 *SingularTable: true* 这选项即可。

```go
db, err := gorm.Open(sqlite.Open("gorm.db"), &gorm.Config{
  NamingStrategy: schema.NamingStrategy{
    TablePrefix: "t_",   // 表名前缀，`User`表为`t_users`
    SingularTable: true, // 使用单数表名，启用该选项后，`User` 表将是`user`
    NameReplacer: strings.NewReplacer("CID", "Cid"), // 在转为数据库名称之前，使用NameReplacer更改结构/字段名称。
  },
})
```

**一般来说这里是一定要配置 \*SingularTable: true\* 这一项的。**

然后db会根据结构体的名称自动识别表名，对于结构体 `User`，根据约定，其表名为 `users`。如果改了singulartable的话就是user表，如果设置了TablePrefix:"t_"，表名则为t_user。

### 索引

### 预加载

GORM 的 `Preload` 方法实际上是在进行 SQL 查询时，通过关联表查询将关联的数据预先加载到内存中，避免了后续每次访问关联数据都需要重新查询数据库的性能问题。

预加载的数据会存储在内存中的对象中，而不是数据库中。当您执行查询时，GORM 将使用 `JOIN` 操作将主要记录与预加载的记录连接在一起，并将预加载的记录填充到主记录的关联字段中。这意味着您可以在不执行额外查询的情况下访问关联数据。

`Preload` 方法是 GORM 中的一种预加载方法，可以用于优化与关联模型的查询。它可以通过单个 SQL 查询获取关联的模型，从而减少查询次数和提高性能。

下面是 `Preload` 方法的使用示例：

假设我们有一个 `Article` 模型和一个 `Tag` 模型，它们之间有一个 `many-to-one` 的关系，也就是一个标签可以对应多个文章，但一个文章只能对应一个标签。在查询文章时，我们希望同时获取与文章关联的标签信息，而不是在后续操作中多次查询数据库获取标签信息。这时可以使用 `Preload` 方法来实现预加载：

```go
var articles []Article
db.Preload("Tag").Find(&articles)
```

这里的 `Preload("Tag")` 意味着要预加载与文章关联的标签模型，这样可以避免在后续操作中进行多次查询。这个方法将在获取文章数据的同时获取标签数据，并将标签信息关联到文章结构体中，这样在后续操作中可以直接使用。

`Preload` 方法还可以用于预加载多个关联模型，例如：

```go
var articles []Article
db.Preload("Tag").Preload("Category").Find(&articles)
```

这里的 `Preload("Category")` 意味着要预加载与文章关联的分类模型，这样就可以同时获取文章的标签和分类信息，从而减少查询次数和提高性能。

### 增删改查

#### 1.数据创建

```go
db.Create(&Article{
		TagID:     data["tag_id"].(int),
		Title:     data["title"].(string),
		Desc:      data["desc"].(string),
		Content:   data["content"].(string),
		CreatedBy: data["created_by"].(string),
		State:     data["state"].(int),
	})

```

直接db.Create()里边是**指针**。

在 GORM 中，使用 `db.Create()` 方法创建新的记录时，参数必须是一个指向模型对象的指针，即一个 `&` 操作符后面跟着模型对象。这是因为 `Create` 方法需要修改模型对象的属性来反映新的数据库记录，而仅仅传递模型对象是无法做到的。

具体来说，当你调用 `db.Create(&model)` 方法时，GORM 会将模型对象的数据转化成一个 SQL 插入语句，并将其发送给数据库服务器。插入完成后，**GORM 会自动更新模型对象的属性，以便它与新插入的记录保持同步**。因此，如果你传递的是一个模型对象而不是指针，那么 GORM 无法修改模型对象的属性，因此无法反映新的数据库记录。模型对象的属性修改后，可以通过访问模型对象的属性来访问新插入的记录的值。**这对于在插入记录后需要立即获取插入的记录的 ID 或自动填充的时间戳等信息非常有用。**

#### 2.数据查询

```go
db.Preload("Tag").Where(maps).Offset(pageNum).Limit(pageSize).Find(&articles)
```

##### 预加载

首先是 `Preload` 方法，它用于预加载一个关联的模型。在这个例子中，`Preload("Tag")` 意味着要预加载与文章关联的标签，这样可以避免在后续操作中进行多次查询。预加载可以提高性能，尤其是当需要处理大量数据时。

接下来是 `Where` 方法，它用于将一些条件应用于数据库查询。在这个例子中，`Where(maps)` 意味着要将 `maps` 中的条件应用于查询。这个 `maps` 变量应该是一个 `map[string]interface{}` 类型的映射，其中键是列名，值是要匹配的值。where和preload顺序无影响。

##### 分页

然后是 `Offset` 和 `Limit` 方法，它们用于设置查询结果的偏移量和限制数量。在这个例子中，`Offset(pageNum)` 意味着从查询结果的第 `pageNum` 条记录开始获取，`Limit(pageSize)` 意味着只获取最多 `pageSize` 条记录。

最后是 `Find` 方法，它用于执行查询并将结果映射到一个结构体的切片中。在这个例子中，`Find(&articles)` 意味着将查询结果映射到 `&articles` 变量所指向的结构体切片中。

条件在前，结构体指针在后。

##### 关联关系

`.Association()` 方法用于获取某个关联关系的 `gorm.Association` 实例，以便进行关联关系的操作。

#### 3.数据修改

```go
db.Model(&Article{}).Where("id = ?", id).Updates(data)
```

`&Article{}` 是一个空的 `Article` 结构体指针，用于指定需要更新的表。

`Updates(data)` 是用于指定更新哪些字段以及它们的值，其中 `data` 是一个 `map` 类型的变量，用于存储需要更新的字段和它们的值。例如，如果要更新 `Title` 字段的值为 `"New Title"`，可以使用 `data := map[string]interface{}{"Title": "New Title"}`。

需要注意的是，`Updates()` 方法会更新指定记录中的所有非零值字段，如果某个字段的值为零值（例如 `0`、`""` 或 `nil`），则该字段不会被更新。如果要更新某个字段的零值，可以使用 `Update()` 方法，并且需要明确指定要更新的字段。

#### 4.数据删除

```go
db.Where("id = ?", id).Delete(Article{})
//或
var article Article
db.First(&article, id)
db.Delete(&article)
```

GORM 的删除操作是软删除，即默认情况下只是将 `deleted_at` 字段更新为当前时间，而不是直接从数据库中删除记录。

### 路由参数

以/为分割符，每个参数以“：”为参数表示动态变量，会自动绑定到路由对应的参数上路由规则:[:]表示可以不用匹配。

`c.Params("name")`

```go
//http://localhost:8080/user/李四/20/北京/男
router.GET("/user/:name/:age/:address/:sex", func(c *gin.Context) {
    //打印URL中所有参数
    //"[{name 李四} {age 20} {address 北京} {sex 男}]\n"
    c.JSON(http.StatusOK, fmt.Sprintln(c.Params))
})
```

`c.DefaultQuery()`和`c.Query()`

```go
//http://localhost:8080/login?name=张三&password=123456
//GET请求
router.GET("/login", func(c *gin.Context) {
  //{ name: "张三", password: "123456" }
  c.JSON(http.StatusOK, gin.H{
    "name":     c.DefaultQuery("name", "默认张三"),
     //"name":     c.Query("name"),
    "password": c.DefaultQuery("password", "默认密码"),
  })
})

//POST请求
router.POST("/login", func(c *gin.Context) {
//{"name":"张三","password":"默认密码"}
	c.JSON(http.StatusOK, gin.H{
	  "name":     c.DefaultQuery("name", "默认张三"),
	  "password": c.DefaultQuery("password", "默认密码"),
	})
})
```

### Hook机制

[gorm hook使用中的问题及核心源码解读 - 腾讯云开发者社区-腾讯云 (tencent.com)](https://cloud.tencent.com/developer/article/1830811)

hook只能定义在model上，不能定义在gorm.DB上。

定义了一个model之后就可以实现接口函数了。常用的有Before/After+Save/Update/Create/Delete()

```go
//gorm v2.0写法
func (tag *Tag) BeforeCreate(tx *gorm.DB) error {
	tx.Statement.SetColumn("CreatedOn", time.Now().Unix())
	return nil
}

func (tag *Tag) BeforeUpdate(tx *gorm.DB) error {
	tx.Statement.SetColumn("ModifiedOn", time.Now().Unix())
	return nil
}
```

`tx.Statement`是一个指向当前语句的指针。你可以使用它来设置列值、条件等。

`SetColumn`是GORM中的一个方法，用于设置当前语句的列值。

### Beego valid验证

```go
valid := validation.Validation{}
valid.Required(name, "name").Message("名称不能为空")
valid.MaxSize(name, 100, "name").Message("名称最长为100字符")
valid.Required(createdBy, "created_by").Message("创建人不能为空")
valid.MaxSize(createdBy, 100, "created_by").Message("创建人最长为100字符")
valid.Range(state, 0, 1, "state").Message("状态只允许0或1")
```

这里的代码通过创建一个 `validation.Validation` 实例来进行数据验证。然后，我们可以使用这个实例的各种验证方法（例如 `Required()`、`MaxSize()`、`Range()` 等）来对数据进行验证。

验证结果可以通过 `valid.HasErrors()` 来判断是否有错误，错误信息可以通过 `valid.Errors` 来获取。