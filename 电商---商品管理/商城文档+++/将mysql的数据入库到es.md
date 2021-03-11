# 1.将mysql数据填充到es

### 1.声明feign接口调用服务 并继承类中的方法

```java
@FeignClient(value = "xxx-server",contextId = "GoodsFeign")
public interface GoodsFeign extends GoodsService {
}
```

### 2.通过goods来查询spu表中的数据，

因为getspuInFo方法内有分页的非空判断，加上page和rows

```java
SpuDTO spuDTO = new SpuDTO();
spuDTO.setPage(1);
spuDTO.setRows(5);
Result<List<SpuDTO>> spuInfo = goodsFeign.getSpuInfo(spuDTO);
```

### 3.判断spu传过来的数据是否成功

，成功后遍历传过来的spu集合

并对GoodsDoc内的数据一一赋值

```java
if(spuInfo.getCode() == 200){
List<SpuDTO> collect = spuInfo.getData().stream().map(spu -> {

                goodsDoc.setId(spu.getId().longValue());
                goodsDoc.setBrandId(spu.getBrandId().longValue());
                goodsDoc.setBrandName(spu.getBrandName());
                goodsDoc.setCategoryName(spu.getCategoryName());
                goodsDoc.setCid1(spu.getCid1().longValue());
                goodsDoc.setCid2(spu.getCid2().longValue());
                goodsDoc.setCid3(spu.getCid3().longValue());
                goodsDoc.setCreateTime(spu.getCreateTime());
                goodsDoc.setTitle(spu.getTitle());
                goodsDoc.setSubTitle(spu.getSubTitle());
}
```

### 4.通过spuId来查询skus集合的数据,

判断传递来的数据是否成功，并遍历skus的数据,建立一个map集合 将sku里的数据放入map中 ，因为GoodsDoc里skus用的是String

类型的 ，所以我们要创建一个JSON数据转String对象，map就是一个JSON数据

Price价格是List<Long> 定义一个list将skus中的价格加入goodsdoc中。

```java
Result<List<SkuDTO>> skus = goodsFeign.getSkusBySpuId(spu.getId());
if (skus.getCode() == 200) {

    List<Long> priceList = new ArrayList<>();
    List<Map<String, Object>> skuListMap = skus.getData().stream().map(sku -> {
        Map<String, Object> map = new HashMap<>();
        map.put("id", sku.getId());
        map.put("title", sku.getTitle());
        map.put("images", sku.getImages());
        map.put("price", sku.getPrice());
        priceList.add(sku.getPrice().longValue());
        return map;
    }).collect(Collectors.toList());

    goodsDoc.setPrice(priceList);
    goodsDoc.setSkus(JSONUtil.toJsonString(skuListMap));
}
```

### 5.规格参数

//规格参数填充
//获取规格参数
先获取规格参数，通过cid3来查询规格参数 因为规格参数就是在分类的最后一层。

查询规格参数

查询spudetail中的参数数据

并将spudetail中的数据分为   通用属性和特有属性

将两种属性 ，分别转为map集合



再将两种集合放入规格参数中

通过判断generic字段判断 是否为通用字段来填充到map中

处理带有范围的属性值

先判断

 `searching` tinyint(1) NOT NULL COMMENT '是否用于搜索过滤，true或false',

`segments` varchar(1000) DEFAULT '' COMMENT '数值类型参数，如果需要搜索，则添加分段间隔值，如CPU频率间隔：0.5-1.0',

这两个字段，并填充入map中



最后将specmap中的数据放入     goodsDoc中的Specs字段中

```java
//规格参数填充
//获取规格参数
先获取规格参数组，通过cid3来查询规格参数 因为规格参数就是在分类的最后一层
SpecParamDTO specParamDTO = new SpecParamDTO();
                specParamDTO.setCid(spu.getCid3());
                specParamDTO.setSearching(true);//直查询有查询条件的规格参数
                Result<List<SpecParamEntity>> specParamList = specificationFeign.getParamList(specParamDTO);
                
将规格参
规格参数有通用的规格参数和普通的规格参数。

 if (specParamList.getCode() == 200) {
                    List<SpecParamEntity> ParamList = specParamList.getData();
                    //获取detail数据
                    Result<SpuDetailEntity> spuDetailList = goodsFeign.getSpuDetailBySpuId(spu.getId());
                    if (spuDetailList.getCode() == 200) {
                        SpuDetailEntity spuDetailData = spuDetailList.getData();
                        ////将json字符串转换成map集合
                        Map<String, String> genericSpec = JSONUtil.toMapValueString(spuDetailData.getGenericSpec());
                        Map<String, List<String>> specialSpec = JSONUtil.toMapValueStrList(spuDetailData.getSpecialSpec());




                        //需要查询两张表的数据 spec_param(规格参数名) spu_detail(规格参数值) --> 规格参数名 : 规格参数值
                        HashMap<String, Object> SpecMap = new HashMap<String, Object>();
                        ParamList.stream().forEach(specParam -> {
                            if (specParam.getGeneric()) {
                                if (specParam.getSearching() && !StringUtils.isEmpty(specParam.getSegments())) {
                                    SpecMap.put(specParam.getName(),
                                            chooseSegment(genericSpec.get(specParam.getId() + ""), specParam.getSegments(), specParam.getUnit()));
                                } else {
                                    SpecMap.put(specParam.getName(), genericSpec.get(specParam.getId() + ""));
                                }
                            } else {
                                SpecMap.put(specParam.getName(), specialSpec.get(specParam.getId() + ""));
                            }
                        });

                        goodsDoc.setSpecs(SpecMap);
```

### 全部数据

```java
package com.baidu.shop.service.impl;

import com.alibaba.fastjson.JSONObject;
import com.baidu.shop.base.BaseApiService;
import com.baidu.shop.base.Result;
import com.baidu.shop.dao.SkuDTO;
import com.baidu.shop.dao.SpecParamDTO;
import com.baidu.shop.dao.SpecificationDTO;
import com.baidu.shop.dao.SpuDTO;
import com.baidu.shop.document.GoodsDoc;
import com.baidu.shop.entity.SpecParamEntity;
import com.baidu.shop.entity.SpuDetailEntity;
import com.baidu.shop.feign.GoodsFeign;
import com.baidu.shop.feign.SpecificationFeign;
import com.baidu.shop.service.GoodsService;
import com.baidu.shop.service.ShopElasticsearchService;
import com.baidu.shop.utils.JSONUtil;
import com.netflix.discovery.converters.Auto;
import org.apache.commons.lang.math.NumberUtils;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.autoconfigure.AutoConfigurationPackage;
import org.springframework.util.StringUtils;
import org.springframework.web.bind.annotation.RestController;

import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;
import java.util.stream.Stream;

/**
 * @ClassName ShopElasticsearchServiceImpl
 * @Description: TODO
 * @Author liuhaojie
 * @Date 2021/3/4
 * @Version V1.0
 **/
@RestController
public class ShopElasticsearchServiceImpl extends BaseApiService implements ShopElasticsearchService {

    @Autowired
    private GoodsFeign goodsFeign;

    @Autowired
    private SpecificationFeign specificationFeign;


    @Override
    public Result<JSONObject> esGoodsInfo() {
        SpuDTO spuDTO = new SpuDTO();
        spuDTO.setPage(1);
        spuDTO.setRows(5);
        Result<List<SpuDTO>> spuInfo = goodsFeign.getSpuInfo(spuDTO);
        GoodsDoc goodsDoc = new GoodsDoc();
        //mysql数据迁移到es  先填充spu
        if(spuInfo.getCode() == 200){
                //讲spu数据填充直es
            List<SpuDTO> collect = spuInfo.getData().stream().map(spu -> {

                goodsDoc.setId(spu.getId().longValue());
                goodsDoc.setBrandId(spu.getBrandId().longValue());
                goodsDoc.setBrandName(spu.getBrandName());
                goodsDoc.setCategoryName(spu.getCategoryName());
                goodsDoc.setCid1(spu.getCid1().longValue());
                goodsDoc.setCid2(spu.getCid2().longValue());
                goodsDoc.setCid3(spu.getCid3().longValue());
                goodsDoc.setCreateTime(spu.getCreateTime());
                goodsDoc.setTitle(spu.getTitle());
                goodsDoc.setSubTitle(spu.getSubTitle());


                //讲sku的属性进行填充
                //price的属性也进行填充  通过list集合 并再skus便利的集合中添加
                Result<List<SkuDTO>> skus = goodsFeign.getSkusBySpuId(spu.getId());
                if (skus.getCode() == 200) {

                    List<Long> priceList = new ArrayList<>();
                    List<Map<String, Object>> skuListMap = skus.getData().stream().map(sku -> {
                        Map<String, Object> map = new HashMap<>();
                        map.put("id", sku.getId());
                        map.put("title", sku.getTitle());
                        map.put("images", sku.getImages());
                        map.put("price", sku.getPrice());
                        priceList.add(sku.getPrice().longValue());
                        return map;
                    }).collect(Collectors.toList());

                    goodsDoc.setPrice(priceList);
                    goodsDoc.setSkus(JSONUtil.toJsonString(skuListMap));
                }

                //规格参数填充
                //获取规格参数
                SpecParamDTO specParamDTO = new SpecParamDTO();
                specParamDTO.setCid(spu.getCid3());
                specParamDTO.setSearching(true);//直查询有查询条件的规格参数
                Result<List<SpecParamEntity>> specParamList = specificationFeign.getParamList(specParamDTO);

                if (specParamList.getCode() == 200) {
                    List<SpecParamEntity> ParamList = specParamList.getData();
                    //获取detail数据
                    Result<SpuDetailEntity> spuDetailList = goodsFeign.getSpuDetailBySpuId(spu.getId());
                    if (spuDetailList.getCode() == 200) {
                        SpuDetailEntity spuDetailData = spuDetailList.getData();
                        ////将json字符串转换成map集合
                        Map<String, String> genericSpec = JSONUtil.toMapValueString(spuDetailData.getGenericSpec());
                        Map<String, List<String>> specialSpec = JSONUtil.toMapValueStrList(spuDetailData.getSpecialSpec());




                        //需要查询两张表的数据 spec_param(规格参数名) spu_detail(规格参数值) --> 规格参数名 : 规格参数值
                        HashMap<String, Object> SpecMap = new HashMap<String, Object>();
                        ParamList.stream().forEach(specParam -> {
                            if (specParam.getGeneric()) {
                                if (specParam.getSearching() && !StringUtils.isEmpty(specParam.getSegments())) {
                                    SpecMap.put(specParam.getName(),
                                            chooseSegment(genericSpec.get(specParam.getId() + ""), specParam.getSegments(), specParam.getUnit()));
                                } else {
                                    SpecMap.put(specParam.getName(), genericSpec.get(specParam.getId() + ""));
                                }
                            } else {
                                SpecMap.put(specParam.getName(), specialSpec.get(specParam.getId() + ""));
                            }
                        });

                        goodsDoc.setSpecs(SpecMap);

                    }
                }

                return spu;
            }).collect(Collectors.toList());


            System.out.println(collect);
        }



        return null;
    }



    private String chooseSegment(String value, String segments, String unit) {
        double val = NumberUtils.toDouble(value);
        String result = "其它";
// 保存数值段
        for (String segment : segments.split(",")) {
            String[] segs = segment.split("-");
// 获取数值范围
            double begin = NumberUtils.toDouble(segs[0]);
            double end = Double.MAX_VALUE;
            if(segs.length == 2){
                end = NumberUtils.toDouble(segs[1]);
            }
// 判断是否在范围内
            if(val >= begin && val < end){
                if(segs.length == 1){
                    result = segs[0] + unit + "以上";
                }else if(begin == 0){
                    result = segs[1] + unit + "以下";
                }else{
                    result = segment + unit;
                }
                break;
            }
        }
        return result;
    }

}
```