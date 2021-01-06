# 规格组参数CRUD

## 规格组参数查询

### 1.前台发送请求并传递id

```js
loadData() {
      this.$http
        .get("/param/list/",{
          params:{
          groupId:this.group.id
        }})
        .then((resp) => {
          resp.data.data.forEach(p => {
              p.segments = p.segments ? p.segments.split(",").map(s => s.split("-")) : [];
          })
          this.params = resp.data.data;
        })
        .catch(() => {
          this.params = [];
        });
    },
```

### 2.实体类

```java
@Data
@Table(name = "tb_spec_param")
public class SpecParamEntity {

    @Id
    private Integer id;
    private Integer cid;
    private Integer groupId;
    private String name;
	//因为numeric在数据库是个关键字  所以需要在实体中声明一下这个列
    @Column(name = "`numeric`")
    private Boolean numeric;
    private String unit;
    private Boolean generic;
    private Boolean searching;
    private String segments;
}
```

### 3.规格组传递参数DTO

```java
@Data
@ApiModel("规格参数传递DTO")
public class SpecParamDTO {

    @Id
    @ApiModelProperty(name = "主键",example = "1")
    @NotNull(message = "id不能为空",groups = {MingruiOperation.Update.class})
    private Integer id;

    @ApiModelProperty(name="cid",example = "1")
    @NotNull(message = "cid不能为空",groups = {MingruiOperation.Add.class})
    private Integer cid;

    @ApiModelProperty(name="groupId",example = "1")
    @NotNull(message = "groupId不能为空",groups = {MingruiOperation.Add.class})
    private Integer groupId;

    @ApiModelProperty(name="name不能为空")
    @NotEmpty(message = "name不能为空",groups = {MingruiOperation.Add.class})
    private String name;

    
    //因为numeric在数据库是个关键字  所以需要在实体中声明一下这个列
    @ApiModelProperty(name="numeric",example = "1")
    @Column(name = "`numeric`")
    private Boolean numeric;

    @ApiModelProperty(name="unit")
    @NotEmpty(message = "unit不能为空",groups = {MingruiOperation.Add.class})
    private String unit;

    @ApiModelProperty(name="generic")
    private Boolean generic;

    @ApiModelProperty(name="searching")
    private Boolean searching;

    @ApiModelProperty(name = "segments")
    @NotEmpty(message = "segments不能为空",groups = {MingruiOperation.Add.class})
    private String segments;
}
```

### 4.后台定义接口

实体接收并返回List集合

```java
@ApiOperation("规格组参数查询")
@GetMapping("param/list/")
Result<List<SpecParamEntity>> getParamList(SpecParamDTO specParamDTO);
```

### 5.接口实现类

规格组参数表中 有规格组的外键,通过groupId 查询 规格组中各个规格组的参数

tb_spec_group规格组表

tb_spec_param规格组参数表

规格组表中的参数全部放在了规格组参数表中

前台传递的是规格组表的id  ,根据规格组表的id在规格组参数表中查询对应所存在的数据,并返回集合

```java
@Override
public Result<List<SpecParamEntity>> getParamList(SpecParamDTO specParamDTO) {

    SpecParamEntity specParamEntity = BaiduBeanUtil.copyProperties(specParamDTO, SpecParamEntity.class);
    Example example = new Example(SpecParamEntity.class);
    example.createCriteria().andEqualTo("groupId", specParamEntity.getGroupId());
    List<SpecParamEntity> paramEntities = specParamMapper.selectByExample(example);

    return this.setResultSuccess(paramEntities);
}
```

### 6.mapper接口

```java
public interface SpecParamMapper extends Mapper<SpecParamEntity> {
}
```

## 规格组参数新增

### 1.前台传递数据到后台

```
save(){
        const p = {};
        Object.assign(p, this.param);
        p.segments = p.segments.map(s => s.join("-")).join(",")
        this.$http({
            method: this.isEdit ? 'put' : 'post',
            url: '/param/save/',
            data: p,
        }).then(() => {
            // 关闭窗口
            this.show = false;
            this.$message.success("保存成功！");
            this.loadData();
          }).catch(() => {
              this.$message.error("保存失败！");
            });
    }
```

### 2.后台定义接口

```java
@ApiOperation("规格组参数新增")
@PostMapping("param/save/")
Result<JSONObject> getParamAdd(@RequestBody SpecParamDTO specParamDTO);
```

### 3.接口实现类

把DTO接受的数据传给实体进行操作数据库

```java
@Override
@Transactional
public Result<JSONObject> getParamAdd(SpecParamDTO specParamDTO) {
    specParamMapper.insertSelective(
            BaiduBeanUtil.copyProperties(specParamDTO,SpecParamEntity.class));
    return this.setResultSuccess();
}
```

## 规格组参数修改

### 1.后台定义接口

```java
@ApiOperation("规格组参数修改")
@PutMapping("param/save/")
Result<JSONObject> getParamEdit(@RequestBody SpecParamDTO specParamDTO);
```

### 2.接口实现类

把DTO接受的数据传给实体进行操作数据库

```java
@Override
@Transactional
public Result<JSONObject> getParamEdit(SpecParamDTO specParamDTO) {
    specParamMapper.updateByPrimaryKeySelective(
            BaiduBeanUtil.copyProperties(specParamDTO,SpecParamEntity.class));
    return this.setResultSuccess();
}
```

## 规格组参数删除

### 1.前台传递id到后台

```js
deleteParam(id) {
        this.$message.confirm("确认要删除该参数吗？")
        .then(() => {
            this.$http.delete("/param/delete/?id=" + id)
            .then(() => {
                this.$message.success("删除成功");
                this.loadData();
            })
            .catch(() => {
                this.$message.error("删除失败");
            })
        })
    },
```

### 2.后台定义接口

```java
@ApiOperation("规格组参数删除")
@DeleteMapping("param/delete")
Result<JSONObject> getParamDelete(Integer id);
```

### 3.接口实现类

直接拿id进行删除

```java
//规格组参数CRUD
@Override
@Transactional
public Result<JSONObject> getParamDelete(Integer id) {
    specParamMapper.deleteByPrimaryKey(id);
    return this.setResultSuccess();
}
```

# 规格组参数CRUD写完后,规格组删除出现问题

如果规格组中存在参数就不能被删除

判断 如果规格组中数据非0时就不能被删除

```java
Example example = new Example(SpecParamEntity.class);
example.createCriteria().andEqualTo("groupId",id);
List<SpecParamEntity> entities = specParamMapper.selectByExample(example);
if(entities.size() >= 1) return this.setResultError("规格组内存在参数，不能删");
```