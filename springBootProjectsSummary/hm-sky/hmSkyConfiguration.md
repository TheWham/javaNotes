## 苍穹外卖项目配置总结

### JWT配置

* **JWT工具类**

  ```java
  package com.sky.utils;
  
  import io.jsonwebtoken.Claims;
  import io.jsonwebtoken.JwtBuilder;
  import io.jsonwebtoken.Jwts;
  import io.jsonwebtoken.SignatureAlgorithm;
  import java.nio.charset.StandardCharsets;
  import java.util.Date;
  import java.util.Map;
  
  public class JwtUtil {
      /**
       * 生成jwt
       * 使用Hs256算法, 私匙使用固定秘钥
       *
       * @param secretKey jwt秘钥
       * @param ttlMillis jwt过期时间(毫秒)
       * @param claims    设置的信息
       * @return
       */
      public static String createJWT(String secretKey, long ttlMillis, Map<String, Object> claims) {
          // 指定签名的时候使用的签名算法，也就是header那部分
          SignatureAlgorithm signatureAlgorithm = SignatureAlgorithm.HS256;
  
          // 生成JWT的时间
          long expMillis = System.currentTimeMillis() + ttlMillis;
          Date exp = new Date(expMillis);
  
          // 设置jwt的body
          JwtBuilder builder = Jwts.builder()
                  // 如果有私有声明，一定要先设置这个自己创建的私有的声明，这个是给builder的claim赋值，一旦写在标准的声明赋值之后，就是覆盖了那些标准的声明的
                  .setClaims(claims)
                  // 设置签名使用的签名算法和签名使用的秘钥
                  .signWith(signatureAlgorithm, secretKey.getBytes(StandardCharsets.UTF_8))
                  // 设置过期时间
                  .setExpiration(exp);
  
          return builder.compact();
      }
  
      /**
       * Token解密
       *
       * @param secretKey jwt秘钥 此秘钥一定要保留好在服务端, 不能暴露出去, 否则sign就可以被伪造, 如果对接多个客户端建议改造成多个
       * @param token     加密后的token
       * @return
       */
      public static Claims parseJWT(String secretKey, String token) {
          // 得到DefaultJwtParser
          Claims claims = Jwts.parser()
                  // 设置签名的秘钥
                  .setSigningKey(secretKey.getBytes(StandardCharsets.UTF_8))
                  // 设置需要解析的jwt
                  .parseClaimsJws(token).getBody();
          return claims;
      }
  
  }
  ```

* **jwt类配置**

  ```java
  package com.sky.properties;
  
  import lombok.Data;
  import org.springframework.boot.context.properties.ConfigurationProperties;
  import org.springframework.stereotype.Component;
  
  @Component
  @ConfigurationProperties(prefix = "sky.jwt")
  @Data
  public class JwtProperties {
  
      /**
       * 管理端员工生成jwt令牌相关配置
       */
      private String adminSecretKey;
      private long adminTtl;
      private String adminTokenName;
  
      /**
       * 用户端微信用户生成jwt令牌相关配置
       */
      private String userSecretKey;
      private long userTtl;
      private String userTokenName;
  
  }
  ```

* **jwt yml配置**

  ```yaml
  sky:
    jwt:
      # 设置jwt签名加密时使用的秘钥
      admin-secret-key: itcast
      # 设置jwt过期时间
      admin-ttl: 7200000
      # 设置前端传递过来的令牌名称
      admin-token-name: token
      # 设置用户端jwt签名加密时使用的密钥
      user-Secret-Key: itheima
      # 设置用户端jwt过期时间
      user-Ttl: 7200000
      # 设置用户端传递过来的令牌名称
      user-token-name: authentication
  ```

* **JWT token Interceptor 配置**

  ```java
  package com.sky.interceptor;
  
  import com.sky.constant.JwtClaimsConstant;
  import com.sky.context.BaseContext;
  import com.sky.properties.JwtProperties;
  import com.sky.utils.JwtUtil;
  import io.jsonwebtoken.Claims;
  import lombok.extern.slf4j.Slf4j;
  import org.springframework.beans.factory.annotation.Autowired;
  import org.springframework.stereotype.Component;
  import org.springframework.web.method.HandlerMethod;
  import org.springframework.web.servlet.HandlerInterceptor;
  import javax.servlet.http.HttpServletRequest;
  import javax.servlet.http.HttpServletResponse;
  
  /**
   * jwt令牌校验的拦截器
   */
  @Component
  @Slf4j
  public class JwtTokenAdminInterceptor implements HandlerInterceptor {
  
      @Autowired
      private JwtProperties jwtProperties;
  
      /**
       * 校验jwt
       *
       * @param request
       * @param response
       * @param handler
       * @return
       * @throws Exception
       */
      @Override
      public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
          //判断当前拦截到的是Controller的方法还是其他资源
          if (!(handler instanceof HandlerMethod)) {
              //当前拦截到的不是动态方法，直接放行
              return true;
          }
  
          //1、从请求头中获取令牌
          String token = request.getHeader(jwtProperties.getAdminTokenName());
  
          //2、校验令牌
          try {
              log.info("jwt校验:{}", token);
              Claims claims = JwtUtil.parseJWT(jwtProperties.getAdminSecretKey(), token);
              Long empId = Long.valueOf(claims.get(JwtClaimsConstant.EMP_ID).toString());
              log.info("当前员工id：{}", empId);
              BaseContext.setCurrentId(empId);
              //3、通过，放行
              return true;
          } catch (Exception ex) {
              //4、不通过，响应401状态码
              response.setStatus(401);
              return false;
          }
      }
  }
  ```

### WebMvcConfiguration

```java
package com.sky.config;

import com.sky.interceptor.JwtTokenAdminInterceptor;
import com.sky.interceptor.JwtTokenUserInterceptor;
import com.sky.json.JacksonObjectMapper;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.converter.HttpMessageConverter;
import org.springframework.http.converter.json.MappingJackson2HttpMessageConverter;
import org.springframework.web.servlet.config.annotation.InterceptorRegistry;
import org.springframework.web.servlet.config.annotation.ResourceHandlerRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurationSupport;
import springfox.documentation.builders.ApiInfoBuilder;
import springfox.documentation.builders.PathSelectors;
import springfox.documentation.builders.RequestHandlerSelectors;
import springfox.documentation.service.ApiInfo;
import springfox.documentation.spi.DocumentationType;
import springfox.documentation.spring.web.plugins.Docket;

import java.util.List;

/**
 * 配置类，注册web层相关组件
 */
@Configuration
@Slf4j
public class WebMvcConfiguration extends WebMvcConfigurationSupport {

    @Autowired
    private JwtTokenAdminInterceptor jwtTokenAdminInterceptor;
    @Autowired
    private JwtTokenUserInterceptor jwtTokenUserInterceptor;
    /**
     * 注册自定义拦截器
     *
     * @param registry
     */
    @Override
    protected void addInterceptors(InterceptorRegistry registry) {
        log.info("开始注册自定义拦截器...");
        registry.addInterceptor(jwtTokenAdminInterceptor)
                .addPathPatterns("/admin/**")
                .excludePathPatterns("/admin/employee/login");
        registry.addInterceptor(jwtTokenUserInterceptor)
                .addPathPatterns("/user/**")
                .excludePathPatterns("/user/user/login");
    }

    /**
     * 通过knife4j生成接口文档
     * @return
     */
    @Bean
    public Docket docket1() {
        ApiInfo apiInfo = new ApiInfoBuilder()
                .title("苍穹外卖项目接口文档")
                .version("2.0")
                .description("苍穹外卖项目接口文档")
                .build();
        Docket docket = new Docket(DocumentationType.SWAGGER_2)
                .groupName("管理员接口端")
                .apiInfo(apiInfo)
                .select()
                .apis(RequestHandlerSelectors.basePackage("com.sky.controller.admin"))
                .paths(PathSelectors.any())
                .build();
        return docket;
    }
    @Bean
    public Docket docket2() {
        ApiInfo apiInfo = new ApiInfoBuilder()
                .title("苍穹外卖项目接口文档")
                .version("2.0")
                .description("苍穹外卖项目接口文档")
                .build();
        Docket docket = new Docket(DocumentationType.SWAGGER_2)
                .groupName("用户接口端")
                .apiInfo(apiInfo)
                .select()
                .apis(RequestHandlerSelectors.basePackage("com.sky.controller.user"))
                .paths(PathSelectors.any())
                .build();
        return docket;
    }

    /**
     * 设置静态资源映射
     * @param registry
     */
    @Override
    protected void addResourceHandlers(ResourceHandlerRegistry registry) {
        registry.addResourceHandler("/doc.html").addResourceLocations("classpath:/META-INF/resources/");
        registry.addResourceHandler("/webjars/**").addResourceLocations("classpath:/META-INF/resources/webjars/");
    }

    @Override
    protected void extendMessageConverters(List<HttpMessageConverter<?>> converters) {
        MappingJackson2HttpMessageConverter messageConverter = new MappingJackson2HttpMessageConverter();
        messageConverter.setObjectMapper(new JacksonObjectMapper());
        converters.add(0,messageConverter);
    }
}

```

