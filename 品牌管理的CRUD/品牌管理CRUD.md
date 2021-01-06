

# 品牌管理CRUD

## 品牌查询

### 后台定义接口

mingrui-shop-service-api-xxx定义接口

```java
@GetMapping(value="brand/list")
@ApiOperation("品牌查询")
Result<PageInfo<BrandEntity>> getBrandList(BrandDTO brandDTO);
```



数据库表有4个字段，而实现的功能有需要 分页/排序 

定义一个brandDTO（相当于实体类）接受前台传递的参数，

### 定义前台传递参数的实体 

定义在mingrui-shop-service-api-xxx中

```java
@Data
@ApiModel(value = "品牌DTO")
public class BrandDTO extends BaseDTO {

    @ApiModelProperty(value = "品牌主键",example = "1")
    @NotNull(message = "主键不能为空", groups = {MingruiOperation.Update.class})
    private Integer id;
    @ApiModelProperty(value = "品牌名称")
    @NotEmpty(message = "名牌名称不能为空", groups =
            {MingruiOperation.Add.class,MingruiOperation.Update.class})
    private String name;
    @ApiModelProperty(value = "品牌图片")
    private String image;

    @ApiModelProperty(value = "品牌首字母")
    private Character letter;

    @ApiModelProperty(value = "品牌类别")
    private String categories;

}
```

### 定义前台传递参数的实体的继承类

因为排序分页有复用性 所以提取出来 

mingrui-shop-common-core  项目下 com.baidu.shop.base包下创建此类

```java
@Data
@ApiModel(value = "BaseDTO用于数据传输,其他dto需要继承此类")
public class BaseDTO {

    @ApiModelProperty(value="当前页" ,example = "1")
    private Integer page;

    @ApiModelProperty(value = "每页条数",example = "5")
    private Integer rows;

    @ApiModelProperty(value = "排序字段")
    private String  sort;

    @ApiModelProperty(value = "是否升序")
    private String  order;


    public String getOrderBy(){
        return sort+" "+(Boolean.valueOf(order) ? "DESC":"ASC");
    }

}
```

在mingrui-shop-service-xxx中写实现类

```java
@Override
public Result<PageInfo<BrandEntity>> getBrandList(BrandDTO brandDTO) {
    //分页
    PageHelper.startPage(brandDTO.getPage(),brandDTO.getRows());
    //排序
    PageHelper.orderBy(brandDTO.getOrderBy());
    //将实体类的属性复制给BrandDTO，用BrandDTO来操作
    BrandEntity brandEntity = BaiduBeanUtil.copyProperties(brandDTO,BrandEntity.class);
	//在搜索框做条件查询  模糊匹配
    Example example = new Example(BrandEntity.class);
    example.createCriteria().andLike("name","%"+brandEntity.getName()+"%");
	
    List<BrandEntity> brandEntities = brandMapper.selectByExample(example);
    PageInfo<BrandEntity> PageInfo = new PageInfo<>(brandEntities);

    return this.setResultSuccess(PageInfo);
}
```

# 品牌新增

## 前台传递数据

```javascript
submitForm() {
      if (!this.$refs.form.validate()) {
        return;
      }

      let formData = this.brand;
      if(!this.isEdit)formData.id = null;
      let categoryIdArr = this.brand.categories.map((category) => category.id);
      formData.categories = categoryIdArr.join();

      this.$http({
        url:"/brand/save",
        method:this.isEdit ?"put":"post",
        data:formData
      }).then((resp) => {
          if (resp.data.code != 200) {
            this.$message.error("新增失败");
            return;
          }
          //刷新页面
          this.cancel();
        })
        .catch((err) => console.log(err));
    },
```



## 后台定义接口接受前台传递的参数

​	

```java
@PostMapping(value = "brand/save")
@ApiOperation("品牌新增")
Result<JSONObject>saveBrandList(@RequestBody BrandDTO brandDTO);
```

#### 1.第一步

##### 		把brandEntity实体类的属性都赋值给，DTO数据传输的实体 

#### 2.第二步

##### 		根据name的属性 判断name第一个大写字母 并插入到 letter字段

#### 第三步，

##### 		新增数据

#### 第四步，

##### 		获取到品牌分类的分类数据

#### 第五步

##### 		新增返回主键

#### 第六步

##### 		在mapper接口上继承

​				InsertListMapper<CategoryBrandEntity>接口

​		泛型是中间表数据

需要在实体id上添加注解

```
@Data
@Table(name = "tb_brand")
public class BrandEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Integer id;

    private String name;

    private String image;

    private Character letter;

}
```

```java
BrandEntity brandEntity = BaiduBeanUtil.copyProperties(brandDTO, BrandEntity.class);

//在pom文件引入pinyinUtil依赖
//根据name的值根据pinyinUril工具类来判断name属性值 开头第一个字母 包括汉字的第一个字母
//把name字符串转换为char类型的数组，要第一个下标为0的 ，因为工具类是string类型的，所以要根据String.valueOf转换为String类型的包装类
//因为letter又是char类型的数据 所以要在后面添加一个tocharArray（）方法转换为char类型的
brandEntity.setLetter(PinyinUtil.getUpperCase(String.valueOf(brandEntity.getName().toCharArray()[0]),false).toCharArray()[0]);

brandMapper.insertSelective(brandEntity);   

        String categories = brandDTO.getCategories();
        if(StringUtils.isEmpty(categories)) return this.setResultError("分类集合不能为空");
//        List<CategoryBrandEntity> list = new ArrayList<>();

        if(categories.contains(",")){

            this.addBatchBrandCategory(brandDTO.getCategories(),brandEntity.getId());
        }
        return this.setResultSuccess();
```

#### 批量新增的方法

```
private void addBatchBrandCategory(String categories,Integer id){
        if(StringUtils.isEmpty(categories)) throw  new RuntimeException();

        if(categories.contains(",")){
            categoryBrandMapper.insertList(Arrays.asList(categories.split(","))
                    .stream()
                    .map(categoryIdStr -> {
                        return new CategoryBrandEntity(Integer.valueOf(categoryIdStr),id);})
                    .collect(Collectors.toList()));
        }else{
            CategoryBrandEntity categoryBrandEntity = new CategoryBrandEntity();
            categoryBrandEntity.setBrandId(id);
            categoryBrandEntity.setCategoryId(Integer.valueOf(categories));
            categoryBrandMapper.insertSelective(categoryBrandEntity);
        }

        //        List<CategoryBrandEntity> list = new ArrayList<>();

//        if(categories.contains(",")){

//        String[] s = categories.split(",");
//            for(String s : split){
//                CategoryBrandEntity categoryBrandEntity = new CategoryBrandEntity();
//                categoryBrandEntity.setBrandId(brandEntity.getId());
//                categoryBrandEntity.setCategoryId(Integer.valueOf(s));
//                list.add(categoryBrandEntity);
//            }
//            categoryBrandMapper.insertList(list);
//        }else{
//            CategoryBrandEntity categoryBrandEntity = new CategoryBrandEntity();
//            categoryBrandEntity.setBrandId(brandEntity.getId());
//            categoryBrandEntity.setCategoryId(Integer.valueOf(categories));
//            categoryBrandMapper.insert(categoryBrandEntity);

        //批量新增2222
//            categoryBrandMapper.insertList(Arrays.asList(categories.split(","))
//                    .stream()
//                    .map(categoryIdStr -> {
//                        return new CategoryBrandEntity(Integer.valueOf(categoryIdStr),
//                                brandEntity.getId());})
//                    .collect(Collectors.toList()));
//          }
    }
```

品牌管理  新增和修改时都需要 去查询商品分类

![image-20210105221706099](C:\Users\LiuHaoJie\AppData\Roaming\Typora\typora-user-images\image-20210105221706099.png)

在CategoryServiceImpl实现类中查询

```java
@Override
public Result<List<CategoryEntity>> getCategoryByBrandId(Integer brandId) {
    List<CategoryEntity> list = categoryMapper.getCategoryByBrandId(brandId);
    return this.setResultSuccess(list);
}
```

mapper接口中写这个查询方法

自己写sql语句，因为框架没那么智能

通过查询中间表来找到品牌分类

```java
@Select(value = "select id,name from tb_category where id in (SELECT category_id from tb_category_brand where brand_id = #{brandId})")
List<CategoryEntity> getCategoryByBrandId(Integer brandId);
```

# 品牌修改

#### 1.第一步

##### 		把brandEntity实体类的属性都赋值给，DTO数据传输的实体 

#### 2.第二步

##### 		根据name的属性 判断name第一个大写字母 并插入到 letter字段

#### 3.第三步

##### 		修改添加的数据

#### 4.第四步

##### 		删除中间表数据

#### 5.第五步

##### 		批量新增中间表数据

### 修改接口

```
@PutMapping(value = "brand/save")
@ApiOperation("品牌修改")
Result<JSONObject> editBrandList(@RequestBody BrandDTO brandDTO);
```

### 修改实体

```java
 BrandEntity brandEntity = BaiduBeanUtil.copyProperties(brandDTO,BrandEntity.class);
            brandEntity.setLetter(PinyinUtil.getUpperCase(String.valueOf(brandEntity.getName().toCharArray()[0]),false).toCharArray()[0]);
            brandMapper.updateByPrimaryKeySelective(brandEntity);

//        Example example = new Example(CategoryBrandEntity.class);
//        example.createCriteria().andEqualTo("brandId",brandEntity.getId());
//        categoryBrandMapper.deleteByExample(example);
        this.deleteBrandCategory(brandEntity.getId());
        //批量新增中间表
        //先得到他的分类集合字符串
        this.addBatchBrandCategory(brandDTO.getCategories(),brandEntity.getId());

        return this.setResultSuccess();
```

### 实体调用的工具类

```java
//批量新增工具类
    private void addBatchBrandCategory(String categories,Integer id){
        if(StringUtils.isEmpty(categories)) throw  new RuntimeException();

        if(categories.contains(",")){
            categoryBrandMapper.insertList(Arrays.asList(categories.split(","))
                    .stream()
                    .map(categoryIdStr -> {
                        return new CategoryBrandEntity(Integer.valueOf(categoryIdStr),id);})
                    .collect(Collectors.toList()));
        }else{
            CategoryBrandEntity categoryBrandEntity = new CategoryBrandEntity();
            categoryBrandEntity.setBrandId(id);
            categoryBrandEntity.setCategoryId(Integer.valueOf(categories));
            categoryBrandMapper.insertSelective(categoryBrandEntity);
        }

        //        List<CategoryBrandEntity> list = new ArrayList<>();

//        if(categories.contains(",")){

//        String[] s = categories.split(",");
//            for(String s : split){
//                CategoryBrandEntity categoryBrandEntity = new CategoryBrandEntity();
//                categoryBrandEntity.setBrandId(brandEntity.getId());
//                categoryBrandEntity.setCategoryId(Integer.valueOf(s));
//                list.add(categoryBrandEntity);
//            }
//            categoryBrandMapper.insertList(list);
//        }else{
//            CategoryBrandEntity categoryBrandEntity = new CategoryBrandEntity();
//            categoryBrandEntity.setBrandId(brandEntity.getId());
//            categoryBrandEntity.setCategoryId(Integer.valueOf(categories));
//            categoryBrandMapper.insert(categoryBrandEntity);

        //批量新增2222
//            categoryBrandMapper.insertList(Arrays.asList(categories.split(","))
//                    .stream()
//                    .map(categoryIdStr -> {
//                        return new CategoryBrandEntity(Integer.valueOf(categoryIdStr),
//                                brandEntity.getId());})
//                    .collect(Collectors.toList()));
//          }
    }

    private void deleteBrandCategory(Integer id){
        Example example = new Example(CategoryBrandEntity.class);
        example.createCriteria().andEqualTo("brandId",id);
        categoryBrandMapper.deleteByExample(example);
    }
```



## 品牌删除

### 1.接口

```java
@DeleteMapping(value="brand/delete")
@ApiOperation("品牌删除")
Result<JSONObject> deleteBrandCategoryById(Integer id);
```

### 2.接口实现类

```
@Override
public Result<JSONObject> deleteBrandCategoryById(Integer id) {

    //品牌删除
    brandMapper.deleteByPrimaryKey(id);
    //删除品牌商品分类列表
    this.deleteBrandCategory(id);

    return this.setResultSuccess();
}
```

#### 1.根据前台的传递的id进行删除操作

#### 2.删除品牌分类中间表数据

