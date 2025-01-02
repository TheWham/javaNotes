### 微信支付接口配置

* **wechat 引入依赖**

  ```xml
  <dependency>
        <groupId>com.github.wechatpay-apiv3</groupId>
        <artifactId>wechatpay-apache-httpclient</artifactId>
        <version>0.4.8</version>
  </dependency>
  ```

* **wechatPay yaml配置**

  ```yaml
  wechat:
      secret: ${sky.wechat.secret}
      appid: ${sky.wechat.appid}
      mchid: ${sky.wechat.mchid}
      mch-serial-no: ${sky.wechat.mchSerialNo}
      we-chat-pay-cert-file-path: ${sky.wechat.we-chat-pay-cert-file-path}
      private-key-file-path: ${sky.wechat.private-key-file-path}
      notify-url: ${sky.wechat.notifyUrl}
      refund-notify-url: ${sky.wechat.refundNotifyUrl}
      api-v3-key: ${sky.wechat.apiV3Key}
  ```

* **WeChatProperties**

  ```java
  package com.sky.properties;
  
  import lombok.Data;
  import org.springframework.beans.factory.annotation.Value;
  import org.springframework.boot.context.properties.ConfigurationProperties;
  import org.springframework.stereotype.Component;
  
  @Component
  @ConfigurationProperties(prefix = "sky.wechat")
  @Data
  public class WeChatProperties {
  
      private String appid; //小程序的appid
      private String secret; //小程序的秘钥
      private String mchid; //商户号
      private String mchSerialNo; //商户API证书的证书序列号
      private String privateKeyFilePath; //商户私钥文件
      private String apiV3Key; //证书解密的密钥
      private String weChatPayCertFilePath; //平台证书
      private String notifyUrl; //支付成功的回调地址
      private String refundNotifyUrl; //退款成功的回调地址
  
  }
  ```

* **WeChatPayUtil**

  ```java
  package com.sky.utils;
  
  import com.alibaba.fastjson.JSON;
  import com.alibaba.fastjson.JSONObject;
  import com.sky.properties.WeChatProperties;
  import com.wechat.pay.contrib.apache.httpclient.WechatPayHttpClientBuilder;
  import com.wechat.pay.contrib.apache.httpclient.util.PemUtil;
  import org.apache.commons.lang.RandomStringUtils;
  import org.apache.http.HttpHeaders;
  import org.apache.http.client.methods.CloseableHttpResponse;
  import org.apache.http.client.methods.HttpGet;
  import org.apache.http.client.methods.HttpPost;
  import org.apache.http.entity.ContentType;
  import org.apache.http.entity.StringEntity;
  import org.apache.http.impl.client.CloseableHttpClient;
  import org.apache.http.util.EntityUtils;
  import org.springframework.beans.factory.annotation.Autowired;
  import org.springframework.stereotype.Component;
  
  import java.io.File;
  import java.io.FileInputStream;
  import java.io.FileNotFoundException;
  import java.math.BigDecimal;
  import java.security.PrivateKey;
  import java.security.Signature;
  import java.security.cert.X509Certificate;
  import java.util.ArrayList;
  import java.util.Arrays;
  import java.util.Base64;
  import java.util.List;
  
  /**
   * 微信支付工具类
   */
  @Component
  public class WeChatPayUtil {
  
      //微信支付下单接口地址
      public static final String JSAPI = "https://api.mch.weixin.qq.com/v3/pay/transactions/jsapi";
  
      //申请退款接口地址
      public static final String REFUNDS = "https://api.mch.weixin.qq.com/v3/refund/domestic/refunds";
  
      @Autowired
      private WeChatProperties weChatProperties;
  
      /**
       * 获取调用微信接口的客户端工具对象
       *
       * @return
       */
      private CloseableHttpClient getClient() {
          PrivateKey merchantPrivateKey = null;
          try {
              //merchantPrivateKey商户API私钥，如何加载商户API私钥请看常见问题
              merchantPrivateKey = PemUtil.loadPrivateKey(new FileInputStream(new File(weChatProperties.getPrivateKeyFilePath())));
              //加载平台证书文件
              X509Certificate x509Certificate = PemUtil.loadCertificate(new FileInputStream(new File(weChatProperties.getWeChatPayCertFilePath())));
              //wechatPayCertificates微信支付平台证书列表。你也可以使用后面章节提到的“定时更新平台证书功能”，而不需要关心平台证书的来龙去脉
              List<X509Certificate> wechatPayCertificates = Arrays.asList(x509Certificate);
  
              WechatPayHttpClientBuilder builder = WechatPayHttpClientBuilder.create()
                      .withMerchant(weChatProperties.getMchid(), weChatProperties.getMchSerialNo(), merchantPrivateKey)
                      .withWechatPay(wechatPayCertificates);
  
              // 通过WechatPayHttpClientBuilder构造的HttpClient，会自动的处理签名和验签
              CloseableHttpClient httpClient = builder.build();
              return httpClient;
          } catch (FileNotFoundException e) {
              e.printStackTrace();
              return null;
          }
      }
  
      /**
       * 发送post方式请求
       *
       * @param url
       * @param body
       * @return
       */
      private String post(String url, String body) throws Exception {
          CloseableHttpClient httpClient = getClient();
  
          HttpPost httpPost = new HttpPost(url);
          httpPost.addHeader(HttpHeaders.ACCEPT, ContentType.APPLICATION_JSON.toString());
          httpPost.addHeader(HttpHeaders.CONTENT_TYPE, ContentType.APPLICATION_JSON.toString());
          httpPost.addHeader("Wechatpay-Serial", weChatProperties.getMchSerialNo());
          httpPost.setEntity(new StringEntity(body, "UTF-8"));
  
          CloseableHttpResponse response = httpClient.execute(httpPost);
          try {
              String bodyAsString = EntityUtils.toString(response.getEntity());
              return bodyAsString;
          } finally {
              httpClient.close();
              response.close();
          }
      }
  
      /**
       * 发送get方式请求
       *
       * @param url
       * @return
       */
      private String get(String url) throws Exception {
          CloseableHttpClient httpClient = getClient();
  
          HttpGet httpGet = new HttpGet(url);
          httpGet.addHeader(HttpHeaders.ACCEPT, ContentType.APPLICATION_JSON.toString());
          httpGet.addHeader(HttpHeaders.CONTENT_TYPE, ContentType.APPLICATION_JSON.toString());
          httpGet.addHeader("Wechatpay-Serial", weChatProperties.getMchSerialNo());
  
          CloseableHttpResponse response = httpClient.execute(httpGet);
          try {
              String bodyAsString = EntityUtils.toString(response.getEntity());
              return bodyAsString;
          } finally {
              httpClient.close();
              response.close();
          }
      }
  
      /**
       * jsapi下单
       *
       * @param orderNum    商户订单号
       * @param total       总金额
       * @param description 商品描述
       * @param openid      微信用户的openid
       * @return
       */
      private String jsapi(String orderNum, BigDecimal total, String description, String openid) throws Exception {
          JSONObject jsonObject = new JSONObject();
          jsonObject.put("appid", weChatProperties.getAppid());
          jsonObject.put("mchid", weChatProperties.getMchid());
          jsonObject.put("description", description);
          jsonObject.put("out_trade_no", orderNum);
          jsonObject.put("notify_url", weChatProperties.getNotifyUrl());
  
          JSONObject amount = new JSONObject();
          amount.put("total", total.multiply(new BigDecimal(100)).setScale(2, BigDecimal.ROUND_HALF_UP).intValue());
          amount.put("currency", "CNY");
  
          jsonObject.put("amount", amount);
  
          JSONObject payer = new JSONObject();
          payer.put("openid", openid);
  
          jsonObject.put("payer", payer);
  
          String body = jsonObject.toJSONString();
          return post(JSAPI, body);
      }
  
      /**
       * 小程序支付
       *
       * @param orderNum    商户订单号
       * @param total       金额，单位 元
       * @param description 商品描述
       * @param openid      微信用户的openid
       * @return
       */
      public JSONObject pay(String orderNum, BigDecimal total, String description, String openid) throws Exception {
          //统一下单，生成预支付交易单
          String bodyAsString = jsapi(orderNum, total, description, openid);
          //解析返回结果
          JSONObject jsonObject = JSON.parseObject(bodyAsString);
          System.out.println(jsonObject);
  
          String prepayId = jsonObject.getString("prepay_id");
          if (prepayId != null) {
              String timeStamp = String.valueOf(System.currentTimeMillis() / 1000);
              String nonceStr = RandomStringUtils.randomNumeric(32);
              ArrayList<Object> list = new ArrayList<>();
              list.add(weChatProperties.getAppid());
              list.add(timeStamp);
              list.add(nonceStr);
              list.add("prepay_id=" + prepayId);
              //二次签名，调起支付需要重新签名
              StringBuilder stringBuilder = new StringBuilder();
              for (Object o : list) {
                  stringBuilder.append(o).append("\n");
              }
              String signMessage = stringBuilder.toString();
              byte[] message = signMessage.getBytes();
  
              Signature signature = Signature.getInstance("SHA256withRSA");
              signature.initSign(PemUtil.loadPrivateKey(new FileInputStream(new File(weChatProperties.getPrivateKeyFilePath()))));
              signature.update(message);
              String packageSign = Base64.getEncoder().encodeToString(signature.sign());
  
              //构造数据给微信小程序，用于调起微信支付
              JSONObject jo = new JSONObject();
              jo.put("timeStamp", timeStamp);
              jo.put("nonceStr", nonceStr);
              jo.put("package", "prepay_id=" + prepayId);
              jo.put("signType", "RSA");
              jo.put("paySign", packageSign);
  
              return jo;
          }
          return jsonObject;
      }
  
      /**
       * 申请退款
       *
       * @param outTradeNo    商户订单号
       * @param outRefundNo   商户退款单号
       * @param refund        退款金额
       * @param total         原订单金额
       * @return
       */
      public String refund(String outTradeNo, String outRefundNo, BigDecimal refund, BigDecimal total) throws Exception {
          JSONObject jsonObject = new JSONObject();
          jsonObject.put("out_trade_no", outTradeNo);
          jsonObject.put("out_refund_no", outRefundNo);
  
          JSONObject amount = new JSONObject();
          amount.put("refund", refund.multiply(new BigDecimal(100)).setScale(2, BigDecimal.ROUND_HALF_UP).intValue());
          amount.put("total", total.multiply(new BigDecimal(100)).setScale(2, BigDecimal.ROUND_HALF_UP).intValue());
          amount.put("currency", "CNY");
  
          jsonObject.put("amount", amount);
          jsonObject.put("notify_url", weChatProperties.getRefundNotifyUrl());
  
          String body = jsonObject.toJSONString();
  
          //调用申请退款接口
          return post(REFUNDS, body);
      }
  }
  ```

* **微信支付回调相关接口**

  ```java
  package com.sky.controller.notify;
  
  import com.alibaba.druid.support.json.JSONUtils;
  import com.alibaba.fastjson.JSON;
  import com.alibaba.fastjson.JSONObject;
  import com.sky.properties.WeChatProperties;
  import com.sky.service.OrderService;
  import com.sky.websocket.WebSocketServer;
  import com.wechat.pay.contrib.apache.httpclient.util.AesUtil;
  import lombok.extern.slf4j.Slf4j;
  import org.apache.http.entity.ContentType;
  import org.springframework.beans.factory.annotation.Autowired;
  import org.springframework.web.bind.annotation.RequestMapping;
  import org.springframework.web.bind.annotation.RestController;
  import javax.servlet.http.HttpServletRequest;
  import javax.servlet.http.HttpServletResponse;
  import java.io.BufferedReader;
  import java.nio.charset.StandardCharsets;
  import java.util.HashMap;
  
  /**
   * 支付回调相关接口
   */
  @RestController
  @RequestMapping("/notify")
  @Slf4j
  public class PayNotifyController {
      @Autowired
      private OrderService orderService;
      @Autowired
      private WeChatProperties weChatProperties;
  
      /**
       * 支付成功回调
       *
       * @param request
       */
      @RequestMapping("/paySuccess")
      public void paySuccessNotify(HttpServletRequest request, HttpServletResponse response) throws Exception {
          //读取数据
          String body = readData(request);
          log.info("支付成功回调：{}", body);
  
          //数据解密
          String plainText = decryptData(body);
          log.info("解密后的文本：{}", plainText);
  
          JSONObject jsonObject = JSON.parseObject(plainText);
          String outTradeNo = jsonObject.getString("out_trade_no");//商户平台订单号
          String transactionId = jsonObject.getString("transaction_id");//微信支付交易号
  
          log.info("商户平台订单号：{}", outTradeNo);
          log.info("微信支付交易号：{}", transactionId);
  
          //业务处理，修改订单状态、来单提醒
          orderService.paySuccess(outTradeNo);
  
          //给微信响应
          responseToWeixin(response);
      }
  
      /**
       * 读取数据
       *
       * @param request
       * @return
       * @throws Exception
       */
      private String readData(HttpServletRequest request) throws Exception {
          BufferedReader reader = request.getReader();
          StringBuilder result = new StringBuilder();
          String line = null;
          while ((line = reader.readLine()) != null) {
              if (result.length() > 0) {
                  result.append("\n");
              }
              result.append(line);
          }
          return result.toString();
      }
  
      /**
       * 数据解密
       *
       * @param body
       * @return
       * @throws Exception
       */
      private String decryptData(String body) throws Exception {
          JSONObject resultObject = JSON.parseObject(body);
          JSONObject resource = resultObject.getJSONObject("resource");
          String ciphertext = resource.getString("ciphertext");
          String nonce = resource.getString("nonce");
          String associatedData = resource.getString("associated_data");
  
          AesUtil aesUtil = new AesUtil(weChatProperties.getApiV3Key().getBytes(StandardCharsets.UTF_8));
          //密文解密
          String plainText = aesUtil.decryptToString(associatedData.getBytes(StandardCharsets.UTF_8),
                  nonce.getBytes(StandardCharsets.UTF_8),
                  ciphertext);
  
          return plainText;
      }
  
      /**
       * 给微信响应
       * @param response
       */
      private void responseToWeixin(HttpServletResponse response) throws Exception{
          response.setStatus(200);
          HashMap<Object, Object> map = new HashMap<>();
          map.put("code", "SUCCESS");
          map.put("message", "SUCCESS");
          response.setHeader("Content-type", ContentType.APPLICATION_JSON.toString());
          response.getOutputStream().write(JSONUtils.toJSONString(map).getBytes(StandardCharsets.UTF_8));
          response.flushBuffer();
      }
  }
  
  ```

* **业务支付方法示例**

  ```java
  public OrderPaymentVO payment(OrdersPaymentDTO ordersPaymentDTO) throws Exception {
              // 当前登录用户id
              Long userId = BaseContext.getCurrentId();
              User user = userMapper.getById(userId);
  
              //调用微信支付接口，生成预支付交易单
              JSONObject jsonObject = weChatPayUtil.pay(
                      ordersPaymentDTO.getOrderNumber(), //商户订单号
                      new BigDecimal(0.01), //支付金额，单位 元
                      "苍穹外卖订单", //商品描述
                      user.getOpenid() //微信用户的openid
              );
  
              if (jsonObject.getString("code") != null && jsonObject.getString("code").equals("ORDERPAID")) {
                  throw new OrderBusinessException("该订单已支付");
              }
  
              OrderPaymentVO vo = jsonObject.toJavaObject(OrderPaymentVO.class);
              vo.setPackageStr(jsonObject.getString("package"));
  
              return vo;
          }
  ```

### 微信登录接口配置

* ```java
  private static final String WX_LOGIN_URL = "https://api.weixin.qq.com/sns/jscode2session";
  
      @Override
      public User wxLogin(String code) {
          String openId = getOpenId(code);
          if (openId == null){
              throw new LoginFailedException(MessageConstant.LOGIN_FAILED);
          }
          User user = userMapper.getByOpenId(openId);
          if (BeanUtil.isEmpty(user)){
             user = User.builder()
                      .openid(openId)
                      .createTime(LocalDateTime.now()).build();
              userMapper.insert(user);
          }
          return user;
      }
  
      private String getOpenId(String code){
          Map<String, String> paramMap = new HashMap<>();
          paramMap.put("appid", weChatProperties.getAppid());
          paramMap.put("secret", weChatProperties.getSecret());
          paramMap.put("js_code", code);
          paramMap.put("grant_type", "authorization_code");
          String json = HttpClientUtil.doGet(WX_LOGIN_URL, paramMap);
          JSONObject jsonObject = JSON.parseObject(json);
          String openId = jsonObject.getString("openid");
          return openId;
      }
  ```

