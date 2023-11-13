# 异步接口执行的过程中发起Feign调用


#### 业务流程

> 用户发起请求后,在A服务中执行业务逻辑,执行过程中需要调用其他微服务B进行流程审批,简单方式是在A服务中所有逻辑(1,2,3)采用同步逻辑放在一个事务中,但是如果调用B服务的过程中,处理逻辑很复杂,会导致A服务的请求一直阻塞,用户体验很不好

![image-20231024110959505](https://fanzhengxiang.oss-cn-chengdu.aliyuncs.com/blog-img/image-20231024110959505.png)

优化后的逻辑为将第2步,调整为异步逻辑,异步调用服务B,可以使A服务快速完成逻辑处理,相应用户,B服务在流程发起成功之后,再回调A服务,回写流程发起的结果(流程发起成功或者流程发起失败的原因),此时就引申出一个问题: 在异步调用时,feign接口的请求会丢失掉请求头中的Token,导致在调用服务B时出现401

#### 解决方案

> 网上大多数的解决办法都是抄来抄去使用拦截器方式实现,不推荐,我这里采用的是Feign官方的解决方案,Feign Builder

[手动创建Feign Client](https://cloud.spring.io/spring-cloud-netflix/multi/multi_spring-cloud-feign.html)

![image-20231024112012585](https://fanzhengxiang.oss-cn-chengdu.aliyuncs.com/blog-img/image-20231024112012585.png)

#### 实际操作

微服务B现有的Feign接口如下:

```java


@FeignClient(
    name = "eip-bpm-runtime",
    url = "${eip-bpm-runtime:}",
    fallbackFactory = FlowApiFallback.class
)
public interface FlowApi {
  
    @RequestMapping(
        value = {"/flow/instance/v1/start"},
        method = {RequestMethod.POST},
        produces = {"application/json; charset=utf-8"}
    )
    ObjectNode start(@RequestBody JSONObject var1) throws Exception;

}

```

现在需要在A服务中手动创建一个`FlowApi`的实例发起调用

首先在A服务中创建一个配置类

```java


import com.aliyun.openservices.shade.com.alibaba.fastjson.support.spring.FastJsonJsonView;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.hotent.api.FlowApi;
import feign.Client;
import feign.Feign;
import org.apache.http.HttpHeaders;
import org.springframework.beans.factory.ObjectFactory;
import org.springframework.boot.autoconfigure.http.HttpMessageConverters;
import org.springframework.cloud.openfeign.support.SpringDecoder;
import org.springframework.cloud.openfeign.support.SpringEncoder;
import org.springframework.cloud.openfeign.support.SpringMvcContract;
import org.springframework.http.converter.HttpMessageConverter;
import org.springframework.http.converter.json.MappingJackson2HttpMessageConverter;
import org.springframework.stereotype.Component;

import javax.annotation.Resource;

/**
 * BPM 异步feign请求头设置
 *
 * @author kiki
 * @date 2022/1/13 14:25
 * @since 0.0.1
 */
@Component
public class BpmFlowApiConfig {

    @Resource
    private Client client;

    /**
     * 接口请求path
     */
    private String url;

    public void setUrl(String url) {
        this.url = url;
    }

    public FlowApi flowApiClient(String accessToken){
        HttpMessageConverter jsonConverter = new MappingJackson2HttpMessageConverter(new ObjectMapper());
        ObjectFactory<HttpMessageConverters> converter = () -> new HttpMessageConverters(jsonConverter);
        return Feign.builder().client(client)
                .encoder(new SpringEncoder(converter))
                .decoder(new SpringDecoder(converter))
                .contract(new SpringMvcContract())
                .requestInterceptor(template -> template.header(org.springframework.http.HttpHeaders.AUTHORIZATION, accessToken))
                .requestInterceptor(template -> template.header(HttpHeaders.CONTENT_TYPE, FastJsonJsonView.DEFAULT_CONTENT_TYPE))
                .target(FlowApi.class, url);

    }
}

```

然后在需要发起对B服务调用的地方注入该配置类,并手动构建Feign客户端发起调用

```java
/**
 * desc
 *
 * @author kiki
 * @date 2022/1/13 11:37
 * @since 0.0.1
 */
@Service
@Slf4j
public class IAsyncServiceImpl implements IAsyncService {

    /**
     * 目标微服务的服务名称
     * 
     */
    @Value("${bpm.url:http://eip-bpm-runtime/}")
    private String flowApiUrl;

    @Resource
    private BpmFlowApiConfig bpmFlowApiConfig;


    /**
     * 异步上传发票并发起审批
     *
     * @param paymentAddEo 请求参数
     * @param header 上游请求的Token
     */
    @Async
    @Override
    public void uploadInvoicesAndStartBpmProcess(PaymentBillAddReqDto paymentAddEo, String header) {
        // TODO A服务的自身业务逻辑...
         startNewBpmProcess(paymentAddEo, fileIdList, header);
        // TODO A服务的自身后续业务逻辑...
       
    }

    private void startNewBpmProcess(PaymentBillAddReqDto paymentAddEo, List<String> fileIdList, String header) {
        try {
            // ...省略业务逻辑
            log.info("{}发起bpm申请流程参数：{}", paymentBillDesc, paramStr);
            // 这里是发起调用配置的关键
            // 1.设置调用服务的服务名
            // 2.手动为Feign Client设置header(token)
            bpmFlowApiConfig.setUrl(flowApiUrl);
            start = bpmFlowApiConfig.flowApiClient(header).start(JSON.parseObject(paramStr));
            ObjectMapper objectMapper = new ObjectMapper();
            String resultStr = objectMapper.writeValueAsString(start);
            log.info("{}发起BPM审批请求结果：{}", paymentBillDesc, resultStr);
            // ...省略业务逻辑
            bpmProcessService.add(reqDto);
        } catch (Exception e) {
            log.error("{}失败，bpm返回信息{}，保存的流程信息{}，错误信息：{}", paymentBillDesc, JSON.toJSONString(start), JSONObject.toJSONString(paymentAddEo, SerializerFeature.PrettyFormat), e);
            throw new BizException(String.format("%s发起BPM审批失败！", e.getMessage()));
        }
    }
}

```

其中,A服务在请求发起时,传递的Token需要在A服务发起调用前事先获取

```java
ServletRequestAttributes requestAttributes = (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
HttpServletRequest request = requestAttributes.getRequest();
String accessToken = request.getHeader("Access-Token");
```





