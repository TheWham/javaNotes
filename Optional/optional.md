## Optional 

* **创建方式**

  ```java
  String value = "xcs";
  // optional类的两种静态创建方式 of, ofNullable
  // of 中的参数必须不为null 否则就会出现空指针异常
  Optional<Object> optional = Optional.of(value);
  Optional<String> optional = Optional.ofNullable(value);
  ```

* **判断方式**

  ```java
  Optional<Object> optional = Optional.of(value);
  // 判断是否有值 存在为ture 反之为false
  optional.isPresent();
  // 判断是否为空 空为true 反正false
  optional.isEmpty();
  ```

* **实际运用**

  ```java
  // 创建user对象
  @Data
  public class User {
      private String name;
      private String fullName;
  }
  // 创建userResposity模拟dao层
  public class UserResposity {
      public Optional<User> optionalFindName(String name) {
          if (name.equals("xcs")) {
              //存在返回Optional对象
              return Optional.of(new User("xcs", "xcsnb"));
          } else {
              //不存在返回空的Optional对象
              return Optional.empty();
          }
      }
  }
  
  Optional<User> optionalFindName = new UserResposity().optionalFindName("xcs1");
  // orElse 里面传的参数是默认值, 不管optional容器是否为空都会执行, 存在返回optional存储的实例, 不存在返回传入的实例
  User user1 = optionalFindName.orElse(new User("xcs", "not found"));
  // orElseGet 采用的是懒加载模式只有当optional容器为空的时候才会执行默认值
  User user2 = optionalFindName.orElseGet(() -> new User("xcs", "not found"));
  // ifPresent 不为空的时候会执行lambda表达式操作
  optionalFindName.ifPresent((user) -> System.out.println(user.getFullName()));
  // ifPresentOrElse 不为空的时候执行实现Consumer接口的函数, 不为空的
  optionalFindName.ifPresentOrElse(
      (user) -> System.out.println(user),
      () -> {
          System.out.println("not found");
      });
  // orElseThrow Optional容器中有值返回存储对象, 没有则抛出异常
  User user3 = optionalFindName.orElseThrow();
  // 还可以自定义异常类型
  User user3 = optionalFindName.orElseThrow(() -> new RuntimeException("not found class"));
  // filter 如果filter表达式里面的结果true的话结果就会返回该对象, 如果为false的时候结果就会被过滤掉 user为空会出现空指针
  Optional<User> optional = optionalFindName.filter((user) -> user.getFullName().equals("xcs"));
  System.out.println(optional.isPresent());
  ```

* **不适用场景**

  1. **类的字段** (由于Optional类的创建和管理有一定的开销所以不适用于类的字段, 而且会使类的序列化变得复杂)

     ```java
     public class User{
         Optional<String> name;
     }
     ```

  2. **方法的参数** (将Optional用作参数, 会使方法的使用和理解变得复杂)

     ```java
     public class User{
         public void getName(Optional<String> name){
         }
     }
     ```

  3. **构造参数** (这种情况会迫使调用者创建Optional实例)

     ```java
     public class User{
         public User(Optional<String> name){
         }
     }
     ```

  4. **集合类型** (应为集合已经可以通过isEmpty来判断是否为空)

     ```java
     public Optional<List<User>> getUsers(){
         return Optional.ofNullable(getList());
     }
     ```

  5. 建议**不要**使用**get()**方法 (这样仍会出现空值, 违背了Optional设计的初衷)

     ```java
     String username = OptionalValue.get();
     ```

     