## 公共字段填充

* **公共填充相关常量**

  ```java
  package com.sky.constant;
  
  /**
   * 公共字段自动填充相关常量
   */
  public class AutoFillConstant {
      /**
       * 实体类中的方法名称
       */
      public static final String SET_CREATE_TIME = "setCreateTime";
      public static final String SET_UPDATE_TIME = "setUpdateTime";
      public static final String SET_CREATE_USER = "setCreateUser";
      public static final String SET_UPDATE_USER = "setUpdateUser";
      public static final String SET_ORDER_TIME = "setOrderTime";
      public static final String SET_CANCEL_TIME = "setCancelTime";
      public static final String SET_CHECKOUT_TIME = "setCheckoutTime";
      public static final String SET_DELIVERY_TIME = "setDeliveryTime";
      public static final String SET_USER_ID = "setUserId";
      public static final String SET_ORDER_STATUS = "setOrderStatus";
  }
  
  ```

* **mybatis 版本**

  1. 设置注解

     ```java
     package com.sky.annotation;
     
     import com.sky.enumeration.OperationType;
     
     import java.lang.annotation.ElementType;
     import java.lang.annotation.Retention;
     import java.lang.annotation.RetentionPolicy;
     import java.lang.annotation.Target;
     
     /**
      * @author Amani
      * @version 2024-09-04
      */
     @Target({ElementType.METHOD})
     @Retention(RetentionPolicy.RUNTIME)
     public @interface AutoFill {
         OperationType value();
     }
     
     ```

  2. 写切面类

     ```java
     package com.sky.aspect;
     
     import com.sky.annotation.AutoFill;
     import com.sky.constant.AutoFillConstant;
     import com.sky.context.BaseContext;
     import com.sky.enumeration.OperationType;
     import org.aspectj.lang.JoinPoint;
     import org.aspectj.lang.Signature;
     import org.aspectj.lang.annotation.Aspect;
     import org.aspectj.lang.annotation.Before;
     import org.aspectj.lang.annotation.Pointcut;
     import org.aspectj.lang.reflect.MethodSignature;
     import org.springframework.stereotype.Component;
     
     import java.lang.reflect.InvocationTargetException;
     import java.lang.reflect.Method;
     import java.time.LocalDateTime;
     
     @Aspect
     @Component
     public class AutoFillAspect {
     
         @Pointcut("execution(* com.sky.mapper.*.*(..)) && @annotation(com.sky.annotation.AutoFill)")
         public void autoFillPointCut(){}
     
         @Before("autoFillPointCut()")
         public void autoFill(JoinPoint joinPoint){
             // 获取对象标签
             MethodSignature signature = (MethodSignature) joinPoint.getSignature();
             AutoFill annotation = signature.getMethod().getAnnotation(AutoFill.class);
             OperationType operationType = annotation.value();
     
             Object[] args = joinPoint.getArgs();
     
             if (args == null || args.length == 0){
                 return;
             }
             Object entity = args[0];
             LocalDateTime now = LocalDateTime.now();
             Long id = BaseContext.getCurrentId();
     
             if (operationType.equals(OperationType.INSERT)){
                 try {
                     Method setCreateTimeMethod = entity.getClass().getDeclaredMethod(AutoFillConstant.SET_CREATE_TIME, LocalDateTime.class);
                     Method setCreateUserMethod = entity.getClass().getDeclaredMethod(AutoFillConstant.SET_UPDATE_USER, Long.class);
                     Method setUpdateTimeMethod = entity.getClass().getDeclaredMethod(AutoFillConstant.SET_UPDATE_TIME, LocalDateTime.class);
                     Method setUpdateUserMethod = entity.getClass().getDeclaredMethod(AutoFillConstant.SET_CREATE_USER, Long.class);
                     setCreateTimeMethod.invoke(entity, now);
                     setCreateUserMethod.invoke(entity, id);
                     setUpdateTimeMethod.invoke(entity, now);
                     setUpdateUserMethod.invoke(entity, id);
                 }catch (Exception ex){
                     ex.printStackTrace();
                 }
             }else if(operationType.equals(OperationType.UPDATE)){
                 try {
                     Method setUpdateTimeMethod = entity.getClass().getDeclaredMethod(AutoFillConstant.SET_UPDATE_TIME, LocalDateTime.class);
                     Method setUpdateUserMethod = entity.getClass().getDeclaredMethod(AutoFillConstant.SET_CREATE_USER, Long.class);
                     setUpdateTimeMethod.invoke(entity, now);
                     setUpdateUserMethod.invoke(entity, id);
                 }catch (Exception ex){
                     ex.printStackTrace();
                 }
             }
         }
     ```

     

* **mybatisPlus 版本**

  * ```java
    import com.baomidou.mybatisplus.core.handlers.MetaObjectHandler;
    import lombok.extern.slf4j.Slf4j;
    import org.apache.ibatis.reflection.MetaObject;
    import org.springframework.stereotype.Component;
    
    import java.time.LocalDateTime;
    
    @Component
    @Slf4j
    public class MyMetaObjectHandler implements MetaObjectHandler {
    
        /**
         * 插入时填充公共字段
         * @param metaObject 元对象
         */
        @Override
        public void insertFill(MetaObject metaObject) {
            log.info("[insert fill...]");
            metaObject.setValue("createTime", LocalDateTime.now());
            metaObject.setValue("updateTime", LocalDateTime.now());
            metaObject.setValue("createUser", BaseContext.getId());
            metaObject.setValue("updateUser", BaseContext.getId());
        }
    
        /**
         * 更新操作时候填充公共字段
         * @param metaObject 元对象
         */
        @Override
        public void updateFill(MetaObject metaObject) {
            log.info("[update fill...]");
            metaObject.setValue("updateTime", LocalDateTime.now());
            metaObject.setValue("updateUser", BaseContext.getId());
        }
    }
    ```

    