# 整合amqp的作用



想要效果就是再商城中添加商品时，会自动创建一个对应的。html文件

不会侵入业务代码

增加数据时发送消息

template和search作为消费者来监听

监听到发数据时，会自动创建





## 引入依赖

```xml
<dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-amqp</artifactId>
    </dependency>
```

mingrui-shop-service-xxx

application.yml

```yml
spring:
rabbitmq:
 host: 127.0.0.1
 port: 5672
 username: guest
 password: guest
  # 是否确认回调
 publisher-confirm-type: correlated
  # 是否返回回调
 publisher-returns: true
 virtual-host: /
  # 手动确认
 listener:
  simple:
   acknowledge-mode: manual
```

com.baidu.shop.constant



```java
public class MqMessageConstant {
  //spu交换机，routingkey
  public static final String SPU_ROUT_KEY_SAVE="spu.save";
  public static final String SPU_ROUT_KEY_UPDATE="spu.update";
  public static final String SPU_ROUT_KEY_DELETE="spu.delete";
  //spu-es的队列
  public static  final String SPU_QUEUE_SEARCH_SAVE="spu_queue_es_save";
  public static  final String SPU_QUEUE_SEARCH_UPDATE="spu_queue_es_update";
  public static  final String SPU_QUEUE_SEARCH_DELETE="spu_queue_es_delete";
  //spu-page的队列
  public static  final String SPU_QUEUE_PAGE_SAVE="spu_queue_page_save";
  public static  final String SPU_QUEUE_PAGE_UPDATE="spu_queue_page_update";
  public static  final String SPU_QUEUE_PAGE_DELETE="spu_queue_page_delete";
  public static final String ALTERNATE_EXCHANGE = "exchange.ae";
  public static final String EXCHANGE = "exchange.mr";
  //Dead Letter Exchanges
  public static final String EXCHANGE_DLX = "exchange.dlx";
  public static final String EXCHANGE_DLRK = "dlx.rk";
  public static final Integer MESSAGE_TIME_OUT = 5000;
  public static final String QUEUE = "queue.mr";
  public static final String QUEUE_AE = "queue.ae";
  public static final String QUEUE_DLX = "queue.dlx";
  public static final String ROUTING_KEY = "mrkey";
}
```





```java
import com.baidu.shop.constant.MqMessageConstant;
import lombok.extern.slf4j.Slf4j;
import org.springframework.amqp.core.Message;
import org.springframework.amqp.rabbit.connection.CorrelationData;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;
import java.util.UUID;
/**
* @ClassName MrRabbitMQ
* @Description: TODO
* @Author shenyaqi
* @Date 2020/9/17
* @Version V1.0
**/
@Component
@Slf4j
public class MrRabbitMQ implements RabbitTemplate.ConfirmCallback,
RabbitTemplate.ReturnCallback{
4.2.4 GoodsServiceImpl
  private RabbitTemplate rabbitTemplate;
  //构造方法注入
  @Autowired
  public MrRabbitMQ(RabbitTemplate rabbitTemplate) {
    this.rabbitTemplate = rabbitTemplate;
    //这是是设置回调能收到发送到响应
    rabbitTemplate.setConfirmCallback(this);
    //如果设置备份队列则不起作用
    rabbitTemplate.setMandatory(true);
    rabbitTemplate.setReturnCallback(this);
 }
  public void send(String sendMsg, String routingKey) {
    CorrelationData correlationId = new
CorrelationData(UUID.randomUUID().toString());
    //convertAndSend(exchange:交换机名称,routingKey:路由关键字,object:发送的消息
内容,correlationData:消息ID)
    rabbitTemplate.convertAndSend(MqMessageConstant.EXCHANGE, routingKey,
sendMsg,correlationId);
 }
  @Override
  public void confirm(CorrelationData correlationData, boolean b, String s) {
    if(b){
      log.info("消息发送成
功:correlationData({}),ack({}),cause({})",correlationData,b,s);
   }else{
      log.error("消息发送失
败:correlationData({}),ack({}),cause({})",correlationData,b,s);
   }
 }
  @Override
  public void returnedMessage(Message message, int i, String s, String s1,
String s2) {
    log.warn("消息丢
失:exchange({}),route({}),replyCode({}),replyText({}),message:
{}",s1,s2,i,s,message);
 }
}
```





```java
@Transactional
  @Override
  public Result<JSONObject> saveGoods(SpuDTO spuDTO) {
    //新增spu
    SpuEntity spuEntity = BaiduBeanUtil.copyProperties(spuDTO,
SpuEntity.class);
    spuEntity.setSaleable(1);
    spuEntity.setValid(1);
    final Date date = new Date();//保持两个时间一致
    spuEntity.setCreateTime(date);
    spuEntity.setLastUpdateTime(date);
    spuMapper.insertSelective(spuEntity);
    //新增spuDetail
    SpuDetailEntity spuDetailEntity =
BaiduBeanUtil.copyProperties(spuDTO.getSpuDetail(), SpuDetailEntity.class);
    spuDetailEntity.setSpuId(spuEntity.getId());
    spuDetailMapper.insertSelective(spuDetailEntity);
    this.saveSkusAndStocks(spuDTO.getSkus(),spuEntity.getId(),date);
    mrRabbitMQ.send(spuEntity.getId() + "",
MqMessageConstant.SPU_ROUT_KEY_SAVE);
    return this.setResultSuccess();
 }
```





application.yml



```yml
spring:
rabbitmq:
 host: 127.0.0.1
 port: 5672
 username: guest
 password: guest
  # 是否确认回调
 publisher-confirm-type: correlated
  # 是否返回回调
 publisher-returns: true
 virtual-host: /
  # 手动确认
 listener:
4.3.2 ShopElasticsearchService
4.3.3 ShopElasticsearchServiceImpl
  simple:
   acknowledge-mode: manual
```



### ShopElasticsearchService

```java
 @ApiOperation(value = "新增数据到es")
  @PostMapping(value = "es/saveData")
  Result<JSONObject> saveData(Integer spuId);
  @ApiOperation(value = "通过id删除es数据")
  @DeleteMapping(value = "es/saveData")
  Result<JSONObject> delData(Integer spuId);
```

```java
 @Override
  public Result<JSONObject> delData(Integer spuId) {
    GoodsDoc goodsDoc = new GoodsDoc();
    goodsDoc.setId(spuId.longValue());
    elasticsearchRestTemplate.delete(goodsDoc);
    return this.setResultSuccess();
 }
```

```java
@Component
@Slf4j
public class GoodsListener {
  @Autowired
  private ShopElasticsearchService shopElasticsearchService;
  @RabbitListener(
      bindings = @QueueBinding(
          value = @Queue(
              value = MqMessageConstant.SPU_QUEUE_SEARCH_SAVE,
4.4 mingrui-shop-service-template
              durable = "true"
         ),
          exchange = @Exchange(
              value = MqMessageConstant.EXCHANGE,
              ignoreDeclarationExceptions = "true",
              type = ExchangeTypes.TOPIC
         ),
          key =
{MqMessageConstant.SPU_ROUT_KEY_SAVE,MqMessageConstant.SPU_ROUT_KEY_UPDATE}
     )
 )
  public void save(Message message, Channel channel) throws IOException {
    log.info("es服务接受到需要保存数据的消息: " + new String(message.getBody()));
    //新增数据到es
    shopElasticsearchService.saveData(Integer.parseInt(new
String(message.getBody())));
    channel.basicAck(message.getMessageProperties().getDeliveryTag(), true);
 }
  @RabbitListener(
      bindings = @QueueBinding(
          value = @Queue(
              value = MqMessageConstant.SPU_QUEUE_SEARCH_DELETE,
              durable = "true"
         ),
          exchange = @Exchange(
              value = MqMessageConstant.EXCHANGE,
              ignoreDeclarationExceptions = "true",
              type = ExchangeTypes.TOPIC
         ),
          key = MqMessageConstant.SPU_ROUT_KEY_DELETE
     )
 )
  public void delete(Message message, Channel channel) throws IOException {
    log.info("es服务接受到需要删除数据的消息: " + new String(message.getBody()));
    //新增数据到es
    shopElasticsearchService.delData(Integer.parseInt(new
String(message.getBody())));
    channel.basicAck(message.getMessageProperties().getDeliveryTag(), true);
 }
}
```

```java
@Component
@Slf4j
public class TemplateListener {
  @Autowired
  private TemplateService templateService;
  @RabbitListener(
      bindings = @QueueBinding(
          value = @Queue(
              value = MqMessageConstant.SPU_QUEUE_PAGE_SAVE,
              durable = "true"
         ),
          exchange = @Exchange(
              value = MqMessageConstant.EXCHANGE,
              ignoreDeclarationExceptions = "true",
              type = ExchangeTypes.TOPIC
         ),
          key =
{MqMessageConstant.SPU_ROUT_KEY_SAVE,MqMessageConstant.SPU_ROUT_KEY_UPDATE}
     )
 )
  public void save(Message message, Channel channel) throws IOException {
    log.info("template服务接受到需要保存数据的消息: " + new
String(message.getBody()));
    //根据spuId生成页面
    templateService.createStaticHTMLTemplate(Integer.valueOf(new
String(message.getBody())));
    channel.basicAck(message.getMessageProperties().getDeliveryTag(), true);
 }
  @RabbitListener(
      bindings = @QueueBinding(
          value = @Queue(
              value = MqMessageConstant.SPU_QUEUE_PAGE_DELETE,
              durable = "true"
         ),
          exchange = @Exchange(
              value = MqMessageConstant.EXCHANGE,
              ignoreDeclarationExceptions = "true",
              type = ExchangeTypes.TOPIC
         ),
          key = MqMessageConstant.SPU_ROUT_KEY_DELETE
     )
 )
  public void delete(Message message, Channel channel) throws IOException {
4.5 第一次测试
4.5.1 环境准备
依次启动eureka-server,xxx,search,template,upload,zuul,两个前端项目manage和portal
    log.info("template服务接受到需要删除数据的消息: " + new
String(message.getBody()));
    //根据spuid删除页面
    templateService.delHTMLBySpuId(Integer.valueOf(new
String(message.getBody())));
    channel.basicAck(message.getMessageProperties().getDeliveryTag(), true);
 }
}
```

