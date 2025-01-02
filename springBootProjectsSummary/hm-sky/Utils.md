### JSON序列化器

```java

package com.xcs.reggie.common;

import com.fasterxml.jackson.databind.DeserializationFeature;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.module.SimpleModule;
import com.fasterxml.jackson.databind.ser.std.ToStringSerializer;
import com.fasterxml.jackson.datatype.jsr310.deser.LocalDateDeserializer;
import com.fasterxml.jackson.datatype.jsr310.deser.LocalDateTimeDeserializer;
import com.fasterxml.jackson.datatype.jsr310.deser.LocalTimeDeserializer;
import com.fasterxml.jackson.datatype.jsr310.ser.LocalDateSerializer;
import com.fasterxml.jackson.datatype.jsr310.ser.LocalDateTimeSerializer;
import com.fasterxml.jackson.datatype.jsr310.ser.LocalTimeSerializer;
import java.math.BigInteger;
import java.time.LocalDate;
import java.time.LocalDateTime;
import java.time.LocalTime;
import java.time.format.DateTimeFormatter;
import static com.fasterxml.jackson.databind.DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES;


/**
 * 对象映射器:基于jackson将Java对象转为json，或者将json转为Java对象
 * 将JSON解析为Java对象的过程称为 [从JSON反序列化Java对象]
 * 从Java对象生成JSON的过程称为 [序列化Java对象到JSON]
 */

public class JacksonObjectMapper extends ObjectMapper {

    public static final String DEFAULT_DATE_FORMAT = "yyyy-MM-dd";
    public static final String DEFAULT_DATE_TIME_FORMAT = "yyyy-MM-dd HH:mm:ss";
    public static final String DEFAULT_TIME_FORMAT = "HH:mm:ss";

    public JacksonObjectMapper() {
        super();
        //收到未知属性时不报异常
        this.configure(FAIL_ON_UNKNOWN_PROPERTIES, false);

        //反序列化时，属性不存在的兼容处理
        this.getDeserializationConfig().withoutFeatures(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES);


        SimpleModule simpleModule = new SimpleModule()
                .addDeserializer(LocalDateTime.class, new LocalDateTimeDeserializer(DateTimeFormatter.ofPattern(DEFAULT_DATE_TIME_FORMAT)))
                .addDeserializer(LocalDate.class, new LocalDateDeserializer(DateTimeFormatter.ofPattern(DEFAULT_DATE_FORMAT)))
                .addDeserializer(LocalTime.class, new LocalTimeDeserializer(DateTimeFormatter.ofPattern(DEFAULT_TIME_FORMAT)))

                .addSerializer(BigInteger.class, ToStringSerializer.instance)
                .addSerializer(Long.class, ToStringSerializer.instance)
                .addSerializer(LocalDateTime.class, new LocalDateTimeSerializer(DateTimeFormatter.ofPattern(DEFAULT_DATE_TIME_FORMAT)))
                .addSerializer(LocalDate.class, new LocalDateSerializer(DateTimeFormatter.ofPattern(DEFAULT_DATE_FORMAT)))
                .addSerializer(LocalTime.class, new LocalTimeSerializer(DateTimeFormatter.ofPattern(DEFAULT_TIME_FORMAT)));

        //注册功能模块 例如，可以添加自定义序列化器和反序列化器
        this.registerModule(simpleModule);
    }
}
 
```

### 随机生成验证码工具类

```java
package com.xcs.reggie.utils;

import java.util.Random;

/**
 * 随机生成验证码工具类
 */
public class ValidateCodeUtils {
    /**
     * 随机生成验证码
     * @param length 长度为4位或者6位
     * @return
     */
    public static Integer generateValidateCode(int length){
        Integer code =null;
        if(length == 4){
            code = new Random().nextInt(9999);
            //生成随机数，最大为9999
            if(code < 1000){
                code = code + 1000;
                //保证随机数为4位数字
            }
        }else if(length == 6){
            code = new Random().nextInt(999999);
            //生成随机数，最大为999999
            if(code < 100000){
                code = code + 100000;
                //保证随机数为6位数字
            }
        }else{
            throw new RuntimeException("只能生成4位或6位数字验证码");
        }
        return code;
    }

    /**
     * 随机生成指定长度字符串验证码
     * @param length 长度
     * @return
     */
    public static String generateValidateCode4String(int length){
        Random rdm = new Random();
        String hash1 = Integer.toHexString(rdm.nextInt());
        String capstr = hash1.substring(0, length);
        return capstr;
    }
}

```

### Result范式

```java
package com.xcs.reggie.common;

import lombok.Data;

import java.io.Serializable;
import java.util.HashMap;
import java.util.Map;

@Data
public class R<T> implements Serializable {
    private static final long serialVersionUID = 1L;
    private String msg;
    private Integer code;
    private T data;
    private Map map = new HashMap();

    public static <T> R<T> success(T data){
      R<T> r = new R<>();
      r.data = data;
      r.code = 1;
      return  r;
    }

    public static <T> R<T> error(String msg){
        R<T> r = new R<>();
        r.code = 0;
        r.msg = msg;
        return r;
    }
    public R<T> add(String key, Object value){
        this.map.put(key, value);
        return this;
    }
}
```

### 公共异常处理

```java
package com.sky.handler;

import com.sky.entity.Employee;
import com.sky.exception.BaseException;
import com.sky.result.Result;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.ControllerAdvice;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.bind.annotation.RestControllerAdvice;

import java.sql.SQLIntegrityConstraintViolationException;

/**
 * 全局异常处理器，处理项目中抛出的业务异常
 */
@RestControllerAdvice
@Slf4j
@ControllerAdvice(annotations = {RestController.class, Controller.class})
public class GlobalExceptionHandler {

    /**
     * 捕获业务异常
     * @param ex
     * @return
     */
    @ExceptionHandler
    public Result exceptionHandler(BaseException ex){
        log.error("异常信息：{}", ex.getMessage());
        return Result.error(ex.getMessage());
    }

    @ExceptionHandler(SQLIntegrityConstraintViolationException.class)
    public Result<Employee> controllerExceptionHandler(SQLIntegrityConstraintViolationException ex){
        log.info("异常{}", ex.getMessage());
        if (ex.getMessage().contains("Duplicate entry"))
        {
            String []splits = ex.getMessage().split(" ");
            String msg = splits[2] + "已存在";
            log.error(msg);
            return Result.error(msg);
        }
        return Result.error("未知错误");
    }

}
```

### HttpClient 类

* **引入依赖**

  ```xml
  <dependency>
      <groupId>org.apache.poi</groupId>
      <artifactId>poi</artifactId>
      <version>3.16</version>
  </dependency>
  <dependency>
      <groupId>org.apache.poi</groupId>
      <artifactId>poi-ooxml</artifactId>
      <version>3.16</version>
  </dependency>
  ```

  

* **实例类**

  ```java
  package com.sky.utils;
  
  import com.alibaba.fastjson.JSONObject;
  import org.apache.http.NameValuePair;
  import org.apache.http.client.config.RequestConfig;
  import org.apache.http.client.entity.UrlEncodedFormEntity;
  import org.apache.http.client.methods.CloseableHttpResponse;
  import org.apache.http.client.methods.HttpGet;
  import org.apache.http.client.methods.HttpPost;
  import org.apache.http.client.utils.URIBuilder;
  import org.apache.http.entity.StringEntity;
  import org.apache.http.impl.client.CloseableHttpClient;
  import org.apache.http.impl.client.HttpClients;
  import org.apache.http.message.BasicNameValuePair;
  import org.apache.http.util.EntityUtils;
  
  import java.io.IOException;
  import java.net.URI;
  import java.util.ArrayList;
  import java.util.List;
  import java.util.Map;
  
  /**
   * Http工具类
   */
  public class HttpClientUtil {
  
      static final int TIMEOUT_MSEC = 5 * 1000;
  
      /**
       * 发送GET方式请求
       * @param url
       * @param paramMap
       * @return
       */
      public static String doGet(String url,Map<String,String> paramMap){
          // 创建Httpclient对象
          CloseableHttpClient httpClient = HttpClients.createDefault();
  
          String result = "";
          CloseableHttpResponse response = null;
  
          try{
              URIBuilder builder = new URIBuilder(url);
              if(paramMap != null){
                  for (String key : paramMap.keySet()) {
                      builder.addParameter(key,paramMap.get(key));
                  }
              }
              URI uri = builder.build();
  
              //创建GET请求
              HttpGet httpGet = new HttpGet(uri);
  
              //发送请求
              response = httpClient.execute(httpGet);
  
              //判断响应状态
              if(response.getStatusLine().getStatusCode() == 200){
                  result = EntityUtils.toString(response.getEntity(),"UTF-8");
              }
          }catch (Exception e){
              e.printStackTrace();
          }finally {
              try {
                  response.close();
                  httpClient.close();
              } catch (IOException e) {
                  e.printStackTrace();
              }
          }
  
          return result;
      }
  
      /**
       * 发送POST方式请求
       * @param url
       * @param paramMap
       * @return
       * @throws IOException
       */
      public static String doPost(String url, Map<String, String> paramMap) throws IOException {
          // 创建Httpclient对象
          CloseableHttpClient httpClient = HttpClients.createDefault();
          CloseableHttpResponse response = null;
          String resultString = "";
  
          try {
              // 创建Http Post请求
              HttpPost httpPost = new HttpPost(url);
  
              // 创建参数列表
              if (paramMap != null) {
                  List<NameValuePair> paramList = new ArrayList();
                  for (Map.Entry<String, String> param : paramMap.entrySet()) {
                      paramList.add(new BasicNameValuePair(param.getKey(), param.getValue()));
                  }
                  // 模拟表单
                  UrlEncodedFormEntity entity = new UrlEncodedFormEntity(paramList);
                  httpPost.setEntity(entity);
              }
  
              httpPost.setConfig(builderRequestConfig());
  
              // 执行http请求
              response = httpClient.execute(httpPost);
  
              resultString = EntityUtils.toString(response.getEntity(), "UTF-8");
          } catch (Exception e) {
              throw e;
          } finally {
              try {
                  response.close();
              } catch (IOException e) {
                  e.printStackTrace();
              }
          }
  
          return resultString;
      }
  
      /**
       * 发送POST方式请求
       * @param url
       * @param paramMap
       * @return
       * @throws IOException
       */
      public static String doPost4Json(String url, Map<String, String> paramMap) throws IOException {
          // 创建Httpclient对象
          CloseableHttpClient httpClient = HttpClients.createDefault();
          CloseableHttpResponse response = null;
          String resultString = "";
  
          try {
              // 创建Http Post请求
              HttpPost httpPost = new HttpPost(url);
  
              if (paramMap != null) {
                  //构造json格式数据
                  JSONObject jsonObject = new JSONObject();
                  for (Map.Entry<String, String> param : paramMap.entrySet()) {
                      jsonObject.put(param.getKey(),param.getValue());
                  }
                  StringEntity entity = new StringEntity(jsonObject.toString(),"utf-8");
                  //设置请求编码
                  entity.setContentEncoding("utf-8");
                  //设置数据类型
                  entity.setContentType("application/json");
                  httpPost.setEntity(entity);
              }
  
              httpPost.setConfig(builderRequestConfig());
  
              // 执行http请求
              response = httpClient.execute(httpPost);
  
              resultString = EntityUtils.toString(response.getEntity(), "UTF-8");
          } catch (Exception e) {
              throw e;
          } finally {
              try {
                  response.close();
              } catch (IOException e) {
                  e.printStackTrace();
              }
          }
  
          return resultString;
      }
      private static RequestConfig builderRequestConfig() {
          return RequestConfig.custom()
                  .setConnectTimeout(TIMEOUT_MSEC)
                  .setConnectionRequestTimeout(TIMEOUT_MSEC)
                  .setSocketTimeout(TIMEOUT_MSEC).build();
      }
  
  }
  ```

