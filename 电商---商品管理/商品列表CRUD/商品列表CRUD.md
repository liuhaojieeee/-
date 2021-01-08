# 商品列表CRUD

## 查询

### 后台定义接口

```java
@ApiOperation(value = "查询spu信息")
@GetMapping(value = "goods/getSpuInfo")
Result<List<SpuDTO>> getSpuInfo(SpuDTO spuDTO);
```

### 实体类

```java
@Id
@GeneratedValue(strategy = GenerationType.IDENTITY)
private Integer id;
private String title;
private String subTitle;
private Integer cid1;
private Integer cid2;
private Integer cid3;
private Integer brandId;
private Integer saleable;
private Integer valid;
private Date createTime;
private Date lastUpdateTime;
```

### spu数据传输DTO

```java
@Id
@ApiModelProperty(value = "主键",example = "1")
@NotNull(message = "主键不能为空",groups = {MingruiOperation.Update.class})//参数校验
private Integer id;

@ApiModelProperty(value = "标题")
@NotEmpty(message = "标题不能为空",groups = {MingruiOperation.Add.class})
private String title;

@ApiModelProperty(value = "子标题")
@NotEmpty(message = "子标题不能为空",groups = {MingruiOperation.Add.class})
private String subTitle;

@ApiModelProperty(value = "一级类目",example = "1")
@NotNull(message = "一级类目不能为空",groups = {MingruiOperation.Add.class})//参数校验
private Integer cid1;

@ApiModelProperty(value = "二级类目",example = "1")
@NotNull(message = "二级类目不能为空",groups = {MingruiOperation.Add.class})//参数校验
private Integer cid2;

@ApiModelProperty(value = "三级类目",example = "1")
@NotNull(message = "三级类目不能为空",groups = {MingruiOperation.Add.class})//参数校验
private Integer cid3;

@ApiModelProperty(value = "商品所属品牌id",example = "1")
@NotNull(message = "商品所属品牌id不能为空",groups = {MingruiOperation.Add.class})//参数校验
private Integer brandId;

@ApiModelProperty(value = "是否上架",example = "1")
@NotNull(message = "是否上架不能为空",groups = {MingruiOperation.Add.class})//参数校验
private Integer saleable;

@ApiModelProperty(value = "是否有效",example = "1")
@NotNull(message = "是否有效不能为空",groups = {MingruiOperation.Add.class})//参数校验
private Integer valid;

@ApiModelProperty(value = "添加时间")
@NotNull(message = "添加时间不能为空",groups = {MingruiOperation.Add.class})//参数校验
private Date createTime;

@ApiModelProperty(value = "最后修改时间")
@NotNull(message = "最后修改时间不能为空",groups = {MingruiOperation.Add.class})//参数校验
private Date lastUpdateTime;

private String brandName;
private String categoryName;
```

### 实现类

.做分页和排序功能

.做页面的是否上下架，模糊查询

```java
package com.baidu.shop.service.impl;

import com.alibaba.fastjson.JSONObject;
import com.baidu.shop.base.BaseApiService;
import com.baidu.shop.base.Result;
import com.baidu.shop.dao.SpuDTO;
import com.baidu.shop.entity.*;
import com.baidu.shop.mapper.*;
import com.baidu.shop.service.GoodsService;
import com.baidu.shop.status.HTTPStatus;
import com.baidu.shop.utils.BaiduBeanUtil;
import com.baidu.shop.utils.ObjectUtil;
import com.github.pagehelper.PageHelper;
import com.github.pagehelper.PageInfo;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.web.bind.annotation.RestController;
import tk.mybatis.mapper.entity.Example;
import tk.mybatis.mapper.util.StringUtil;

import javax.annotation.Resource;
import java.util.Arrays;
import java.util.Date;
import java.util.List;
import java.util.stream.Collectors;
import java.util.stream.Stream;

/**
 * @ClassName GoodsServiceImpl
 * @Description: TODO
 * @Author liuhaojie
 * @Date 2021/1/5
 * @Version V1.0
 **/
@RestController
public class GoodsServiceImpl extends BaseApiService implements GoodsService {
    @Resource
    private BrandMapper brandMapper;

    @Resource
    private CategoryMapper categoryMapper;

    @Resource
    private GoodsMapper goodsMapper;

    @Override
    public Result<List<SpuDTO>> getSpuInfo(SpuDTO spuDTO) {
//做分页查询
        if(ObjectUtil.isNotNull(spuDTO.getPage()) && ObjectUtil.isNotNull(spuDTO.getRows()))
            PageHelper.startPage(spuDTO.getPage(),spuDTO.getRows());
//做排序
        if(!StringUtil.isEmpty(spuDTO.getSort()) && !StringUtil.isEmpty(spuDTO.getOrder()))
            PageHelper.orderBy(spuDTO.getOrderBy());

        Example example = new Example(SpuEntity.class);
        Example.Criteria criteria = example.createCriteria();
//做页面的是否上下架
        if(ObjectUtil.isNotNull(spuDTO.getSaleable()) && spuDTO.getSaleable() < 2)
            criteria.andEqualTo("saleable",spuDTO.getSaleable());
//做页面的模糊查询title标题
        if(!StringUtil.isEmpty(spuDTO.getTitle()))
            criteria.andLike("title","%"+spuDTO.getTitle()+"%");


        List<SpuEntity> entities = goodsMapper.selectByExample(example);
//第一种方式但是分页出不来
//        List<SpuDTO> collect = entities.stream().map(spuEntity -> {
//            SpuDTO spuDTO1 = BaiduBeanUtil.copyProperties(spuEntity, SpuDTO.class);
//
//            CategoryEntity categoryEntity1 = categoryMapper.selectByPrimaryKey(spuEntity.getCid1());
//            CategoryEntity categoryEntity2 = categoryMapper.selectByPrimaryKey(spuEntity.getCid2());
//            CategoryEntity categoryEntity3 = categoryMapper.selectByPrimaryKey(spuEntity.getCid3());
//            spuDTO1.setCategoryName(categoryEntity1.getName() + "/" + categoryEntity2.getName() + "/" + categoryEntity3.getName());
//
//            BrandEntity brandEntity = brandMapper.selectByPrimaryKey(spuEntity.getBrandId());
//            spuDTO1.setBrandName(brandEntity.getName());
//            return spuDTO1;
//        }).collect(Collectors.toList());

        //第二种方式
        List<SpuDTO> collect = entities.stream().map(spuEntity -> {
            SpuDTO spuDTO1 = BaiduBeanUtil.copyProperties(spuEntity, SpuDTO.class);
            //通过id查询 分类管理的数据
            List<CategoryEntity> categoryEntities = categoryMapper.selectByIdList(
                    Arrays.asList(spuEntity.getCid1(), spuEntity.getCid2(), spuEntity.getCid3()));

            //根据查询的数据，来获取到分类名称
            String collect1 = categoryEntities.stream().map(categoryEntity ->
                categoryEntity.getName()).collect(Collectors.joining("/"));
            spuDTO1.setCategoryName(collect1);

            //根据brandId查询到品牌名称 赋值给spu中的BrandName
            BrandEntity brandEntity = brandMapper.selectByPrimaryKey(spuEntity.getBrandId());
            spuDTO1.setBrandName(brandEntity.getName());
            return spuDTO1;
        }).collect(Collectors.toList());


        PageInfo<SpuEntity> pageInfo = new PageInfo<>(entities);
        return this.setResult(HTTPStatus.OK,pageInfo.getTotal()+"",collect);
    }
}
```

## 新增

后台定义接口

```java
@ApiOperation(value = "新增商品")
@PostMapping(value = "goods/save")
Result<JSONObject> saveGoods(@RequestBody SpuDTO spuDTO);
```

实现类

需要新增 四张表  

spu 

spu_detail

sku

stock

spu 和 spu_detail 表是一对一的关系  完全可以分开，但是detail表 数据过大 影响 spu表的查询效率所以提出来的

### 1.先新增spu表 

spu表中   是否上架 

​				是否启用

​				创建时间

​				最后修改时间

的值都为null  在页面上不能新增（可以先提交一下查询数据，测试哪些字段值为null）

新增表的时候给这些字段一个默认值，

### 2.spu查询完后，新增返回主键  detail表根据返回的主键查询数据

### 3.把返回的id数据返回给spu_detail 表的spuId字段进行查询 

### 4.给spuEntity中添加 detail表的集合  和  sku表的集合

### 5.lambda表达式遍历 spuDTO中sku的集合数据

### 6.新增返回主键在id上添加@GeneratedValue(strategy = GenerationType.IDENTITY)

### 7.根据返回的主键再次新增stock表

### 8.把sku的数据和spuDetail表中的数据都引入到spu表中

```java
@ApiModelProperty(value = "大字段数据")
    private SpuDetailDTO spuDetail;
@ApiModelProperty(value = "sku属性数据集合")
	private List<SkuDTO> skus;
```
```
package com.baidu.shop.dao;

import com.baidu.shop.base.BaseDTO;
import com.baidu.shop.validate.group.MingruiOperation;
import io.swagger.annotations.ApiModel;
import io.swagger.annotations.ApiModelProperty;
import lombok.Data;

import javax.persistence.Id;
import javax.validation.constraints.NotEmpty;
import javax.validation.constraints.NotNull;
import java.util.Date;
import java.util.List;

/**
 * @ClassName SpuDTO
 * @Description: TODO
 * @Author liuhaojie
 * @Date 2021/1/5
 * @Version V1.0
 **/
@ApiModel(value = "SPU数据传输DTO")
@Data
public class SpuDTO extends BaseDTO {

    @Id
    @ApiModelProperty(value = "主键",example = "1")
    @NotNull(message = "主键不能为空",groups = {MingruiOperation.Update.class})//参数校验
    private Integer id;

    @ApiModelProperty(value = "标题")
    @NotEmpty(message = "标题不能为空",groups = {MingruiOperation.Add.class})
    private String title;

    @ApiModelProperty(value = "子标题")
    @NotEmpty(message = "子标题不能为空",groups = {MingruiOperation.Add.class})
    private String subTitle;

    @ApiModelProperty(value = "一级类目",example = "1")
    @NotNull(message = "一级类目不能为空",groups = {MingruiOperation.Add.class})//参数校验
    private Integer cid1;

    @ApiModelProperty(value = "二级类目",example = "1")
    @NotNull(message = "二级类目不能为空",groups = {MingruiOperation.Add.class})//参数校验
    private Integer cid2;

    @ApiModelProperty(value = "三级类目",example = "1")
    @NotNull(message = "三级类目不能为空",groups = {MingruiOperation.Add.class})//参数校验
    private Integer cid3;

    @ApiModelProperty(value = "商品所属品牌id",example = "1")
    @NotNull(message = "商品所属品牌id不能为空",groups = {MingruiOperation.Add.class})//参数校验
    private Integer brandId;

    @ApiModelProperty(value = "是否上架",example = "1")
    @NotNull(message = "是否上架不能为空",groups = {MingruiOperation.Add.class})//参数校验
    private Integer saleable;

    @ApiModelProperty(value = "是否有效",example = "1")
    @NotNull(message = "是否有效不能为空",groups = {MingruiOperation.Add.class})//参数校验
    private Integer valid;

    @ApiModelProperty(value = "添加时间")
    @NotNull(message = "添加时间不能为空",groups = {MingruiOperation.Add.class})//参数校验
    private Date createTime;

    @ApiModelProperty(value = "最后修改时间")
    @NotNull(message = "最后修改时间不能为空",groups = {MingruiOperation.Add.class})//参数校验
    private Date lastUpdateTime;

    private String brandName;
    private String categoryName;

    @ApiModelProperty(value = "大字段数据")
    private SpuDetailDTO spuDetail;

    @ApiModelProperty(value = "sku属性数据集合")
    private List<SkuDTO> skus;

}
```

```java
@Override
@Transactional
public Result<JSONObject> saveGoods(SpuDTO spuDTO) {

    final Date date = new Date();

    //新增spu表
    SpuEntity spuEntity = BaiduBeanUtil.copyProperties(spuDTO, SpuEntity.class);
        spuEntity.setSaleable(1);
        spuEntity.setValid(1);
        spuEntity.setCreateTime(date);
        spuEntity.setLastUpdateTime(date);
    goodsMapper.insertSelective(spuEntity);

    //新增spuDetail表
    SpuDetailEntity spuDetailEntity = BaiduBeanUtil.copyProperties(spuDTO.getSpuDetail(), SpuDetailEntity.class);
        spuDetailEntity.setSpuId(spuEntity.getId());
    spuDetailMapper.insertSelective(spuDetailEntity);

    //新增sku表
    spuDTO.getSkus().stream().forEach(skuDTO -> {
        SkuEntity skuEntity = BaiduBeanUtil.copyProperties(skuDTO, SkuEntity.class);
            skuEntity.setSpuId(spuEntity.getId());
            skuEntity.setCreateTime(date);
            skuEntity.setLastUpdateTime(date);
        skuMapper.insertSelective(skuEntity);

        //新增stock表
        StockEntity stockEntity = new StockEntity();
            stockEntity.setSkuId(skuEntity.getId());
            stockEntity.setStock(skuDTO.getStock());
        stockMapper.insertSelective(stockEntity);
    });

    return this.setResultSuccess();
}
```

## 修改

### 1.数据回显

定义接口

```java
@ApiOperation(value = "根据spuId查询detail数据")
@GetMapping(value = "goods/getSpuDetailBySpuId")
Result<JSONObject> getSpuDetailBySpuId(Integer spuId);

@ApiOperation(value = "根据spuId查询detail数据")
@GetMapping(value = "goods/getSkusBySpuId")
Result<List<SkuDTO>> getSkusBySpuId(Integer spuId);
```

实现类

```java
@Resource
    private SpuDetailMapper spuDetailMapper;

@Resource
    private SkuMapper skuMapper;

@Override
public Result<List<SkuDTO>> getSkusBySpuId(Integer spuId) {
    //跟据spuId 查询sku表和stock表
    List<SkuEntity> list =  skuMapper.getSkusBySpuId(spuId);
    return this.setResultSuccess(list);
}

@Override
public Result<JSONObject> getSpuDetailBySpuId(Integer spuId) {
    SpuDetailEntity spuDetailEntity = spuDetailMapper.selectByPrimaryKey(spuId);
    return this.setResultSuccess(spuDetailEntity);
}
```

mapper接口

```java
public interface SkuMapper extends Mapper<SkuEntity>, InsertListMapper<SkuEntity>, DeleteByIdListMapper<SkuEntity,Long> {

    @Select(value = "SELECT k.*,t.stock from tb_sku k,tb_stock t where k.id = t.sku_id and k.spu_id = #{spuId}")
    List<SkuEntity> getSkusBySpuId(Integer spuId);
}


public interface SpuDetailMapper  extends Mapper<SpuDetailEntity> {
}

```

### 2.修改

前台传递SpuId ,后台用DTO接受

后台定义接口

```java
@ApiOperation(value = "修改商品")
@PutMapping(value = "goods/save")
Result<JSONObject> editGoods(@Validated(MingruiOperation.Update.class) @RequestBody SpuDTO spuDTO);
```

实现类

可以直接通过spuId来修改spu表 和detail表

如果直接修改sku表的话  就没skuId了 stock表就没发修改

所以要先查询出来sku表和stock表中所有的数据再进行新增

根据spuId查询出sku的数据，返回集合

遍历返回的集合，返回skuId

再进行新增



```java
@Override
public Result<JSONObject> editGoods(SpuDTO spuDTO) {
    //需要修改4张表 spu -> spuDetail -> sku -> stock
    //修改spu表
    SpuEntity spuEntity = BaiduBeanUtil.copyProperties(spuDTO, SpuEntity.class);
    goodsMapper.updateByPrimaryKeySelective(spuEntity);
    //修改detail表
    SpuDetailEntity spuDetailEntity = BaiduBeanUtil.copyProperties(spuDTO.getSpuDetail(), SpuDetailEntity.class);
    spuDetailMapper.updateByPrimaryKeySelective(spuDetailEntity);

    //修改sku表和stock表
    //先通过spuId来查询到skus表中的数据
    this.deleteSkuAndStock(spuDTO.getId());

    //新增sku表和stock表
    this.addSkuAndStock(spuDTO,new Date(),spuEntity);

    return this.setResultSuccess();
}
////////////////////////下面是提取的工具类
	private void deleteSkuAndStock(Integer spuId){
        //查询出来sku表中的数据
        Example example = new Example(SkuEntity.class);
        example.createCriteria().andEqualTo("spuId",spuId);
        List<SkuEntity> skuEntities = skuMapper.selectByExample(example);

        List<Long> collect = skuEntities.stream().map(skuEntity -> skuEntity.getId()).collect(Collectors.toList());

        skuMapper.deleteByIdList(collect);
        stockMapper.deleteByIdList(collect);
    }

    private void addSkuAndStock(SpuDTO spuDTO,Date date,SpuEntity spuEntity){
        // 新增sku表和stock表
        spuDTO.getSkus().stream().forEach(skuDTO -> {
            SkuEntity skuEntity = BaiduBeanUtil.copyProperties(skuDTO, SkuEntity.class);
            skuEntity.setSpuId(spuEntity.getId());
            skuEntity.setCreateTime(date);
            skuEntity.setLastUpdateTime(date);
            skuMapper.insertSelective(skuEntity);

            //新增stock表
            StockEntity stockEntity = new StockEntity();
            stockEntity.setSkuId(skuEntity.getId());
            stockEntity.setStock(skuDTO.getStock());
            stockMapper.insertSelective(stockEntity);
        });
    }
```

## 删除

### 1.根据spuId来删除四张表

### 后台定义接口接受传递的id

```java
@ApiOperation(value = "删除数据")
@DeleteMapping(value = "/goods/delete")
Result<List<SkuDTO>> deleteGoods(Integer spuId);
```

实现类

### 2.根据spuid来删除4个表

```
@Override
@Transactional
public Result<List<SkuDTO>> deleteGoods(Integer spuId) {
    //删除4表
    goodsMapper.deleteByPrimaryKey(spuId);
    //删除spuDetail表
    spuDetailMapper.deleteByPrimaryKey(spuId);

    this.deleteSkuAndStock(spuId);

    return this.setResultSuccess();
}
```