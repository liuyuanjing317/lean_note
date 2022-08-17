# java_使用redis 进行请求限流

应用场景：在后端处理流程复杂，前端可能会高频点击的情况下，做请求限流来进行系统保护；

本文的应用场景为：前端请求导出excel,出现大数据量的情况下，限制每个ip 用户对于每个地区条件，一分钟内只能请求两次

思路：使用注解在对要限流的方法进行标识，自定义拦截器，在进入这个方法体之前，通过redis 获取缓存的key,该key 设置一个超时时间，该key 的value 为访问次数，每访问一次便+1，达到次数后则不进入方法体，直接返回请求已满，直到key 过期；

代码实现：

0.配置信息：

pom.xml :

<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-data-redis</artifactId>
  <version>2.5.7</version>
</dependency>

 

1, 自定义注解 ======================================================

```
@Retention(RUNTIME)//运行时有效
@Target(ElementType.METHOD)//用在方法上
public @interface AccessLimit {

    //时间范围(单位:秒)
    int seconds();
    //在这个时间范围内最大访问次数
    int maxCount();
}

2.自定义拦截器============================================================
@Slf4j
@Component
public class ExportExcelLimitInterceptor implements HandlerInterceptor {

    @Autowired
    private RedisTemplate redisTemplate;
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        if (handler instanceof HandlerMethod) {
            HandlerMethod hm = (HandlerMethod) handler;
            //设置redisTemplate的序列化方式(必须设置为这种方式，因为要用到incr)
            redisTemplate.setKeySerializer(new StringRedisSerializer());
            redisTemplate.setValueSerializer(new StringRedisSerializer());
            AccessLimit accessLimit = hm.getMethodAnnotation(AccessLimit.class);

            if (null == accessLimit) {
                return true;
            }
            int seconds = accessLimit.seconds();
            int maxCount = accessLimit.maxCount();
            String ipAddr = IpUtil.getIpAddr(request);
            String key = request.getContextPath() + ":" + request.getServletPath() + "_exportExcel"+":" + ipAddr ;
            String countStr = (String) redisTemplate.opsForValue().get(key);
            Integer count = null;
            //如果不是第一次访问，则把访问次数转换为integer类型
            if(countStr != null){
                count = Integer.valueOf(redisTemplate.opsForValue().get(key).toString());
            }
            log.info("count:{}",count);
            //拿到访问次数的过期时间
            Long keySeconds = redisTemplate.getExpire(key);
            if (null == count || -1 == count) {
                redisTemplate.opsForValue().set(key,String.valueOf(1));
                //设置过期时间
                redisTemplate.expire(key,seconds, TimeUnit.SECONDS);
                return true;
            }
            if (count < maxCount) {
                log.info("count:{}",count);
                //访问次数+1
                redisTemplate.opsForValue().increment(key,1);
                //设置剩余过期时间(修改完该key的value值后，对应的过期时间会失效，需重新设置)
                redisTemplate.expire(key,keySeconds, TimeUnit.SECONDS);
                return true;
            }
            if (count >= maxCount) {
//                response 返回 json 请求过于频繁请稍后再试
                this.responseResult(response, new Result(500, "访问次数已超过限制，请稍后重试"));
                return false;
            }
        }
        return true;
    }

    void responseResult(HttpServletResponse response, Result result) throws IOException {
        response.setHeader("Content-Type", "application/json;charset=utf8");
        Writer writer = response.getWriter();
        writer.write(JSONObject.toJSONString(result));
        writer.close();
    }
}

3.添加拦截器到webconfig ===============================================================================
@Configuration
public class WebMvcConfig implements WebMvcConfigurer {



    @Autowired
    private ExportExcelLimitInterceptor excelLimitInterceptor; // 访问限制拦截器

    
    @Override
    public void addInterceptors(InterceptorRegistry registry) {

        registry.addInterceptor(excelLimitInterceptor)
                .addPathPatterns("/qiannan/test");
    }


}

4.添加注解到方法体===================================================================================
@AccessLimit(seconds=30,maxCount=2)
@GetMapping("/qiannan/test")
public Object test(String id) throws IOException {
    return linkFeignClient.detail(id);

}

5. 用到的工具类：
public class IpUtil {
    private IpUtil() { }

    /**
     * 获取登录用户的IP地址
     * @param request
     * @return
     */
    public static String getIpAddr(HttpServletRequest request) {
        String ip = request.getHeader("x-forwarded-for");
        if (ip == null || ip.length() == 0 || "unknown".equalsIgnoreCase(ip)) {
            ip = request.getHeader("Proxy-Client-IP");
        }
        if (ip == null || ip.length() == 0 || "unknown".equalsIgnoreCase(ip)) {
            ip = request.getHeader("WL-Proxy-Client-IP");
        }
        if (ip == null || ip.length() == 0 || "unknown".equalsIgnoreCase(ip)) {
            ip = request.getRemoteAddr();
        }
        if (ip.equals("0:0:0:0:0:0:0:1")) {
            ip = "127.0.0.1";
        }
        if (ip.split(",").length > 1) {
            ip = ip.split(",")[0];
        }
        return ip;
    }

    //ip转化为有序的长整形
    public static long getIp2long(String ip) {
        ip = ip.trim();
        String[] ips = ip.split("\\.");
        long ip2long = 0L;
        for (int i = 0; i < 4; ++i) {
            ip2long = ip2long << 8 | Integer.parseInt(ips[i]);
        }
        return ip2long;
    }
}
```