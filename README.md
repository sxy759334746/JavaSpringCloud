## Java SpringCloud微服务学习


## MybatisPlus

- mp依赖替换mybatis依赖
- 自定义的Mapper继承自mp的BaseMapper接口 
- @TableName @TableId @TableField 
用来在实体类中指明哪个是数据库表名 主键字段 如果名字一致就不用指明
- @TableId中的type有：
AUTO 数据库自增 INPUT set方法自行输入 ASSIGN_ID 默认雪花算法分配的一个Long整数
- QueryWrapper UpdateWrapper lambdaQueryWrapper lambdaUpdateWrapper
- wrapper自定义sql来满足mvc分层 
- IService接口 



## Docker 

- docker run start stop load save build pull push images rmi ps exec
- docker 数据卷挂载 目录挂载
- dockerfile 自定义镜像
- docker compose
- docker network容器网络连接



## 微服务拆分

- 配置service本地运行 新建module
- item-service拆分 cart-service拆分
依赖 启动类 配置文件
每个微服务用同一个mysql的不同database模拟
- 服务调用  暂时配置RestTemplate的Bean来购物车调用item服务  restTemplate.exchange
- lombok spring推荐用构造函数来注入resttemplate
- @RequiredArgsConstructor  rest加final注释

- hm-api 接口调用
- user-service：用户微服务，包含用户登录、管理等功能
- trade-service：交易微服务，包含订单相关功能
- pay-service：支付微服务，包含支付相关功能
- hm-gateway

## Nacos注册中心 

- 多个service集群之间的调用
- Nacos注册中心
- Nacos mysql存储Nacos地址 env修改mysql地址 部署docker容器 
- 添加依赖 修改item-service配置文件 启动多个item-service实例
- cart-service 同样配置 Nacos控制台访问
- DiscoveryCilent调用


## OpenFeign

- Feign 添加OpenFeign和loadBalancer依赖 在启动类添加@EnableFeignCilents注解
- 在service层中写ItemClient接口@FeignClient("item-service") 声明方法 然后在实现类里调用
- Feign底层发起http请求，依赖于其它的框架。其底层支持的http客户端实现包括：
- HttpURLConnection：默认实现，不支持连接池
- Apache HttpClient ：支持连接池
- OKHttp：支持连接池

- feign-okhttp okHttp依赖 开启连接池
- item-service配置文件中okhttp:enabled: true # 开启OKHttp功能
- ### 抽取API模块
- OpenFeign扫描包路径
- @EnableFeignCilents(basePackages = "com.xxx")

- ### 日志配置
- 在api模块写配置类DefaultFeignConfig 用来描述feign的配置 将日志方法设置为@Bean


    @Bean
    public Logger.Level feignLogLevel(){
        return Logger.Level.FULL;}

- 局部生效 在FeignClient里写config属性
- @FeignClient(value = "item-service", configuration = DefaultFeignConfig.class)
- 全局生效
- 启动类 @EnableFeignClients(defaultConfiguration = DefaultFeignConfig.class)



## 网关路由

- 网关路由，解决前端请求入口的问题。
- 网关鉴权，解决统一登录校验和用户信息获取的问题。
- 统一配置管理，解决微服务的配置文件重复和配置热更新问题。

- 引入SpringCloudGateway、NacosDiscovery依赖
- 创建module hm-gatewayyaml文件添加：
- spring:
   - cloud:
    - gateway:
      - routes:
        - id: item
          uri: lb://item-service
          predicates:
            - Path=/items/\*\*,/search/\*\*

- GatewayFilter  过滤任意指定的路由
- GolbalFilter  全局过滤器 范围所有路由
- 自定义GatewayFilter过滤器 需要extends AbstractGatewayFilterFactory<Object> 加Component注解
- 类名必须GatewayFilter结尾
- !!后面还有实现细节，遇到再说

- ### 登录校验 
- 编写自己的GlobalFilter类 implements GlobalFilter，实现filter方法，加component注解 
- 在filter方法中exchange.getRequest获取请求 得到请求头
- antPathMatcher Spring提供的对比路径匹配的工具类 new然后使用match方法
- 如果要控制过滤器的优先级，需要同时implements Orderd接口 实现getOrder方法返回int值，越小越高
- 放行 chain.filter(exchange)
- 拦截响应后 如果发生错误，更改状态码：1.拿到响应 2.设置状态码 3.终止后续过滤器


    ServerHttpRequest re = exchange.getRequest();
    re.setStatusCode(HttpStatus.XXX);
    //枚举类型的状态码
    return re.setComplete();终止后续过滤器 直接返回Mono

- ### 网关传递用户信息
- - 在网关的登录校验中 修改网关请求头 把获取到的用户写入请求头 ServerWebExchange类的exchange.mutate()
- - 编写拦截器获取用户信息并保存到ThreadLocal implements HandlerInterceptor
    实现preHandle 和afterCompletion方法 在pre方法里保存用户信息到ThreadLocal after方法里清除
- - 编写SpringMVC配置类 加configuration注解 实现WebMvcConfigurer 重写addInterceptors方法、
- - resource文件夹下META-INF文件夹spring.factories添加装配信息

- 注意 现在网关里也有这个拦截器 但是不想让他生效 因为拦截器是springMVC的 网关是基于webflux 响应式非阻塞的
- 网关也引用了common包 里的这个config 所以这个config类加一个ContidionalOnClass注解


    org.springframework.boot.autoconfigure.EnableAutoConfiguration=配置类类名
    
    //config类 这个注解在扫描到这个class的时候才会作用
    @ContidionalOnClass(DispatcherServlet.class)

- ### 微服务之间调用传递用户信息
- OpenFeign实现RequestInterceptor接口 实现apply方法 利用RequestTemplate类来添加请求头
- 从ThreadLocal（封装到UserContext）获取用户信息 将用户信息添加到请求头中

- ### 微服务配置管理 热更新
- 微服务共享的配置可以统一交给Nacos保存和管理 在Nacos控制台新增共享yaml文件
- Nacos配置是SpringCloud上下文（ApplicationContext）初始化时处理的，发生在项目的引导阶段。
- 然后才会初始化SpringBoot上下文，去读取application.yaml 也就是说引导阶段，application.yaml文件尚未读取
- SpringCloud在初始化上下文的时候会先读取一个名为bootstrap.yaml(或者bootstrap.properties)的文件
- 需要引入读取bootstrap的包的依赖
- 在Nacos控制台修改配置后，Nacos会将配置变更推送给相关的微服务，并且无需重启即可生效，实现配置热更新。
- 
- Nacos控制台 控制管理

- ### 动态路由
1. 监听Nacos配置的变更
2. 更新路由表
- 在gateway包中新建类编写监听器
- @PostConstruct注解监听器方法 让这个bean初始化之后加载监听器方法
- 注入NacosConfigManager @RequiredArgsConstructor 生成需要的构造函数
- 路由配置使用json格式而不是yaml 方便解析
- RouteDefinitionWriter 在监听器方法中 监听到配置变更后 解析configInfo为RouteDefinition类
- 然后删除旧的路由表 更新新的路由表


    @PostConstruct
    public void initRouteConfigListener() throws NacosException {
        // 1.注册监听器并首次拉取配置
        String configInfo = nacosConfigManager.getConfigService()
                .getConfigAndSignListener(dataId, group, 5000, new Listener() {
                    @Override
                    public Executor getExecutor() {return null;}
                    @Override
                    public void receiveConfigInfo(String configInfo) {
                        updateConfigInfo(configInfo);}});
        updateConfigInfo(configInfo);// 2.首次启动时，更新一次配置
    }

    private void updateConfigInfo(String configInfo) {
        // 1.反序列化
        List<RouteDefinition> routeDefinitions = JSONUtil.toList(configInfo, RouteDefinition.class);
        // 2.更新前先清空旧路由
        // 2.1.清除旧路由
        for (String routeId : routeIds) {
            writer.delete(Mono.just(routeId)).subscribe();
        }
        routeIds.clear();
        // 2.2.判断是否有新的路由要更新
        if (CollUtils.isEmpty(routeDefinitions)) {
            // 无新路由配置，直接结束
            return;
        }
        // 3.更新路由
        routeDefinitions.forEach(routeDefinition -> {
            // 3.1.更新路由
            writer.save(Mono.just(routeDefinition)).subscribe();
            // 3.2.记录路由id，方便将来删除
            routeIds.add(routeDefinition.getId());
- 然后在Naocs里更新gatewat-routes.json 把路由信息配置上

## 服务保护、分布式事务

- ### Sentinel 
- jar包运行 控制台启动 微服务引入依赖 配置文件开启
- 请求限流 线程隔离 服务降级和熔断
- 降级逻辑，
1. 在hm-api模块中给ItemClient定义降级处理类，实现FallbackFactory
2. FallbackFactory类 实现FallbackFactory<ItemClient>接口  泛型是是feign的调用接口客户端类
3. 在hm-api模块中feign配置类 DefaultFeignConfig类 中将ItemClientFallback注册为一个Bean
4. 在hm-api模块中的ItemClient接口中使用ItemClientFallbackFactory  
5. 与定义feign日志类型的一样 写FallbackFactory = (ItemClientFallbackFactory.class)

- 服务熔断
- 断路器的状态机：
- open 打开状态，服务调用被熔断，访问被熔断服务的请求会被拒绝，快速失败，直接走降级逻辑。Open状态持续一段时间后会进入half-open状态
- closed 关闭状态，断路器放行所有请求，并开始统计异常比例、慢请求比例。超过阈值则切换到open状态
- half-open 半开状态，放行一次请求，根据执行结果来判断接下来的操作 成功：则切换到closed状态 失败：则切换到open状态

- ### Seata 分布式事务
- TC (Transaction Coordinator)事务协调者
- TM (Transaction Manager)事务管理器
- RM (Resource Manager)资源管理器

- 数据库准备 docker部署 引入依赖 Nacos添加共享配置
- 微服务trade 添加bootstrap.yml 增加共享shared-saeta.yaml配置
- 使用seata的每个微服务端也要建表保存seate客户端记录的数据
- 在serviceImpl中使用@GlobalTransactional注解
- 分布式事务解决方案 XA TCC AT SAGA


    XA模式
    一阶段：
      事务协调者通知每个事务参与者执行本地事务
      本地事务执行完成后报告事务执行状态给事务协调者，此时事务不提交，继续持有数据库锁
    
    二阶段：
      事务协调者基于一阶段的报告来判断下一步操作
      如果一阶段都成功，则通知所有事务参与者，提交事务
      如果一阶段任意一个参与者失败，则通知所有事务参与者回滚事务
    AT模式
    rm使用undo-log 然后直接提交 tc通知他删除/回滚
    利用数据快照进行回滚

- 3pc:增加了cancommit阶段 引入超时机制 仍然可能阻塞 tcc:try confirm cancel



## 消息队列

- RabbitMQ
- 生产者 消费者 虚拟主机 交换机 队列
- 新建virtual host 绑定exchange和queue
- Docker部署 引入SpringAmqp依赖 注入RabbitTemplate 生产者消费者启动类中更改转换器

        rabbitTemplate.convertAndSend(queueName, message);
        ... 
        @RabbitListener(bindings = @QueueBinding(
            value = @Queue(name = "direct.queue1"),
            exchange = @Exchange(name = "hmall.direct", type = ExchangeTypes.DIRECT),
            key = {"red", "blue"}
        ))
        public void listenDirectQueue1(String msg){
            System.out.println("消费者1接收到direct.queue1的消息：【" + msg + "】");
        }

- Spring Bean声明 xxxExchange Queue Bingding 还可以用工厂模式xxxBuilder 比较麻烦需要新建多个@Bean
- 基于注解的声明 @RabbitListener bindings = @QueueBinding(value exchange key)
- Fanout：广播，将消息交给所有绑定到交换机的队列。我们最早在控制台使用的正是Fanout交换机
- Direct：订阅，基于RoutingKey（路由key）发送给订阅了消息的队列
- Topic：通配符订阅，与Direct类似，只不过RoutingKey可以使用通配符
- Headers：头匹配，基于MQ的消息头匹配，用的较少

- 使用MQ改造下单业务
- ### MQ可靠性和延迟消息
- 生产者确认和重试 配置文件设置超时重试
- - ReturnCallback 一个RabbitTemplate 针对一个Callback 可以在mq配置类中统一配置
- - 在mqconfig中注入 RbbitTemplate @Postconstruct注解添加一个初始化函数 在函数中setReturnCallBack()

        @PostConstruct
        public void init(){
            rabbitTemplate.setReturnsCallback(new RabbitTemplate.ReturnsCallback() {
                @Override
                public void returnedMessage(ReturnedMessage returned) {
                    log.error("触发return callback,");
                    log.debug("exchange: {}", returned.getExchange());
                    log.debug("routingKey: {}", returned.getRoutingKey());
                    log.debug("message: {}", returned.getMessage());
                    log.debug("replyCode: {}", returned.getReplyCode());
                    log.debug("replyText: {}", returned.getReplyText());
                }
            });
        }
- - ConfirmCallback 在发消息时多一个参数 CorrelationData 需要创先然后使用cd.getFuture().addCallback 处理onFailure和onSuccess
- ### MQ数据持久化和LazyQueue
- 消费者确认重试 配置文件配置SpringAMQP消费者确认机制 配置消费者失败重试
- - MessageRecovery接口，自定义重试次数耗尽后的消息处理策略 有3个不同实现：
- - RejectAndDontRequeueRecoverer 重试耗尽后，直接reject，丢弃消息。默认就是这种方式 
- - ImmediateRequeueMessageRecoverer 重试耗尽后，返回nack，消息重新入队 
- - RepublishMessageRecoverer 重试耗尽后，将失败消息投递到指定的交换机 

        @Bean
        public MessageRecoverer republishMessageRecoverer(RabbitTemplate rabbitTemplate){
            return new RepublishMessageRecoverer(rabbitTemplate, "error.direct", "error");
        }


- ### 业务幂等性 
- 每个消息带唯一id 数据库判断是否已处理
- - 唯一消息ID AMQP的MessageConverter自带唯一id;  jjmc.setCreateMessageIds(true);
- - 业务判断 不同业务执行时有自己的判断 订单业务需要先查询订单状态 先判断 后更新 但这样会有线程安全问题
  - - 可以在sql层面用mybatisplus改动


    @Override
    public void markOrderPaySuccess(Long orderId) {
        // UPDATE `order` SET status = ? , pay_time = ? WHERE id = ? AND status = 1
        lambdaUpdate()
                .set(Order::getStatus, 2)
                .set(Order::getPayTime, LocalDateTime.now())
                .eq(Order::getId, orderId)
                .eq(Order::getStatus, 1)
                .update();
    }





- ### 死信交换机 
1. 收集那些因处理失败而被拒绝的消息
2. 收集那些因队列满了而被拒绝的消息
3. 收集因TTL（有效期）到期的消息延迟消息
- 死信交换机可以用来实现延迟消息 
- RabbitMQ的消息过期是基于追溯方式来实现的，也就是说当一个消息的TTL到期以后不一定会被移除或投递到死信交换机，而是在消息恰好处于队首时才会被处理。
当队列中消息堆积很多的时候，过期消息可能不会被按时处理，因此你设置的TTL时间不一定准确。
- 延迟消息DelayExchange插件 docker目录挂载 安装 
- 通过注解或者Bean声明


      @RabbitListener(bindings = @QueueBinding(
              value = @Queue(name = "delay.queue", durable = "true"),
              exchange = @Exchange(name = "delay.direct", delayed = "true"),
              key = "delay"
      ))
      public void listenDelayMessage(String msg){
          log.info("接收到delay.queue的延迟消息：{}", msg);
      }
    //发送演示消息  需要MessagePostProcessor 以及message.getMessageProperties().setDelay(5000)
    @Test
    void testPublisherDelayMessage() {
        // 1.创建消息
        String message = "hello, delayed message";
        // 2.发送消息，利用消息后置处理器添加消息头
        rabbitTemplate.convertAndSend("delay.direct", "delay", message, new MessagePostProcessor() {
            @Override
            public Message postProcessMessage(Message message) throws AmqpException {
                // 添加延迟消息属性
                message.getMessageProperties().setDelay(5000);
                return message;
            }
        });
    }

##  Elasticsearch

- 倒排索引
- ik分词器 扩展词库
- 创建索引库：PUT /索引库名
- 查询索引库：GET /索引库名
- 删除索引库：DELETE /索引库名
- 修改索引库，添加字段：PUT /索引库名/_mapping
- 索引库一旦创建 无法修改 只能新增mapping

- ### RestHighLevelClient
- 初始化RestHighLevelClient
- 创建XxxIndexRequest。XXX是Create、Get、Delete
- 准备请求参数（ Create时需要，其它是无参，可以省略）
- 发送请求。调用RestHighLevelClient#indices().xxx()方法，xxx是create、exists、delete



    @Test
    void testCreateIndex() throws IOException {
        // 1.创建Request对象
        CreateIndexRequest request = new CreateIndexRequest("items");
        // 2.准备请求参数
        request.source(MAPPING_TEMPLATE, XContentType.JSON);
        // 3.发送请求
        client.indices().create(request, RequestOptions.DEFAULT);
    }


- 文档的增删改查java接口
- 批量导入文档


    @Test
    void testBulk() throws IOException {
        // 1.创建Request
        BulkRequest request = new BulkRequest();
        // 2.准备请求参数
        request.add(new IndexRequest("items").id("1").source("json doc1", XContentType.JSON));
        request.add(new IndexRequest("items").id("2").source("json doc2", XContentType.JSON));
        // 3.发送请求
        client.bulk(request, RequestOptions.DEFAULT);
    }

- DSL 全文检索查询 match_all muti-match term range
## Redis高级

- 配置集群
- 主从同步 
- - 全量同步 replid offset  rdb文件 repl_baklog 环形数组
- - 增量同步 repl——baklog






