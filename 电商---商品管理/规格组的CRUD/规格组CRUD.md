# 规格组CRUD

## 规格组查询

### 1.前台传递cid进行查询

```js
loadData(){
          this.$http.get("/spec/list/?cid=" + this.cid)
          .then((resp) => {
              this.groups = resp.data.data;
          })
          .catch(() => {
              this.groups = [];
          })
      },
```

### 2.后台用DTO进行查询，并返回List集合

```java
@ApiOperation("规格组查询")
@GetMapping("spec/list/")
Result<List<SpecificationEntity>> getSpecicationList(SpecificationDTO specificationDTO);
```

### 3.实体类SpecificationEntity

```java
@Table(name = "tb_spec_group")
@Data
public class SpecificationEntity {

    @Id
    private Integer id;

    private Integer cid;

    private String name;


}
```

### 4.规格组传递DTO

```java
@Data
@ApiModel(value = "规格组传递DTO")
public class SpecificationDTO extends BaseDTO {

    @Id
    @ApiModelProperty(value = "主键", example = "1")
    @NotNull(message = "主键不能为空",groups = {MingruiOperation.Update.class})
    private Integer id;

    @ApiModelProperty(value = "类型ID",example = "1")
    @NotNull(message = "类型Id不能为空",groups = {MingruiOperation.Add.class})
    private Integer cid;

    @ApiModelProperty(value = "规格组名称")
    @NotEmpty(message = "规格组名称不能为空",groups = {MingruiOperation.Add.class})
    private String name;

}
```

### 5.接口的实现类

用Example进行条件查询

DTO接受的前台参数，赋值给实体类（实体都是无参数的）让实体类去数据库查询

并返回list集合

```java
@Override
public Result<List<SpecificationEntity>> getSpecicationList(SpecificationDTO specificationDTO) {

    Example example = new Example(SpecificationEntity.class);
    
    example.createCriteria().andEqualTo("cid",
         BaiduBeanUtil.copyProperties(specificationDTO,
                                      SpecificationEntity.class).getCid()
         );

    List<SpecificationEntity> list = specificationMapper.selectByExample(example);

    return this.setResultSuccess(list);
}
```

### 6.mapper接口

```java
public interface SpecificationMapper extends Mapper<SpecificationEntity> {
}
```

## 规格组新增

### 1.前台传递数据到后台

```js
save(){
           this.$http({
            method: this.isEdit ? 'put' : 'post',
            url: '/spec/save/',
            data: this.group
          }).then(() => {
              this.show = false;
              this.$message.success("保存成功！");
              this.loadData();
          }).catch(() => {
              this.$message.error("保存失败！");
            });
      }
```

### 2.后台接口接受

后台接受前台传递的‘’数据‘’（参数）时都需要在接口上添加@RequestBody注解。

```java
@ApiOperation("规格组新增")
@PostMapping("spec/save/")
Result<JSONObject> addSpecification(@RequestBody SpecificationDTO specificationDTO);
```

### 3.接口的实现类

DTO接口的参数赋值给实体进行查询操作就可以了

```java
@Override
@Transactional
public Result<JSONObject> addSpecification(SpecificationDTO specificationDTO) {

    specificationMapper.insertSelective(
            BaiduBeanUtil.copyProperties(specificationDTO,SpecificationEntity.class));

    return this.setResultSuccess();
}
```

## 规格组修改

### 1.前台发送请求到后台和新增用同一个请求

根据RestFul风格区分

### 2.后台定义接口

```java
@ApiOperation("规格组修改")
@PutMapping("spec/save/")
Result<JSONObject> updateSpecification(@RequestBody SpecificationDTO specificationDTO);
```

### 3.接口的实现类

DTO接口的参数赋值给实体进行修改操作

```java
@Override
@Transactional
public Result<JSONObject> updateSpecification(SpecificationDTO specificationDTO) {
    specificationMapper.updateByPrimaryKeySelective(
            BaiduBeanUtil.copyProperties(specificationDTO,SpecificationEntity.class));
    return this.setResultSuccess();
}
```

## 规格组删除

### 1.前台获取id请求后台

```js
deleteGroup(id){
          this.$message.confirm("确认要删除分组吗？")
          .then(() => {
            this.$http.delete("/spec/delete/" + id)
                .then((resp) => {
                  if(resp.data.code != 200){
                    this.$message.error("删除失败")
                    return ;
                  }
                    this.$message.success("删除成功");
                    this.loadData();
                })
          })
      },
```

### 2.后台定义接口

这个方式是RestFul风格传递参数 需要加@PathVariable注解

这个注解@PathVariable(name="") 默认的name属性，需要和@DeleteMapper（/////{id}）里面的参数一致

接口参数和前台发送的参数一致也可以，不过必须加注解

```java
@ApiOperation("规格组删除")
@DeleteMapping("spec/delete/{id}")
Result<JSONObject> deleteSpecification(@PathVariable Integer id);
```

### 3.接口实现类

直接进行删除操作就可以了

```java
@Override
public Result<JSONObject> deleteSpecification(Integer id) {
    //Example example = new Example(SpecParamEntity.class);
    //example.createCriteria().andEqualTo("groupId",id);
    //List<SpecParamEntity> entities = specParamMapper.selectByExample(example);
    //if(entities.size() >= 1) return this.setResultError("规格组内存在参数，不能删");
    specificationMapper.deleteByPrimaryKey(id);
    return this.setResultSuccess();
}
```

## 写完规格参数组后要考虑分类管理的删除，如果节点下带有规格组的话不能被删除

在CategoryServiceImpl包下 的deleteCategoryById方法中加入

```java
//查询删除节点绑定的品牌,存在的话不能被删除
Example example1 = new Example(CategoryBrandEntity.class);
example1.createCriteria().andEqualTo("categoryId", id);
List<CategoryBrandEntity> categoryBrandEntities = categoryBrandMapper.selectByExample(example1);
if(categoryBrandEntities.size() >= 1) return setResultError("此节点被品牌绑定,不能被删除");
```