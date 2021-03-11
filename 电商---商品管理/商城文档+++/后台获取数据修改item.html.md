获取后台数据。通过thymeleaf技术+vue技术  实现前台的展示

# 1.获取spu数据

前台传递spuId；

通过spuid查询查询spu数据集合，因为一个id对应一条数据，所以只要一条数据。

将spu数据放如map中

# 2.获取spudetail数据

根据spuid直接查询spuDetail数据,并放入map集合

# 3.获取sku数据

根据spuid直接查询sku数据，并放入map集合

# 4.获取分类数据

分类数据是spu中的 cid1，cid2，cid3

根据id集合查询分类信息

String 类型的ids

通过spuData的数据，获取cid1，cid2，cid3

放入Arrays.asList中，并通过+“”将数据转换为String类型的数组

在通过String.join()方法将String类型集合的数据进行拼接为“1，2，3“

```java
通过String类型的ids进行批量查询
@Override
    public Result<List<CategoryEntity>> getCateByIds(String cateIds) {

        List<Integer> collect = Arrays.asList(cateIds.split(","))
                .stream().map(idStr -> Integer.valueOf(idStr)).collect(Collectors.toList());
        List<CategoryEntity> list = categoryMapper.selectByIdList(collect);
        return this.setResultSuccess(list);
    }
```

查询完之后并将数据放入map集合中

# 5.查询品牌数据

通过spuData中的spuid查询品牌数据

查询完后并放入map集合

# 6.//规格组通用数据

1.spu的cid3的值 就是规格组中cid的字段 将cid3 赋值给group中的cid字段1

2.通过feign调用规格组的查询方法查询到Result<List<SpecificationEntity>>类型的集合

3.判断查出来是否有数据

4.getData获取集合的数据，并通过蓝不大表达式来遍历数据

5.SpecificationDTO specificationDTO = BaiduBeanUtil.copyProperties(specGroup, SpecificationDTO.class);

用dto来操作

将group中的cid赋值给规格组参数的cid

并将规格参数中的是否是通用数据字段赋值，只取通用字段



6。查询规格参数数据

查询的数据放入规格组dto定义的参数字段中

并放入map集合中



# 7.规格组特有数据

1.通过spucid3的数据查询规格组特有数据,并调用查询方法

2.查询完后获取数据

3.遍历查询后数据并放入定义的map

4.将遍历后的map放入返回数据的map



```java
    SpecParamDTO specParamDTO = new SpecParamDTO();
        specParamDTO.setCid(spuData.getCid3());
        specParamDTO.setGeneric(false);
        Result<List<SpecParamEntity>> paramResult = specificationFeign.getParamList(specParamDTO);
        if(paramResult.isSuccess()){
            List<SpecParamEntity> paramData = paramResult.getData();
            Map<Integer,String> specParamMap  = new HashMap<>();
            paramData.stream().forEach(param-> specParamMap.put(param.getId(),param.getName()));
            goodsInfoMap.put("specParamMap",specParamMap);
   	 }
```



```java

@Service
public class PageServiceImpl implements PageService {
    @Override
    public Map<String, Object> getGoodsInfo(Integer spuId) {
        Map<String, Object> goodsInfoMap = new HashMap<>();
        //获取spu数据
        SpuDTO spuDTO = new SpuDTO();
        spuDTO.setId(spuId);
        Result<List<SpuDTO>> spuResultData = goodsFeign.getSpuInfo(spuDTO);
        SpuDTO spuData = null;
        if(spuResultData.isSuccess()){
            spuData = spuResultData.getData().get(0);
            goodsInfoMap.put("spuInfo", spuData);
        }
        //spudetail数据
        Result<SpuDetailEntity> spuDetailInfo = goodsFeign.getSpuDetailBySpuId(spuId);
        if (spuDetailInfo.isSuccess()) {
            goodsInfoMap.put("spuDetailInfo",spuDetailInfo.getData());
        }
        //sku数据
        Result<List<SkuDTO>> skusInfo = goodsFeign.getSkusBySpuId(spuId);
        if (skusInfo.isSuccess()){
            goodsInfoMap.put("skuInfo",skusInfo.getData());
        }
        //分类数据

        Result<List<CategoryEntity>> cateInfo = categoryFeign.getCateByIds(
                String.join(
                        ","
                        , Arrays.asList(
                                spuData.getCid1() + "",
                                spuData.getCid2() + "",
                                spuData.getCid3() + "")
                )
        );
        if(cateInfo.isSuccess()){
            goodsInfoMap.put("category",cateInfo.getData());
        }
        //品牌数据
        BrandDTO brandDTO = new BrandDTO();
        brandDTO.setId(spuData.getBrandId());
        Result<PageInfo<BrandEntity>> brandInfo = brandFeign.getBrandList(brandDTO);
        if (brandInfo.isSuccess()){
            goodsInfoMap.put("brandInfo",brandInfo.getData().getList().get(0));
        }

        //规格组通用数据
        SpecificationDTO specGroupDTO = new SpecificationDTO();
        specGroupDTO.setCid(spuData.getCid3());
        Result<List<SpecificationEntity>> specGroupResult  = specificationFeign.getSpecicationList(specGroupDTO);
        if(specGroupResult.isSuccess()){
            List<SpecificationEntity> specGroupList = specGroupResult.getData();
            List<SpecificationDTO> SpecGroupAndParamList = specGroupList.stream().map(specGroup -> {
                SpecificationDTO specificationDTO = BaiduBeanUtil.copyProperties(specGroup, SpecificationDTO.class);

                SpecParamDTO specParamDTO = new SpecParamDTO();
                specParamDTO.setCid(specificationDTO.getCid());
                specParamDTO.setGeneric(true);
                Result<List<SpecParamEntity>> paramList = specificationFeign.getParamList(specParamDTO);

                if (paramList.isSuccess()) {
                    specificationDTO.setSpecParamList(paramList.getData());
                }
                return specificationDTO;
            }).collect(Collectors.toList());
            goodsInfoMap.put("specGroupAndParam",SpecGroupAndParamList);
        }
        //规格组特有数据
        SpecParamDTO specParamDTO = new SpecParamDTO();
        specParamDTO.setCid(spuData.getCid3());
        specParamDTO.setGeneric(false);
        Result<List<SpecParamEntity>> paramResult = specificationFeign.getParamList(specParamDTO);
        if(paramResult.isSuccess()){
            List<SpecParamEntity> paramData = paramResult.getData();
            Map<Integer,String> specParamMap  = new HashMap<>();
            paramData.stream().forEach(param-> specParamMap.put(param.getId(),param.getName()));
            goodsInfoMap.put("specParamMap",specParamMap);

        }

        return goodsInfoMap;
    }
}
```