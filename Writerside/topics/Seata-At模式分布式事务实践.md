# Seata_At模式分布式事务实践

#### 演示案例架构图

![image-20231024141913651](https://fanzhengxiang.oss-cn-chengdu.aliyuncs.com/blog-img/image-20231024141913651.png)

#### 中间件服务构建

根据SpringCloudAlibaba与SpringCloud的[对应关系](https://github.com/alibaba/spring-cloud-alibaba/wiki/%E7%89%88%E6%9C%AC%E8%AF%B4%E6%98%8E),本次案例实现采用中间件/框架版本如下

- Spring Boot: 2.6.13
- Spring Cloud: 2021.0.5
- Spring Cloud Alibaba: 2021.0.5.0
- Nacos: 2.2.0
- Seata: 1.6.1
- Mysql: 8.1.0

##### Nacos构建

从[这里](https://github.com/alibaba/nacos/releases/tag/2.2.2)下载Nacos预编译包,nacos配置数据我们采用持久化到数据库的方式

首先准备nacos的数据库实例:

```yaml
version: '3.1'
services:
  nacos:
    image: mysql:8.1.0
    restart: always
    container_name: mysql_nacos
    environment:
      MYSQL_ROOT_PASSWORD: 5566
    command:
      --default-authentication-plugin=mysql_native_password
      --character-set-server=utf8mb4
      --collation-server=utf8mb4_general_ci
      --explicit_defaults_for_timestamp=true
      --lower_case_table_names=1
    ports:
      - 3306:3306
    volumes:
      - ./config/data/mysql:/var/lib/mysql

```

然后修改Nacos包中的scheme导入到该数据库实例中

![image-20231024143134671](https://fanzhengxiang.oss-cn-chengdu.aliyuncs.com/blog-img/image-20231024143134671.png)

修改conf目录下的application.properties文件,主要修改以下几项(单机模式,集群参考官方文档修改):

```properties
#*************** Config Module Related Configurations ***************#
### If use MySQL as datasource:
spring.datasource.platform=mysql

### Count of DB:
db.num=1

### Connect URL of DB:
db.url.0=jdbc:mysql://127.0.0.1:3306/nacos?characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true&useUnicode=true&useSSL=false&serverTimezone=UTC
db.user.0=root
db.password.0=5566
```

[启动nacos](http://127.0.0.1:8848/nacos/index.html):

`/Users/fanzhengxiang/data/service/nacos2.2.0/bin/startup.sh -m standalone`

![image-20231024143828005](https://fanzhengxiang.oss-cn-chengdu.aliyuncs.com/blog-img/image-20231024143828005.png)

##### Seata构建

从[这里](https://github.com/seata/seata/releases/tag/v1.6.1)下载seata的预编译包,本次实验seata数据同样持久化到数据库中,准备一个数据库实例

```yaml
version: '3.1'
services:
  seata_db:
    image: mysql:8.1.0
    restart: always
    container_name: mysql_seata
    environment:
      MYSQL_ROOT_PASSWORD: 5566
    command:
      --default-authentication-plugin=mysql_native_password
      --character-set-server=utf8mb4
      --collation-server=utf8mb4_general_ci
      --explicit_defaults_for_timestamp=true
      --lower_case_table_names=1
    ports:
      - 3307:3306
    volumes:
      - ./config/data/mysql:/var/lib/mysql

```

将mysql的建表数据导入到seata的数据库实例中

![image-20231024144338602](https://fanzhengxiang.oss-cn-chengdu.aliyuncs.com/blog-img/image-20231024144338602.png)

修改seata的启动配置文件`conf/application.yml`,主要修改config项,registry项目,store项目,以下是实例

> 注意:由于mysql的版本为8,所以驱动连接为`com.mysql.cj.jdbc.Driver`,并且需要下载mysql8的[java驱动](https://repo1.maven.org/maven2/com/mysql/mysql-connector-j/8.1.0/mysql-connector-j-8.1.0.jar)到seata的lib报下

```yaml
#  Copyright 1999-2019 Seata.io Group.
#
#  Licensed under the Apache License, Version 2.0 (the "License");
#  you may not use this file except in compliance with the License.
#  You may obtain a copy of the License at
#
#  http://www.apache.org/licenses/LICENSE-2.0
#
#  Unless required by applicable law or agreed to in writing, software
#  distributed under the License is distributed on an "AS IS" BASIS,
#  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
#  See the License for the specific language governing permissions and
#  limitations under the License.

server:
  port: 7091

spring:
  application:
    name: seata-server

logging:
  config: classpath:logback-spring.xml
  file:
    path: ${user.home}/logs/seata
  # extend:
  #   logstash-appender:
  #     destination: 127.0.0.1:4560
  #   kafka-appender:
  #     bootstrap-servers: 127.0.0.1:9092
  #     topic: logback_to_logstash

console:
  user:
    username: seata
    password: seata

seata:
  config:
    # support: nacos 、 consul 、 apollo 、 zk  、 etcd3
    type: nacos
    nacos:
      server-addr: 127.0.0.1:8848
      namespace:
      group: SEATA_GROUP
      username:
      password:
      context-path:
      ##if use MSE Nacos with auth, mutex with username/password attribute
      #access-key:
      #secret-key:
      data-id: seataServer.properties
  registry:
    # support: nacos 、 eureka 、 redis 、 zk  、 consul 、 etcd3 、 sofa
    type: nacos
    nacos:
      application: seata-server
      server-addr: 127.0.0.1:8848
      group: SEATA_GROUP
      namespace:
      cluster: default
      username:
      password:
      context-path:
      ##if use MSE Nacos with auth, mutex with username/password attribute
      #access-key:
      #secret-key:
  store:
    db:
      datasource: druid
      db-type: mysql
      ## 这里注意mysql8的驱动连接为com.mysql.cj.jdbc.Driver
      driver-class-name: com.mysql.cj.jdbc.Driver
      url: jdbc:mysql://127.0.0.1:3307/seata?rewriteBatchedStatements=true
      user: root
      password: 5566
      min-conn: 10
      max-conn: 100
      global-table: global_table
      branch-table: branch_table
      lock-table: lock_table
      distributed-lock-table: distributed_lock
      query-limit: 1000
      max-wait: 5000
#  server:
#    service-port: 8091 #If not configured, the default is '${server.port} + 1000'
  security:
    secretKey: SeataSecretKey0c382ef121d778043159209298fd40bf3850a017
    tokenValidityInMilliseconds: 1800000
    ignore:
      urls: /,/**/*.css,/**/*.js,/**/*.html,/**/*.map,/**/*.svg,/**/*.png,/**/*.ico,/console-fe/public/**,/api/v1/auth/login
```

启动seata: `nohup sh bin/seata-server.sh -p 18091 -n 1 &`

nacos: 注册信息

![image-20231024145159020](https://fanzhengxiang.oss-cn-chengdu.aliyuncs.com/blog-img/image-20231024145159020.png)

[seata控制台](http://127.0.0.1:7091)

![image-20231024145355822](https://fanzhengxiang.oss-cn-chengdu.aliyuncs.com/blog-img/image-20231024145355822.png)

#### 代码准备

> 源码在[这里]()查看

准备三个微服务模块的数据库实例

```yaml
version: '3.1'
services:
  seata_order:
    image: mysql:8.1.0
    restart: always
    container_name: seata_order
    environment:
      MYSQL_ROOT_PASSWORD: 5566
    command:
      --default-authentication-plugin=mysql_native_password
      --character-set-server=utf8mb4
      --collation-server=utf8mb4_general_ci
      --explicit_defaults_for_timestamp=true
      --lower_case_table_names=1
    ports:
      - 3308:3306
    volumes:
      - ./order/config/data/mysql:/var/lib/mysql

  seata_inventory:
    image: mysql:8.1.0
    restart: always
    container_name: seata_inventory
    environment:
      MYSQL_ROOT_PASSWORD: 5566
    command:
      --default-authentication-plugin=mysql_native_password
      --character-set-server=utf8mb4
      --collation-server=utf8mb4_general_ci
      --explicit_defaults_for_timestamp=true
      --lower_case_table_names=1
    ports:
      - 3309:3306
    volumes:
      - ./inventory/config/data/mysql:/var/lib/mysql

  seata_account:
    image: mysql:8.1.0
    restart: always
    container_name: seata_account
    environment:
      MYSQL_ROOT_PASSWORD: 5566
    command:
      --default-authentication-plugin=mysql_native_password
      --character-set-server=utf8mb4
      --collation-server=utf8mb4_general_ci
      --explicit_defaults_for_timestamp=true
      --lower_case_table_names=1
    ports:
      - 3310:3306
    volumes:
      - ./account/config/data/mysql:/var/lib/mysql

```

项目依赖:

```xml
<properties>
   <maven.compiler.target>8</maven.compiler.target>
   <maven.compiler.source>8</maven.compiler.source>
   <spring.boot.version>2.6.13</spring.boot.version>
   <spring.cloud.version>2021.0.5</spring.cloud.version>
   <spring.cloud.alibaba.version>2021.0.5.0</spring.cloud.alibaba.version>
</properties>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-parent</artifactId>
            <version>${spring.boot.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>${spring.cloud.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-alibaba-dependencies</artifactId>
            <version>${spring.cloud.alibaba.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>

        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>1.18.28</version>
        </dependency>
    </dependencies>
</dependencyManagement>
```

以订单服务为例

![image-20231024145709277](https://fanzhengxiang.oss-cn-chengdu.aliyuncs.com/blog-img/image-20231024145709277.png)

项目配置文件`application.yml`

```yaml
server:
  port: 8081 # 端口

spring:
  application:
    name: order-service
  datasource:
    url: jdbc:mysql://127.0.0.1:3308/seata_order?useSSL=false&useUnicode=true&characterEncoding=UTF-8
    driver-class-name: com.mysql.cj.jdbc.Driver
    username: root
    password: 5566
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848
seata:
  application-id: ${spring.application.name}
  tx-service-group: ${spring.application.name}-group
  service:
    vgroup-mapping:
      order-service-group: default
  registry:
    type: nacos
    nacos:
      cluster: default
      server-addr: localhost
      namespace:

```

其中账户服务和库存服务的Feign接口如下

```java
import com.aires.feign.dto.AccountReduceBalanceDTO;
import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;

/**
 * `account-service` 服务的 Feign 客户端
 */
@FeignClient(name = "account-service")
public interface AccountServiceFeignClient {

    @PostMapping("/account/reduce-balance")
    void reduceBalance(@RequestBody AccountReduceBalanceDTO accountReduceBalanceDTO);

}

```

```java
import com.aires.feign.dto.ProductReduceStockDTO;
import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;

/**
 * `product-service` 服务的 Feign 客户端
 */
@FeignClient(name = "product-service")
public interface ProductServiceFeignClient {

    @PostMapping("/product/reduce-stock")
    void reduceStock(@RequestBody ProductReduceStockDTO productReduceStockDTO);

}

```

下单的核心业务代码

```java
@Service
public class OrderServiceImpl implements OrderService {

    private Logger logger = LoggerFactory.getLogger(getClass());

    @Resource
    private OrderDao orderDao;

    @Resource
    private AccountServiceFeignClient accountService;

    @Resource
    private ProductServiceFeignClient productService;

    @Override
    @GlobalTransactional
    public Integer createOrder(Long userId, Long productId, Integer price) {
        Integer amount = 1; // 购买数量，暂时设置为 1。
        logger.info("[createOrder] 当前 XID: {}", RootContext.getXID());

        // 扣减库存
        productService.reduceStock(new ProductReduceStockDTO().setProductId(productId).setAmount(amount));

        // 扣减余额
        accountService.reduceBalance(new AccountReduceBalanceDTO().setUserId(userId).setPrice(price));

        // 保存订单
        OrderDO order = new OrderDO().setUserId(userId).setProductId(productId).setPayAmount(amount * price);
        orderDao.saveOrder(order);
        logger.info("[createOrder] 保存订单: {}", order.getId());

        // 返回订单编号
        return order.getId();
    }

}
```

#### 分布式事务演示

// TODO
