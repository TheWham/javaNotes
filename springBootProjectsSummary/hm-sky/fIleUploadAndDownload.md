## 文件上传下载方法汇总

1. **直接上传到服务器文件中**

   * **上传路径**

   ```yaml
   img:
       path: D:/img/   #以windows为例
   # windows 和 linux 两种路径
   ```

   ```java
   package com.xcs.reggie.controller;
   
   import com.alibaba.fastjson.JSON;
   import com.xcs.reggie.common.R;
   import lombok.extern.slf4j.Slf4j;
   import org.springframework.beans.factory.annotation.Value;
   import org.springframework.web.bind.annotation.GetMapping;
   import org.springframework.web.bind.annotation.PostMapping;
   import org.springframework.web.bind.annotation.RequestMapping;
   import org.springframework.web.bind.annotation.RestController;
   import org.springframework.web.multipart.MultipartFile;
   
   import javax.servlet.ServletOutputStream;
   import javax.servlet.http.HttpServletResponse;
   import java.io.File;
   import java.io.FileInputStream;
   import java.io.IOException;
   import java.util.UUID;
   
   @RestController
   @Slf4j
   @RequestMapping("/common")
   public class FileController {
   
       @Value("${img.path}")
       private String imgPath;
   
       //解决冗余图片问题(待解决)
   
       /**
        * 文件的上传
        * @param file
        * @return
        */
       @PostMapping("/upload")
       public R<String> upload(MultipartFile file){
           try {
               String filename = file.getOriginalFilename();
               String suffix = filename.substring(filename.lastIndexOf('.') + 1).toLowerCase();
   
               // 检查文件类型是否支持
               if (!isSupportedImageType(suffix)) {
                   return R.error("不支持的文件类型：" + suffix);
               }
   
               String id = UUID.randomUUID().toString();
               String lastFileName = id + "." + suffix;
               log.info("path {}", imgPath  + lastFileName);
   
               File file1 = new File(imgPath);
               if (!file1.exists()) {
                   file1.mkdirs(); // 确保创建整个目录路径
               }
   
               file.transferTo(new File(imgPath +  lastFileName));
               log.info("path {}", imgPath + lastFileName);
               return R.success(lastFileName);
           } catch (IOException e) {
               log.error("上传文件失败", e);
               return R.error("上传文件失败");
           }
       }
   
       /**
        * 文件的下载
        * @param name
        * @param response
        */
       @GetMapping("/download")
       public void download(String name, HttpServletResponse response) {
           try {
               log.info("name {}", name);
               File file = new File(imgPath + name);
   
               if (!file.exists()) {
                   response.setStatus(HttpServletResponse.SC_NOT_FOUND);
                   response.getWriter().write(JSON.toJSONString(R.error("文件不存在")));
                   return;
               }
   
               FileInputStream inputStream = new FileInputStream(file);
   
               // 动态设置 Content-Type
               String contentType = determineContentType(name);
               response.setContentType(contentType);
   
               ServletOutputStream outputStream = response.getOutputStream();
   
               byte[] bytes = new byte[1024];
               int len;
               while ((len = inputStream.read(bytes)) != -1) {
                   outputStream.write(bytes, 0, len);
               }
   
               inputStream.close();
               outputStream.close();
           } catch (IOException e) {
               log.error("文件下载失败", e);
               try {
                   response.setStatus(HttpServletResponse.SC_INTERNAL_SERVER_ERROR);
                   response.getWriter().write(JSON.toJSONString(R.error("文件下载失败")));
               } catch (IOException ioException) {
                   log.error("响应错误时出现异常", ioException);
               }
           }
       }
   
       private boolean isSupportedImageType(String extension) {
           switch (extension) {
               case "jpg":
               case "jpeg":
               case "png":
               case "gif":
               case "bmp":
                   return true;
               default:
                   return false;
           }
       }
   
       private String determineContentType(String filename) {
           String extension = filename.substring(filename.lastIndexOf('.') + 1).toLowerCase();
           switch (extension) {
               case "jpg":
               case "jpeg":
                   return "image/jpeg";
               case "png":
                   return "image/png";
               case "gif":
                   return "image/gif";
               case "bmp":
                   return "image/bmp";
               default:
                   return "application/octet-stream";
                   // 默认二进制流
           }
       }
   }
   
   ```

   

2. **上传到aliyunOSS中存储**

   ```java
   package com.sky.controller.admin;
   
   
   import cn.hutool.core.lang.UUID;
   import com.sky.constant.MessageConstant;
   import com.sky.result.Result;
   import com.sky.utils.AliOssUtil;
   import io.swagger.annotations.Api;
   import lombok.extern.slf4j.Slf4j;
   import org.springframework.beans.factory.annotation.Autowired;
   import org.springframework.web.bind.annotation.PostMapping;
   import org.springframework.web.bind.annotation.RequestMapping;
   import org.springframework.web.bind.annotation.RestController;
   import org.springframework.web.multipart.MultipartFile;
   
   import java.io.IOException;
   
   @RestController
   @Api(tags = "文件上传接口")
   @RequestMapping("/admin/common")
   @Slf4j
   public class UploadController {
   
       @Autowired
       private AliOssUtil aliOssUtil;
   
       @PostMapping("/upload")
       public Result<String> imgUpload(MultipartFile file){
           try {
               String filename = file.getOriginalFilename();
               String suffix = filename.substring(filename.lastIndexOf("."));
               String prefix = UUID.randomUUID().toString(false);
               String fileName = prefix + suffix;
               String fileUrl = aliOssUtil.upload(file.getBytes(), fileName);
               return Result.success(fileUrl);
           } catch (IOException e) {
               log.error(MessageConstant.UPLOAD_FAILED);
               throw new RuntimeException(e);
           }
       }
   }
   // 没有下载接口应为上传到oss之后默认返回原图片的地址
   ```

   * **OSS工具类**

     ```java
     package com.sky.utils;
     
     import com.aliyun.oss.ClientException;
     import com.aliyun.oss.OSS;
     import com.aliyun.oss.OSSClientBuilder;
     import com.aliyun.oss.OSSException;
     import lombok.AllArgsConstructor;
     import lombok.Data;
     import lombok.extern.slf4j.Slf4j;
     import java.io.ByteArrayInputStream;
     
     @Data
     @AllArgsConstructor
     @Slf4j
     public class AliOssUtil {
     
         private String endpoint;
         private String accessKeyId;
         private String accessKeySecret;
         private String bucketName;
     
         /**
          * 文件上传
          *
          * @param bytes
          * @param objectName
          * @return
          */
         public String upload(byte[] bytes, String objectName) {
     
             // 创建OSSClient实例。
             OSS ossClient = new OSSClientBuilder().build(endpoint, accessKeyId, accessKeySecret);
     
             try {
                 // 创建PutObject请求。
                 ossClient.putObject(bucketName, objectName, new ByteArrayInputStream(bytes));
             } catch (OSSException oe) {
                 System.out.println("Caught an OSSException, which means your request made it to OSS, "
                         + "but was rejected with an error response for some reason.");
                 System.out.println("Error Message:" + oe.getErrorMessage());
                 System.out.println("Error Code:" + oe.getErrorCode());
                 System.out.println("Request ID:" + oe.getRequestId());
                 System.out.println("Host ID:" + oe.getHostId());
             } catch (ClientException ce) {
                 System.out.println("Caught an ClientException, which means the client encountered "
                         + "a serious internal problem while trying to communicate with OSS, "
                         + "such as not being able to access the network.");
                 System.out.println("Error Message:" + ce.getMessage());
             } finally {
                 if (ossClient != null) {
                     ossClient.shutdown();
                 }
             }
     
             //文件访问路径规则 https://BucketName.Endpoint/ObjectName
             StringBuilder stringBuilder = new StringBuilder("https://");
             stringBuilder
                     .append(bucketName)
                     .append(".")
                     .append(endpoint)
                     .append("/")
                     .append(objectName);
     
             log.info("文件上传到:{}", stringBuilder.toString());
     
             return stringBuilder.toString();
         }
     }
     ```

   * **OSS配置类**

     ```java
     package com.sky.properties;
     
     import lombok.Data;
     import org.springframework.boot.context.properties.ConfigurationProperties;
     import org.springframework.stereotype.Component;
     
     @Component
     @ConfigurationProperties(prefix = "sky.alioss")
     @Data
     public class AliOssProperties {
     
         private String endpoint;
         private String accessKeyId;
         private String accessKeySecret;
         private String bucketName;
     
     }
     ```

   * **OSS yaml**

     ```yaml
       alioss:
         access-key-id: ${sky.alioss.access-key-id}
         access-key-secret: ${sky.alioss.access-key-secret}
         # 上传目录
         bucket-name: ${sky.alioss.bucket-name}
         # 上传节点 例如: oss-cn-beijing.aliyuncs.com
         endpoint: ${sky.alioss.endpoint}
     ```

     

   * **OSS配置加入spring管理自动注入**

     ```java
     package com.sky.config;
     
     import com.sky.properties.AliOssProperties;
     import com.sky.utils.AliOssUtil;
     import lombok.extern.slf4j.Slf4j;
     import org.springframework.beans.factory.annotation.Autowired;
     import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean;
     import org.springframework.context.annotation.Bean;
     import org.springframework.context.annotation.Configuration;
     
     @Configuration
     @Slf4j
     public class OssConfiguration {
     
     
         @Bean
         @ConditionalOnMissingBean
         public AliOssUtil aliOssUtil(AliOssProperties aliOssProperties){
             log.info("开始上传aliyun文件工具类对象 {}", aliOssProperties);
             return new AliOssUtil(aliOssProperties.getEndpoint(),
                     aliOssProperties.getAccessKeyId(),
                     aliOssProperties.getAccessKeySecret(),
                     aliOssProperties.getBucketName());
         }
     }
     
     ```

   * 引入依赖

     ```xml
     <dependency>
          <groupId>com.aliyun.oss</groupId>
          <artifactId>aliyun-sdk-oss</artifactId>
          <version>3.10.2</version>
      </dependency>
     ```

     

