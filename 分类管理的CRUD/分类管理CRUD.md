# 分类管理CRUD

## 查询

- ### 前台请求后台

- ```js
  <v-tree url="/category/list"
                  :isEdit="isEdit"
                  :key="delKey"
                  @handleAdd="handleAdd"
                  @handleEdit="handleEdit"
                  @handleDelete="handleDelete"
                  @handleClick="handleClick"
          />
  ```

- ### 后台定义接口

```java
@ApiOperation(value = "通过查询商品分类")
@GetMapping(value = "category/list")
Result<List<CategoryEntity>> getCategoryByPid(Integer pid);
```

### 实体类

```
@ApiModel(value = "分类实体类")
@Data
@Table(name = "tb_category")
public class CategoryEntity {

    @Id
    @ApiModelProperty(name = "类目id",example = "1")
    @NotNull(message = "id不能为空",groups = {MingruiOperation.Update.class})
    private Integer id;

    @ApiModelProperty(name="类目名称")
    @NotEmpty(message = "分类名称不能为空",groups = {MingruiOperation.Add.class,MingruiOperation.Update.class})
    private String name;

    @ApiModelProperty(name="父类目id,顶级类目填0",example = "1")
    @NotNull(message = "pid不能为空",groups = {MingruiOperation.Add.class})
    private Integer parentId;

    @ApiModelProperty(name="是否为父节点,0为否,1为是",example = "1")
    @NotNull(message = "isParent不能为空",groups = {MingruiOperation.Add.class})
    private Integer isParent;

    @ApiModelProperty(name="排序指数，越小越靠前",example = "1")
    @NotNull(message = "sort不能为空",groups = {MingruiOperation.Add.class})
    private Integer sort;

}
```

### 定义接口实现类

实例一个实体对象

将传过来的 pid 对它赋值

查询这个实例对象并返回list集合

```java
@Override
public Result<List<CategoryEntity>> getCategoryByPid(Integer pid) {

    CategoryEntity categoryEntity = new CategoryEntity();
    categoryEntity.setParentId(pid);

    List<CategoryEntity> list = categoryMapper.select(categoryEntity);

    return this.setResultSuccess(list);
}
```

### 定义Mapper

```java
public interface CategoryMapper extends Mapper<CategoryEntity> {

    @Select(value = "select id,name from tb_category where id in (SELECT category_id from tb_category_brand where brand_id = #{brandId})")
    List<CategoryEntity> getCategoryByBrandId(Integer brandId);
}
```

## 删除

只能先删除完所有子节点才能删除当前父节点

从前台传来id要删除的

```ajax
handleDelete(id) {
        this.$http.delete('/category/del?id='+10000).then(resp=>{
          if(resp.data.code != 200){
            this.$message.error('删除失败:'+resp.data.message);
            return;
          }
          this.key = new Date().getTime();
          this.$message.success('删除成功!')
        }).catch(error => console.log(error))
      }
```

定义接口

```java
@ApiOperation(value = "删除商品分类")
@DeleteMapping(value = "category/delete")
Result<Object> deleteCategoryById(Integer id);
```

到了后台先判断传来的id是否为空或小于等于0 是则return error('id不合法')

```java
if (null == id || id<= 0) return this.setResultError("id不合法");
```

合法就根据id值查询出来一条完整的数据

```java
CategoryEntity categoryEntity = categoryMapper.selectByPrimaryKey(id);
```

判断查询出的数据是否存在(是否为空)

```java
if (null == categoryEntity) return  this.setResultError("数据不存在");
```

根据查询的数据 判断IsParent是否为1,为1则为父节点不能被删除

```java
if (categoryEntity.getIsParent() == 1) return this.setResultError("当前节点为父节点");
```

实例化一个Example对象,参数为要查询的实体类,调用方法填入查询条件的属性名

mapper调用根据Example查询的方法,返回多条数据,用list接收

```java
Example example = new Example(categoryEntity.getClass());     example.createCriteria().andEqualTo("parentId",categoryEntity.getParentId());
List<CategoryEntity>categoryEntityList=categoryMapper.selectByExample(example);
```

判断list长度,如果大于1,就不用修改父类状态

因为修改父类变为子的状态需要该父类已经没有子类是才可修改

现在实际进行的是删除操作,剩余一条数据的时候改数据会被删除,所以只要剩余最后一条数据或者没有数据就可以修改

如果小于等于1,实例化对象,把IsParent修改为0,把要修改的数据id修改为ParentId,调用修改方法,完成对状态的修改

```java
if (categoryEntityList.size() <= 1){
    CategoryEntity UpdateCategory = new CategoryEntity();
    UpdateCategory.setIsParent(0);
    UpdateCategory.setId(categoryEntity.getParentId());
    categoryMapper.updateByPrimaryKeySelective(UpdateCategory);
}
```

最后就是我们这个方法实际要进行的操作	根据传来的id删除数据

```java
categoryMapper.deleteByPrimaryKey(id);
```

最后return方法成功

```java
return this.setResultSuccess();
```

## 修改

### 定义接口

```java
@ApiOperation(value = "修改")
@PutMapping(value = "/category/edit")//声明哪个组下面的参数参加校验-->当前是校验Update组
Result<Object> editCategoryById(@Validated(MingruiOperation.Update.class) @RequestBody CategoryEntity entity);
```

### 实现类

```java
@Override
@Transactional
public Result<Object> editCategoryById(CategoryEntity entity) {

    categoryMapper.updateByPrimaryKeySelective(entity);
    return this.setResultSuccess();
}
```

## 新增

接口

```java
@ApiOperation(value = "新增")
@PostMapping(value = "/category/add")//声明哪个组下面的参数参加校验-->当前是校验Add组
Result<Object> addCategorybyquery(@Validated(MingruiOperation.Add.class)  @RequestBody CategoryEntity entity);
```

实现类

根据新增时传递的pid 能查询到 父类的id

通过父类的id获取父类到isParent（是否为父类id）

判断，如果不是父类id的话，将当前节点设置为父类id

新增操作

```java
@Override
@Transactional
public Result<Object> addCategorybyquery(CategoryEntity entity) {
    //查询这条新增数据的pid 也就是父类的id
    CategoryEntity categoryEntity = categoryMapper.selectByPrimaryKey(entity.getParentId());

    //判断父类是否为父节点
    if(categoryEntity.getIsParent() != 1){

        //判断新增时 将当前节点设置为父节点
        CategoryEntity updateCategoryEntity2 = new CategoryEntity();
        updateCategoryEntity2.setId(entity.getParentId());
        updateCategoryEntity2.setIsParent(1);
        categoryMapper.updateByPrimaryKeySelective(updateCategoryEntity2);
    }

    categoryMapper.insertSelective(entity);
    return this.setResultSuccess();
}
```