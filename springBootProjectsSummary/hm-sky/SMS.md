## aliyun短信发送配置

* **sms yaml配置**

  ```yaml
  alibaba:
    ACCESS_KEY_SECRET: xxxxx
    ACCESS_KEY_ID: xxxx
    SIGN_NAME: amani
    TEMPLATE_CODE: SMS_468975746
  ```

* **SMS xml配置**

  ```xml
  <dependency>
      <groupId>com.aliyun</groupId>
      <artifactId>alibabacloud-dysmsapi20170525</artifactId>
      <version>3.0.0</version>
  </dependency>
  ```

* **SMS工具类**

  ```java
  package com.xcs.reggie.utils;
  
  import com.aliyun.auth.credentials.Credential;
  import com.aliyun.auth.credentials.provider.StaticCredentialProvider;
  import com.aliyun.sdk.service.dysmsapi20170525.models.*;
  import com.aliyun.sdk.service.dysmsapi20170525.*;
  import com.google.gson.Gson;
  import com.xcs.reggie.common.R;
  import darabonba.core.client.ClientOverrideConfiguration;
  import lombok.extern.slf4j.Slf4j;
  import org.springframework.beans.factory.annotation.Value;
  import org.springframework.stereotype.Component;
  
  import java.time.Duration;
  import java.util.concurrent.CompletableFuture;
  import java.util.concurrent.ExecutionException;
  
  @Component
  @Slf4j
  public class SMSUtils {
  	@Value("${alibaba.ACCESS_KEY_ID}")
  	private String ALIBABA_CLOUD_ACCESS_KEY_ID;
  	@Value("${alibaba.ACCESS_KEY_SECRET}")
  	private String ALIBABA_CLOUD_ACCESS_KEY_SECRET;
  	@Value("${alibaba.TEMPLATE_CODE}")
  	private String TEMPLATE_CODE;
  	@Value("${alibaba.SIGN_NAME}")
  	private String SIGN_NAME;
  	public boolean sendMessage(String phoneNumbers, String code) throws ExecutionException, InterruptedException {
  
  		log.info(ALIBABA_CLOUD_ACCESS_KEY_ID);
  		log.info(ALIBABA_CLOUD_ACCESS_KEY_SECRET);
  		log.info(TEMPLATE_CODE);
  		log.info(SIGN_NAME);
  		// Configure Credentials authentication information, including ak, secret, token
  		StaticCredentialProvider provider = StaticCredentialProvider.create(Credential.builder()
  				// Please ensure that the environment variables ALIBABA_CLOUD_ACCESS_KEY_ID and ALIBABA_CLOUD_ACCESS_KEY_SECRET are set.
  				.accessKeyId(ALIBABA_CLOUD_ACCESS_KEY_ID)
  				.accessKeySecret(ALIBABA_CLOUD_ACCESS_KEY_SECRET)
  				//.securityToken(System.getenv("ALIBABA_CLOUD_SECURITY_TOKEN")) // use STS token
  				.build());
  
  		// Configure the Client
  		AsyncClient client = AsyncClient.builder()
  				.credentialsProvider(provider)
  				.overrideConfiguration(
  						ClientOverrideConfiguration.create()
  								// Endpoint 请参考 https://api.aliyun.com/product/Dysmsapi
  								.setEndpointOverride("dysmsapi.aliyuncs.com")
  						.setConnectTimeout(Duration.ofSeconds(30))
  				)
  				.build();
  
  		// Parameter settings for API request
  		SendSmsRequest sendSmsRequest = SendSmsRequest.builder()
  				.phoneNumbers(phoneNumbers)
  				.signName(SIGN_NAME)
  				.templateCode(TEMPLATE_CODE)
  				.templateParam("{\"code\":\""+code+"\"}")
  				// Request-level configuration rewrite, can set Http request parameters, etc.
  				// .requestConfiguration(RequestConfiguration.create().setHttpHeaders(new HttpHeaders()))
  				.build();
  
  		// Asynchronously get the return value of the API request
  		CompletableFuture<SendSmsResponse> response = client.sendSms(sendSmsRequest);
  		// Synchronously get the return value of the API request
  		SendSmsResponse resp = response.get();
  		log.info(new Gson().toJson(resp));
  		// Asynchronous processing of return values
          /*response.thenAccept(resp -> {
              System.out.println(new Gson().toJson(resp));
          }).exceptionally(throwable -> { // Handling exceptions
              System.out.println(throwable.getMessage());
              return null;
          });*/
  
  		// Finally, close the client
  		client.close();
  		return true;
  	}
  }
  
  ```

  

  