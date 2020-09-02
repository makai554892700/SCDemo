## SpringCloud+eureka/zookeeper+robbin+hystrix+openFeign+zuul/gateway+SpringCloud config+RabbitMQ
* eureka的集成(注册中心)

  * 新建springboot项目并在build.gradle文件添加相关依赖

        ext {
            set('springCloudVersion', "Hoxton.SR4")
        }
        
        dependencies {
            implementation 'org.springframework.cloud:spring-cloud-starter-netflix-eureka-server'
        }
        dependencyManagement {
            imports {
                mavenBom "org.springframework.cloud:spring-cloud-dependencies:${springCloudVersion}"
            }
        }

  * 在新建的eureka服务项目中配置相关文件(application.yml对应配置可自行更改，下面给的是单机配置)

        eureka:
          instance:
            hostname: localhost
          # 是否向注册中心注册服务(如果单eureka服务则false/避免自己对自己注册服务)
          client:
            register-with-eureka: false
            # 是否去检索其它服务，
            fetch-registry: false
            # 指定服务注册中心的位置
            service-url:
                defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/

  * 在新建的eureka服务项目Application主类添加注解 @EnableEurekaServer

        @EnableEurekaServer
        public class EurekaServerApplication {

  * 此时再启动新建的eureka服务项目便完成了单eureka服务注册中心的搭建
  * eureka集群的配置
    * 配置文件(在项目中复制配置文件，并加后缀<application-eureka8761.yml>)
      * 配置文件1

            server:
              port: 8761
            eureka:
              instance:
                hostname: eureka8761
              # 是否向注册中心注册服务(如果单eureka服务则false/避免自己对自己注册服务)
              client:
                register-with-eureka: false
                # 是否去检索其它服务，
                fetch-registry: false
                # 指定服务注册中心的位置
                service-url:
                    defaultZone: http://eureka8762:eureka8762/eureka/

      * 配置文件2

            server:
              port: 8762
            eureka:
              instance:
                hostname: eureka8762
              # 是否向注册中心注册服务(如果单eureka服务则false/避免自己对自己注册服务)
              client:
                register-with-eureka: false
                # 是否去检索其它服务，
                fetch-registry: false
                # 指定服务注册中心的位置
                service-url:
                    defaultZone: http://eureka8761:eureka8761/eureka/

    * 因为hostname改变了，但是本地并不能识别，因此需要更改本地host文件，添加本地配置
    
          127.0.0.1 eureka8761
          127.0.0.1 eureka8762

  * 运行时命令

        java -jar xxx.jar --spring.profiles.active=<服务名(eureka8761)>

  * eureka常用配置
    
        // 服务心跳s
        eureka.instance.lease-renewal-interval-in-seconds=2
        // 服务判定故障时间s
        eureka.instance.lease-expiration-duration-in-seconds=10
        
        
  * 普通服务提供者/消费者接入eureka注册中心
    * 提供者注册服务
    * 为服务提供者项目的build.gradle项目添加eureka相关依赖

          ext {
            set('springCloudVersion', "Hoxton.SR4")
          }
        
          dependencies {
            implementation 'org.springframework.cloud:spring-cloud-starter-netflix-eureka-client'
          }
          dependencyManagement {
            imports {
                mavenBom "org.springframework.cloud:spring-cloud-dependencies:${springCloudVersion}"
            }
          }

    * 为服务提供者项目的application.yml添加eureka相关配置(多配置时service-url以,分隔)

          # 服务名称
          spring:
            application:
              name: cloud-provider
          # eureka注册中心连接方式
          eureka:
            client:
              service-url: 
                defaultZone: http://localhost:8761/eureka,http://localhost:8762/eureka

    * 在服务提供者项目的Application主类添加注解 @EnableEurekaClient/@EnableDiscoveryClient 以使配置文件生效并注册服务/表明以可以使用eureka服务

          @EnableDiscoveryClient
          public class CloudProviderApplication {

    * 此时再启动服务提供者项目，服务便注册至eureka注册中心了
  * 服务者的调用
    * 为消费者项目的build.gradle项目添加eureka相关依赖

          ext {
            set('springCloudVersion', "Hoxton.SR4")
          }
        
          dependencies {
            implementation 'org.springframework.cloud:spring-cloud-starter-netflix-eureka-client'
          }
          dependencyManagement {
            imports {
                mavenBom "org.springframework.cloud:spring-cloud-dependencies:${springCloudVersion}"
            }
          }

    * 为服务消费者项目的application.yml添加eureka相关配置(多配置时service-url以,分隔)

          # 服务名称
          spring:
            application:
              name: cloud-consumer
          # eureka注册中心连接方式
          eureka:
            client:
              service-url: 
                defaultZone: http://localhost:8761/eureka,http://localhost:8762/eureka

    * 再在消费者RestTemplate bean注入的地方加入 @LoadBalanced 注解加入负载均衡功能

          @Bean
          @LoadBalanced
          public RestTemplate restTemplate() {

    * 此时只需要将服务消费者的restTemplate.getForEntity方法里的host改为服务提供者名称即可(例：<restTemplate.getForEntity("http://cloud-provider/provider/hello"...>)

          @RestController
          public class ConsumerController {
        
            @Autowired
            RestTemplate restTemplate;
            
            @RequestMapping("/consumer/hello")
            public String hello() {
                // eureka注册服务调用方式 直接返回数据
                return restTemplate.getForObject("http://cloud-provider/provider/hello"
                            , String.class);
            }
          }

   * ribbon负载均衡的使用
    * 启动多个服务提供者并注册至注册中心(eureka)
    * 在消费者RestTemplate bean注入的地方加入 @LoadBalanced 注解

          @Bean
          @LoadBalanced
          public RestTemplate restTemplate() {

    * ribbon负载均衡策略与配置
      * 策略
        * RoundRobinRule                轮询策略(默认)
        * RandomRule                    随机策略
        * RetryRule                     重试策略,失败重试，错误过多访问其它服务
        * AvailabilityFilteringRule     过滤故障/并发连接超阈值服务，对剩下的服务轮询
        * BestAvailableRule             过滤访问过多故障的服务，返回并发量最小的服务
        * ZoneAvoidanceRule             综合判断性能/可用性，返回更优的服务
    * 配置

        * 配置bean注入
        
                @Bean
                public IRule iRule() {
                    return new RandomRule();//根据自己的需求返回不同的rule实例
                }

* Hystrix 集成(熔断器)
  * 为项目的build.gradle项目添加hystrix相关配置

        dependencies {
            implementation 'org.springframework.cloud:spring-cloud-starter-netflix-hystrix'
        }

  * 在项目的Application主类添加注解 @EnableCircuitBreaker

        @EnableCircuitBreaker
        public class CloudProviderApplication {

  * 在项目的controller类中添加注解 @HystrixCommand 及方法错误时调用方法

        @HystrixCommand(fallbackMethod = "error")
        public String hystrix() {
            //业务处理
        }
        
        public String error() {
            //错误处理
            return "error";
        }

  * 同时 @SpringBootApplication/@EnableEurekaClient/@EnableCircuitBreaker 三个注解可用 @SpringCloudApplication 代替
  * @EnableDiscoveryClient 与 @EnableEurekaClient 相似
  * hystrix默认超时时间为1000ms
  * hystrix 常用配置

        // 设置超时时间
        @HystrixCommand(fallbackMethod = "error", commandProperties = {
                    @HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds", value = "1500")
            })

  * hystrix的服务降级(当某个服务产生了熔断，此服务将不再被调用)
  * hystrix的异常处理(当调用者自己抛出了异常，此时也会触发服务的降级，当我们自己发生异常时，只需要在服务降级方法中添加一个Throwable类型的参数就能够获取到抛出的异常类型)

        public String error(Throwable throwable) {
            return "error:" + (throwable == null ? "null" : throwable.getMessage());
        }

  * 如果需要抛某些异常给用户，则直接在 @HystrixCommand 注解中添加 ignoreExceptions = {<忽略的异常类>}

        @HystrixCommand(fallbackMethod = "error", commandProperties = {
                    @HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds", value = "1500")
            }, ignoreExceptions = {
                    Exception.class
            })

  * hystrix自定义请求服务熔断
    * 自定义类并继承 com.netflix.hystrix.HystrixCommand 类
    * 覆写 run 方法执行业务逻辑并返回业务数据
    * 覆写 getFallback 方法处理熔断逻辑
    * 重载构造方法传入必要参数Setter及RestTemplate
    * 下面给出示例:
        
            public class MyHystrixCommand extends HystrixCommand<String> {
            
                private RestTemplate restTemplate;
            
                public MyHystrixCommand(Setter setter, RestTemplate restTemplate) {
                    super(setter);
                    this.restTemplate = restTemplate;
                }
            
                @Override
                protected String run() throws Exception {
            //        int b = 10 / 0;//异常模拟
                    return restTemplate.getForObject("http://cloud-provider/provider/hello"
                            , String.class);
                }
            
                @Override
                protected String getFallback() {
                    Throwable throwable = getExecutionException();
                    return "error2:" + (throwable == null ? "null" : throwable.getMessage());
                }
            
            }

  * 调用时则不再需要写注解，而是直接 new 自定义类，然后调用 execute 方法返回结果。

        @RequestMapping("/consumer/hystrix2")
        public String hystrix2() {
            //使用默认的setter
            MyHystrixCommand myHystrixCommand = new MyHystrixCommand(
                    com.netflix.hystrix.HystrixCommand.Setter.withGroupKey(HystrixCommandGroupKey.Factory.asKey(""))
                    , restTemplate);
            return myHystrixCommand.execute();
        }

  * 异步调用方法

        MyHystrixCommand myHystrixCommand = new MyHystrixCommand(
                com.netflix.hystrix.HystrixCommand.Setter.withGroupKey(HystrixCommandGroupKey.Factory.asKey(""))
                , restTemplate);
        Future<String> future = myHystrixCommand.queue();
        //可以在这里做一些其它逻辑
        //阻塞方法
        String result = future.get();
        return result;

* hystrix dashboard (hystrix仪表盘监控)接入
  * 新建项目并为项目的build.gradle项目添加hystrix-dashboard相关配置

        ext {
            set('springCloudVersion', "Hoxton.SR4")
        }
        
        dependencies {
            implementation 'org.springframework.cloud:spring-cloud-starter-netflix-hystrix-dashboard'
        }

        dependencyManagement {
            imports {
                mavenBom "org.springframework.cloud:spring-cloud-dependencies:${springCloudVersion}"
            }
        }

  * 在项目的Application主类添加注解 @EnableHystrixDashboard

        @EnableHystrixDashboard
        public class HystrixDashboardApplication {

  * 启动项目并访问 http://\<host>:\<port>/hystrix (http://localhost:3721/hystrix) 查看服务是否部署成功
  * 为需要监控的项目的build.gradle文件添加依赖

        dependencies {
            implementation 'org.springframework.boot:spring-boot-starter-actuator'
        }

  * 为需要监控的项目配置springboot监控端口的权限(用来暴露endpoints，由于endpoints包含很多敏感信息，默认只可访问 health 和 info 两个权限)
    
        management:
          endpoints:
            web:
              exposure:
                include: hystrix.stream    

  * 访问 http://\<host>:\<port>/actuator/hystrix.stream 查看是否部署成功 (例： http://localhost:8081/actuator/hystrix.stream ，记得先访问一个其它接口，否之返回的是ping...)
  * 为消费者添加完监控依赖再访问 http://\<host>:\<port>/hystrix ，在title输入要监控的项目名称，点击 Monitor Stream 即可查看项目hystrix运行情况。
* 声明式服务消费 openFeign (hystrix + ribbon 的整合项目) 集成
  * 为消费者项目的build.gradle文件添加依赖

        dependencies {
            //本身须包含 spring-cloud-starter-netflix-eureka-client
            implementation 'org.springframework.cloud:spring-cloud-starter-openfeign'
        }

  * 在项目的Application主类添加注解 @EnableFeignClients

        @EnableFeignClients
        public class FeignConsumerApplication {

  * 在项目需要访问注册中心的service添加注解 @FeignClient("<服务名称,不区分大小写>")并为方法添加          @RequestMapping("<访问服务路径>")注解

        /访问的服务名称
        @FeignClient("cloud-provider")
        public interface ProviderService {
        
            //访问的服务路径
            @RequestMapping("/provider/hello")
            public String hello();
        
        }

  * 为服务消费者项目的application.yml添加相关配置(多配置时service-url以,分隔)

        # 服务名称
        spring:
          application:
            name: feign-consumer
        # eureka注册中心连接方式
        eureka:
          client:
            service-url: 
              defaultZone: http://localhost:8761/eureka,http://localhost:8762/eureka

  * openFeign 开启熔断机制
    * 在 application.yml 配置文件添加配置
        
            feign:
              hystrix:
                enabled: true

    * 在Service的 @FeignClient 注解中添加 fallback 赋值
            
            @FeignClient(name = "cloud-provider", fallback = ProviderFallBack.class)
            public interface ProviderService {
    
    * 自定义的 ProviderFallBack 须实现对应的Service接口，当熔断时调用实现的方法
            
            @Component
            public class ProviderFallBack implements ProviderService {
            
                @Override
                public String hello() {
                    System.out.println("服务不可用时调用");
                    return "fall back.";
                }
            }

  * openFeign获取服务熔断异常信息
    * 在 @FeignClient 注解中赋值属性 fallbackFactory
        
            @FeignClient(name = "cloud-provider", fallbackFactory = ProviderFallBackFactory.class)

    * 编写 ProviderFallBackFactory 类，实现 FallbackFactory 接口并重写相应方法，其中create方法里边的cause即报错信息。
    
            @Component
            public class ProviderFallBackFactory implements FallbackFactory<ProviderService> {
            
                @Override
                public ProviderService create(Throwable cause) {
                    return new ProviderService() {
                        @Override
                        public String hello() {
                            System.out.println("服务不可用时调用 factory.");
                            return cause.getMessage();
                        }
                    };
                }
            }

* zuul 集成(api网关)
  * 新建项目并为项目的build.gradle文件添加依赖

        dependencies {
            implementation 'org.springframework.cloud:spring-cloud-starter-netflix-eureka-client'
            implementation 'org.springframework.cloud:spring-cloud-starter-netflix-zuul'
        }

  * 在项目的Application主类添加注解 @EnableZuulProxy 开启zuul功能

        @EnableZuulProxy
        public class ZuulServerApplication {

  * 为项目的application.yml添加相关配置(多配置时service-url以,分隔)

        # 服务名称
        spring:
          application:
            name: zuul-server
        # eureka注册中心连接方式
        eureka:
          client:
            service-url: 
              defaultZone: http://localhost:8761/eureka,http://localhost:8762/eureka
        # 经典路由配置(与简化配置2选1)
        zuul:
          routes:
            <自定义名字>:
              path: /api-mys/**
              serviceId: cloud-provider
        # 简化路由配置(与经典配置2选1)
        zuul:
          routes:
            cloud-provider: /api-mys/**

      * 其中自定义名字必须对应path和serviceId，意为serviceId指向path,其中serviceId为访问的服务名，path为项目中访问路径，当前规则为匹配所有 /api-mys/** 的请求，只要请求路径包含 /api-mys/ 则会被转发到cloud-provider上
      * 如果什么配置都不写，默认也会帮我们做代理，默认为
            
            zuul:
              routes:
                <服务名>: /<服务名>/**
                      
    * 访问host://\<host>:\<port>/\<配置服务名>/\<配置路径> 测试(例： http://localhost:8085/api-mys/provider/hello )

  * 为zuul添加过滤器
    * 创建 Filter 类并继承ZuulFilter类重写相应方法
    * filterType 方法表示当前filter在哪个生命周期运行 pre->routing->post error任意阶段错误执行
        * pre 表示在路由之前执行
        * post 表示请求完成后执行
        * error 表示错误时执行
        * route 表示环绕请求前后执行
        * static 表示
        * 最后还可以自定义
    * filterOrder 方法表示执行顺序，当filter很多时，根据这个值的大小进行排序，
    * shouldFilter 方法判断当前过滤器是否需要执行
    * run 方法则是具体过滤逻辑
    * 示例代码:
    
            @Component
            public class AuthFilter extends ZuulFilter {
            
                @Override
                public String filterType() {
                    return "pre";
                }
            
                @Override
                public int filterOrder() {
                    return 10;
                }
            
                @Override
                public boolean shouldFilter() {
                    return true;
                }
            
                @Override
                public Object run() throws ZuulException {
                    RequestContext requestContext = RequestContext.getCurrentContext();
                    HttpServletRequest request = requestContext.getRequest();
                    String token = request.getParameter("token");
                    if (token == null) {
                        //是否转发
                        requestContext.setSendZuulResponse(false);
                        requestContext.setResponseStatusCode(401);
                        requestContext.addZuulResponseHeader("content-type", "text/html;charset=utf-8");
                        requestContext.setResponseBody("非法访问");
                    }
                    //目前返回值无意义，可以返回null
                    return null;
                }
            }

  * 忽略路由规则
    * 在 application.yml 文件添加配置
        * 忽略所有服务
            
                zuul:
                    ignored-services: cloud-provider

        * 忽略 某个/某些 路径
        
                zuul:
                    ignored-patterns: /provider/hello

        * 为路由添加前缀
                
                zuul:
                  prefix: /api-mys

  * zuul错误处理
  * 禁用默认错误处理器
    
        zuul:
          SendErrorFilter:
            error:
              disable: true

  * 创建 Filter 类并继承ZuulFilter类重写相应方法，与之前差异仅filterType返回"error"及run中的处理差异，下面给出示例：

        @Component
        public class ErrorFilter extends ZuulFilter {
        
            @Override
            public String filterType() {
                return "error";
            }
        
            @Override
            public int filterOrder() {
                return 10;
            }
        
            @Override
            public boolean shouldFilter() {
                return true;
            }
        
            @Override
            public Object run() throws ZuulException {
                try {
                    RequestContext requestContext = RequestContext.getCurrentContext();
                    ZuulException exception = (ZuulException) requestContext.getThrowable();
                    System.out.println("系统异常拦截：" + exception);
                    HttpServletResponse response = requestContext.getResponse();
                    response.setContentType("application/json; charset=utf8");
                    response.setStatus(exception.nStatusCode);
                    PrintWriter writer = null;
                    try {
                        writer = response.getWriter();
                        writer.println("{\"code\":" + exception.nStatusCode
                                + ",\"message\":\"" + exception.getMessage() + "\"}");
                    } catch (IOException e) {
                        e.printStackTrace();
                    } finally {
                        if (writer != null) {
                            writer.close();
                        }
                    }
                } catch (Exception e) {
                    ReflectionUtils.rethrowRuntimeException(e);
                }
                //目前返回值无意义，可以返回null
                return null;
            }
        }

  * 自定义全局错误页面(与自定义错误处理不兼容)
    * 自定义controller并实现 ErrorController 接口，实现相应方法，下面给出示例：
    
            @RestController
            public class AllErrorController implements ErrorController {
                @Override
                public String getErrorPath() {
                    //访问哪个路径
                    return "/error";
                }
            
                @RequestMapping("/error")
                public Object error() {
                    RequestContext context = RequestContext.getCurrentContext();
                    ZuulException exception = (ZuulException) context.getThrowable();
                    //具体处理看个人需求
                    return exception.nStatusCode + "--" + exception.getMessage();
                }
            
            }
    
* Spring Cloud Config (分布式配置管理系统)
  * Spring Cloud 默认情况使用git存放配置文件，也可使用svn/gitlab等
  * SpringCloud Config 服务搭建
  * Spring Cloud Config 服务端搭建(服务端作为中心)
    * 新建项目并为项目 build.gradle 添加相应依赖

            ext {
                set('springCloudVersion', "Hoxton.SR4")
            }
            
            dependencies {
                implementation 'org.springframework.cloud:spring-cloud-config-server'
            }
            
            dependencyManagement {
                imports {
                    mavenBom "org.springframework.cloud:spring-cloud-dependencies:${springCloudVersion}"
                }
            }

    * 在项目的Application主类添加注解 @EnableConfigServer 开启功能

            @EnableConfigServer
            public class ConfigServerApplication {

    * 在项目的 application.yml 中配置git仓库信息 (这里使用的自己搭建的本地gitlab)
    
            spring:
              cloud:
                config:
                  server:
                    git:
                      uri: http://git.woaizhuangbi.com/makai/springconfigdemo.git
                      search-paths: config-center
                      username: springconfigdemo
                      password: admin1234

    * 此时再启动项目便配置好了配置服务中心 
    * 为了测试可以在项目下创建文件以测试
        * 在根目录创建config-center文件夹
        * 在config-center文件夹下创建application.properties/application-dev.properties/application-test.properties/application-online.yml 文件
        * 在不同文件设置不同配置(支持.yml文件与.properties文件)

              application.properties
                url=http://www.woaizhuangbi.com
              application-test.properties
                url=http://test.woaizhuangbi.com
              application-online.yml
                url:
                    http://online.woaizhuangbi.com
              application-dev.properties
                url=http://dev.woaizhuangbi.com
        * 配置文件大致取名规则
            * /{application}/{profile}[/{label}]
            * /{application}-{profile}.yml
            * /{label}/{application}-{profile}.yml
            * /{application}-{profile}.properties
            * /{label}/{application}-{profile}.properties
    * 访问指定路径获取指定配置 http://\<host>:\<port>/<配置文件前缀>/<配置文件中缀>/<分支> 例： http://localhost:3721/application/online/master
  * Spring Cloud Config 客户端搭建(可以理解为所有需要获取动态配置的项目，可以是所有服务)
    * 为项目 build.gradle 添加相应依赖

            dependencies {
                implementation 'org.springframework.cloud:spring-cloud-starter-config'
            }

    * 创建 bootstrap.yml 文件并配置 name对应配置文件前缀，profile对应中缀，label对应分支，uri即根访问路径
    
            spring:
              application:
                name: config-client
              cloud:
                config:
                  profile: dev
                  label: master
                  uri: http://localhost:3721/

    * 创建controller进行测试,可以直接使用 @Value("${配置名}") 的方式或者 @Environment 的方式进行数据获取，例：
    
            @RestController
            public class ConfigController {
            
                @Value("${url}")
                private String url;
            
                @Autowired
                private Environment environment;
            
                @RequestMapping("/cloud/url")
                public String url() {
                    return environment.getProperty("url");
            //        return url;
                }
            
            }            
    
    * 这样就简单的完成配置的使用

* SpringCloud Config 动态更新
  * 客户端处理
    * 为config客户端项目的 build.gradle 添加相应依赖

            dependencies {
                implementation 'org.springframework.boot:spring-boot-starter-actuator'
            }
    
    * 在服务端项目的 bootstrap.yml 文件中添加暴露监控端点配置及服务注册配置
    
            management:
              endpoint:
                web:
                  exposure:
                    include: "*"
                    
    * 在controller上面添加 @RefreshScope 注解
    
            @RefreshScope
            public class ConfigController {
            
    * 最后发送一个post请求给每个客户端，以动态刷新 curl -X POST "http://<host>:<port>/actuator/refresh" 例如：
        
        curl -X POST "http://localhost:3721/actuator/refresh"

    * 此时再请求发送过post请求的客户端就能获取最新的配置了

  * SpringCloud Config 安全保护
    * 服务端处理
      * 为config服务端项目的 build.gradle 添加相应依赖

            dependencies {
                implementation 'org.springframework.cloud:spring-cloud-starter-security'
            }
    
      * 在服务端项目的 application.yml 文件中配置用户名密码
    
            spring:
              security:
                user:
                  name: springconfigadmin
                  password: admin1234

      * 此时再直接访问服务端的url便需要登录了

    * 客户端处理
      * 在客户端项目的 bootstrap.yml 文件中配置用户名密码
    
            spring:
              cloud:
                config:
                  username: springconfigadmin
                  password: admin1234

* rabbitMQ 安装(docker安装)
  * 拉取镜像
    
        docker pull rabbitmq
        docker pull rabbitmq:management

  * 运行镜像(这里仅单机,端口自定义指定,详细集群等这里不深究)

        docker run -p 4369:4369 -p 5672:5672 -p 15672:15672 -p 25672:25672 --name rabbitmq -d rabbitmq:management
        
  * 默认端口
    * 4369 -- erlang发现口
    * 5672 --client端通信口
    * 15672 -- 管理界面ui端口
    * 25672 -- server间内部通信口
  * 默认访问地址 http://localhost:15672
  * 默认用户名密码：

        guest
        guest

* SpringCloud Bus 集成(消息总线/配合rabbitMQ或者kafka使用/这里使用rabbitMQ)
  * 服务端处理
    * 为config服务端项目的 build.gradle 添加相应依赖

            dependencies {
                implementation 'org.springframework.cloud:spring-cloud-starter-bus-amqp'
            }
    
    * 在服务端项目的 application.yml 文件中配置rabbitmq相关配置及暴露bus刷新配置的端点配置
    
            spring:
              rabbitmq:
                host: 192.168.3.6
                port: 5672
                username: guest
                password: guest
            management:
              endpoints:
                web:
                  exposure:
                    include: 'bus-refresh'

  * 客户端处理
    * 为config客户端项目的 build.gradle 添加相应依赖

            dependencies {
                implementation 'org.springframework.cloud:spring-cloud-starter-bus-amqp'
            }
    
    * 在客户端项目的 application.yml 文件中配置rabbitmq相关配置
    
            spring:
              rabbitmq:
                host: 192.168.3.6
                port: 5672
                username: guest
                password: guest
                
  * 最后发送一个post请求给服务端，以动态刷新 curl -X POST "http://<host>:<port>/actuator/bus-refresh" 例如(这里如果开启了登录则会冲突)：

        curl -X POST "http://localhost:3722/actuator/bus-refresh"

  * 此时再请求所有的客户端就都能获取最新的配置了
  * 动态刷新定点通知(只通知部分客户端)
  * 指定某一个通知： http://<server_host>:<server_port>/actuator/bus-refresh/{destination} 例：
    
        http://localhost:3721/actuator/bus-refresh/config-client:3722

* zookeeper 的集成(注册中心)
  * docker 常用命令
    * docker实例查看

          docker ps -a
        
    * docker复制文件

          docker cp <local path> <applicationID>:<docker path>
        
    * docker登录

          docker exec -i -t <applicationID> /bin/sh
        
  * zookeeper 在 docker 下运行
  * 拉取镜像

        docker pull zookeeper

  * 运行镜像(这里仅单机,端口自定义指定,详细集群等这里不深究)

        docker run -p 8080:8080 -p 2181:2181 --name zookeeper -d zookeeper

  * 访问 http://\<host>:\<port> 测试服务是否能正常访问
* zookeeper 接入
  * 注册服务至zookeeper
    * 服务提供者项目的 build.gradle 添加相应依赖

            dependencies {
                implementation 'org.springframework.cloud:spring-cloud-starter-zookeeper-discovery'
            }
    
    * 服务提供者项目的 application.yml 文件中添加相应配置
    
            spring:
              cloud:
                zookeeper:
                  connect-string: 192.168.3.6:2181

    * 在服务提供者项目的Application主类添加注解 @EnableDiscoveryClient 以使配置文件生效并注册服务

            @EnableDiscoveryClient
            public class CloudProviderApplication {

    * 此时服务提供者的服务便注册至zookeeper了
  * 服务消费者的接入
    * 服务消费者项目的 build.gradle 添加相应依赖
    
            dependencies {
                implementation 'org.springframework.cloud:spring-cloud-starter-zookeeper-discovery'
            }
    
    * 服务消费者项目的 application.yml 文件中添加相应配置
    
            spring:
              cloud:
                zookeeper:
                  connect-string: 192.168.3.6:2181
                  
    * (后面的流程与eureka无异)
    * 再在消费者RestTemplate bean注入的地方加入 @LoadBalanced 注解加入负载均衡功能

            @Bean
            @LoadBalanced
            public RestTemplate restTemplate() {

    * 此时只需要将服务消费者的restTemplate.getForEntity方法里的host改为服务提供者名称即可(例：<restTemplate.getForEntity("http://cloud-provider/provider/hello"...>)
    
            @RestController
            public class ConsumerController {
            
                @Autowired
                RestTemplate restTemplate;
                
                @RequestMapping("/consumer/hello")
                public String hello() {
                    // eureka注册服务调用方式 直接返回数据
                    return restTemplate.getForObject("http://cloud-provider/provider/hello"
                                , String.class);
                }
            }

* gateway 集成(api网关)
  * 基本概念 Predicate->Filter->Route
  * 新建项目并为项目 build.gradle 添加相应依赖

        ext {
            set('springCloudVersion', "Hoxton.SR4")
        }
        
        dependencies {
            implementation 'org.springframework.cloud:spring-cloud-starter-zookeeper-discovery'
        }
        
        dependencyManagement {
            imports {
                mavenBom "org.springframework.cloud:spring-cloud-dependencies:${springCloudVersion}"
            }
        }

  * 在项目的Application主类添加注解 @EnableDiscoveryClient 将服务注册至注册中心

        @EnableDiscoveryClient
        public class GatewayServerApplication {

  * 在项目的 application.yml 中配置相应的路由/服务/断言等(路由可多个)

        spring:
          cloud:
            gateway:
              discovery:
                locator:
                  enabled: true     # 开启从注册中心动态创建路由的功能，利用服务名进行路由
              routes:
                - id: cloud-provider            # 路由的id，没有固定规则但要求唯一，建议配合服务名
        #          uri: http://localhost:8080    # 匹配提供服务的服务根地址
                  uri: lb://cloud-provider    # 匹配后提供服务的路由地址
                  predicates:
                    - Path=/provider/**    # 断言，路径相匹配的进行路由

    * predicates这个配置除了Path还有其它参数
        * After 之前              - After=2020-05-23T15:15:21.234+08:00[Asia/Shanghai] # 指定时间之后可访问
        * Before 之前             - Before=2020-06-23T15:15:21.234+08:00[Asia/Shanghai] # 指定时间之前可以访问 
        * Between 之间            - Between=2020-06-23T15:15:21.234+08:00[Asia/Shanghai],2020-07-23T15:15:21.234+08:00[Asia/Shanghai] # 指定时间之间可以访问
        * Cookie 带Cookie        - Cookie=username,aabb # 带有Cookie且key为username，value为aabb
        * Header 带Header        - Header=X-Request-Id, \d+ # 含X-Request-Id的头且为数字
        * Method 指定请求方式         - Method=GET   # get请求
        * Path 路径匹配             - Path=/provider/** # 请求路径匹配 /provider 开头
        * Query 参数匹配            - Query=username, \d+ # 带username参数且参数值为数字
    * filter 官方有详细介绍
        * 局部过滤器
        * 全局过滤器
        * 自定义过滤器
            * GlobalFilter  全局过滤器
            * Ordered       权重
  * 此时再启动项目便配置好了相关网关服务了
  * 可以访问 http://\<host>:\<port>/<具体路径> 例: http://localhost:8087/provider/hello
* SpringCloud Stream 绑定器(兼容所有MQ(目前仅支持RabbitMQ/Kafka))
  * 常用注解
      * @Input 注解标识输入通道，通过该通道接收到的消息进入应用程序
      * @Output 注解标识输入通道，发布的消息将通过该通道离开应用程序
      * @StreamListener 监听队列，用于消费者队列的消息接收
      * @EnableBinding 指信道channel和exchange绑定在一起
  * 消息提供者者相关
  * 为消息提供者项目的build.gradle文件添加依赖(此处兼容rabbitMQ)

        dependencies {
            implementation 'org.springframework.cloud:spring-cloud-starter-stream-rabbit'
        }

  * 为消息提供者项目的 application.yml 文档添加对应的配置

        spring:
          cloud:
            stream:
              binders:
                defaultRabbit:
                  type: rabbit
                  environment:
                    spring:
                      rabbitmq:
                        host: 192.168.3.6
                        port: 5672
                        username: guest
                        password: guest
              bindings:
                output:
                  destination: studyExchange
                  content-type: application/json
                  binder: defaultRabbit

  * 创建service接口并实现，再在service实现类添加 @EnableBinding 注解并定义消息的推送管道，再注入消息发送管道 MessageChannel，使用时调用 output.send(MessageBuilder.withPayload(<message>).build()) 例：

        @EnableBinding(Source.class) //定义消息的推送管道
        public class MessageProviderImpl implements IMessageProvider {
        
            @Resource
            private MessageChannel output;//消息发送管道
        
            @Override
            public String send() {
                String serial = UUID.randomUUID().toString();
                output.send(MessageBuilder.withPayload(serial).build());
                System.out.println("send:" + serial);
                return serial;
            }
        
        }

  * 使用时直接注入IMessageProvider进行消息操作即可
* 消息消费者相关
  * 为消息消费者项目的build.gradle文件添加依赖(此处兼容rabbitMQ)

        dependencies {
            implementation 'org.springframework.cloud:spring-cloud-starter-stream-rabbit'
        }

  * 为消息消费者项目的 application.yml 文档添加对应的配置

        spring:
          cloud:
            stream:
              binders:
                defaultRabbit:
                  type: rabbit
                  environment:
                    spring:
                      rabbitmq:
                        host: 192.168.3.6
                        port: 5672
                        username: guest
                        password: guest
              bindings:
                input:
                  destination: studyExchange
                  content-type: application/json
                  binder: defaultRabbit

  * 创建Controller并添加 @EnableBinding 注解定义管道类 Sink.class，创建消息方法添加 @StreamListener 注解定义类型，例：

        @Component
        @EnableBinding(Sink.class)
        public class ReceiveMessageListenerController {
        
            @Value("${server.port}")
            private String serverPort;
        
            @StreamListener(Sink.INPUT)
            public void input(Message<String> message) {
                System.out.println(serverPort + " receive message = " + message);
            }
        
        }

  * 使用时 若收到消息即会调用 input(Message<String> message) 方法
  * mq重复消费问题
    * 默认消息生产者会发送给消息给每个用户，即多次消费。当我们的消息只需要被一个用户消费时，需要对多个消息消费者进行共同分组
    * 在多个消费者项目配置文件里添加配置即可，例：
    
          spring:
            cloud:
              stream:
                bindings:
                  input:
                    group: config-client

  * mq持久化
  * 自定义分组的消息消费者掉线后再上线不会丢失错过的消息













