<img src="D:\桌面文件\javaStream\stream的执行过程.jpg">

##  Stream Creation

* 创建固定数据类型流的方式

  ```java
  Stream<Integer> stream1 = List.of(1, 2, 3).stream();
  Stream<String> stream3 = Stream.of("a", "b", "c", "d");
  Stream<String> stream4 = Stream.of("e", "f", "g");
  //  流的合并
  Stream<String> streamSum = Stream.concat(stream3, stream4);
  ```

* 基础类型流创建

  ```java
  IntStream stream2 = Arrays.stream(new int[] { 1, 2, 3, 4, 5 });
  //将基础类型转换成对象的形式
  Stream<Integer> boxed = stream2.boxed();
  IntStream range = IntStream.range(1, 4);
  // 闭区间
  IntStream rangeClosed = IntStream.rangeClosed(1, 4);
  // 生成n个随机数的流
  new Random().ints(5).forEach(System.out::println);
  ```

* 文件流

  ```java
  Path path = Paths.get("test.txt");
  try (Stream<String> lines = Files.lines(path)) {
      lines.forEach(System.out::println);
  } catch (Exception e) {
      e.printStackTrace();
  }
  ```

* 动态stream, 通过builder方法创建StringBuilder对象

  ```java
  Builder<String> builder = Stream.builder();
  builder.add("a");
  builder.add("b");
  builder.add("c");
  
  // 通过build方法创建流对象
  Stream<String> stream = builder.build();
  ```

* 处理无限流 通常用limit来限制个数

  ```java
  Stream<String> constantSteams = Stream.generate(() -> "amani").limit(5);
  // 生成Double类型的随机数
  Stream.generate(Math::random).limit(5).forEach(System.out::print);
  //  iterate 用来生成数学中的序列
  //  等差
  Stream<Integer> iterateStream = Stream.iterate(0, n -> n + 2).limit(5);
  iterateStream.forEach(System.out::print);
  // 高级 添加添加判断
  Stream<Integer> iterate = Stream.iterate(0, n -> n < 10, n -> n = n + 2);
  iterate.forEach(System.out::println);
  // 并行流创建
  iterate.parallel();
  // 集合并行流的创建
  List.of(1, 2, 3, 4, 5).parallelStream();
  ```

## Intermediate  Operation

```java
// init testDatas
List<Person> persons = 
    List.of(
        new Person(12, "xcs", "123"),
        new Person(14, "bob", "123"),
        new Person(15, "alice", "123"),
        new Person(16, "tom", "123"),
        new Person(17, "xcs", "123"),
        new Person(18, "bob", "123")
);
List<List<Person>> personGroup = 
    List.of(
        List.of(
            new Person(12, "xcs", "123"),
            new Person(14, "bob", "123")),
        List.of(
            new Person(15, "alice", "123"),
            new Person(16, "tom","123")),
        List.of(
            new Person(17, "xcs", "123"),
            new Person(18, "bob", "123"))
);
```

* **distinct()** 去重操作

  ```java
  persons.stream().distinct().forEach(System.out::println);
  ```

* **map()** 提取出对象中的某一属性单独作为一个集合

  ```java
  List<String> names = persons.stream()
                                  .map(Person::getName)
                                  .collect(Collectors.toList());
  ```

  

* **limit(n)** 限制收集个数

  ```java
  persons.stream().limit(3).forEach(System.out::println);
  ```

* **skip(n)** 跳过元素个数

  ```java
  persons.stream().skip(3).forEach(System.out::println);
  ```

* **flatMap()**  flagMap 可以对原始流中的每一个元素做相同的处理生成多个stream流最后将这些流合成成为单一的扁平化的流

  ```java
  personGroup.stream().flatMap(person -> person.stream())
                          .map(Person::getName)
                          .forEach(System.out::println);
   // 想通过将person装换成name流再将name流合并
   personGroup.stream().flatMap(person -> person.stream().map(Person::getName))
                  .forEach(System.out::println);
  ```

* **sorted()** stream中sort函数的使用 可以根据**Comparator接口**自定义排序规则

  ```java
  persons.stream()
          .sorted(Comparator.comparingInt(Person::getAge))
          .forEach(System.out::println);
  persons.stream()
     		.sorted(Comparator.comparingInt((Person person) -> person.getName().length())
      	.reversed())
      	.forEach(System.out::println);
  ```

* 结合

  ```java
  personGroup.stream()
      //将内聚集合多个流合并成扁平化的流
              .flatMap(person -> person.stream())
      // 过滤
              .filter(person -> person.getAge() >= 14)
      // 去重
              .distinct()
      // 根据年龄排序
              .sorted(Comparator.comparingInt(Person::getAge))
      // 限制结果集个数
              .limit(3)
      // 跳过一个元素
              .skip(1)
      // 见结果集合中的每个元素的name属性提取出来
              .map(Person::getName).forEach(System.out::println);
  ```

*  **anyMatch()**    对象中有满足条件的元素返回true

  ```java
  boolean result1 = persons.stream().anyMatch(person -> person.getAge() > 15);
  ```

* **allMatch()**        对象所有元素都满足条件返回true

  ```java
  boolean result1 = persons.stream().allMatch(person -> person.getAge() > 15);
  ```

* **noneMatch()**    对象中的元素都没有满足条件返回true

  ```java
  boolean result1 = persons.stream().noneMatch(person -> person.getAge() > 19);
  ```

* **findFirst()**         得到集合的第一个元素

  ```java
  // 返回的结果是Optional类型 为了是处理null
  Optional<Person> first = persons.stream().findFirst();
  // 如果不为空则输出
  first.ifPresent(System.out::println);
  ```

* **findAny()**          得到集合任意一个元素

  ```java
  Optional<Person> any = persons.stream().findAny();
  any.ifPresent(System.out::println);
  ```

  ### 聚合操作 Aggregation

  * **count** 统计集合数量

    ```java
    long count = persons.stream().count();
    ```

  * **min**    计算集合最小值

    ```java
    // min 要集合Comparator来判断条件
    Optional<Person> minPersOptional = persons.stream().min(Comparator.comparingInt(Person::getAge));
    minPersOptional.ifPresent(System.out::println);
    ```

  * **max**   计算集合最大值

    ```java
    Optional<Person> maxPersOptional = persons.stream().max(Comparator.comparingInt(Person::getAge));
    maxPersOptional.ifPresent(System.out::println);
    ```

  * **sum**  操作是基于基础类型的流

    ```java
    // 使用sum需要先将流转换成基础类型的流
    long sumAge = persons.stream().mapToInt(Person::getAge).sum();
    ```

  * **average** 计算平均值也是基于基础类型的流计算

    ```java
    OptionalDouble average = persons.stream().mapToInt(Person::getAge).average();
    average.ifPresent(System.out::println);
    ```

## Terminal Operations

* **reduce()** 自定义迭代器实现合并 拼接

  ```java
  long sumReduce = persons.stream()
                                  .mapToInt(Person::getAge)
                                  .reduce(0, (a, b) -> a + b);
  String sumNames = persons.stream()
      						  .map(Person::getName)
      						  .reduce("", (a, b) -> a + b + ",");
  ```

* **collect()** 收集器

  * **groupingBy()**               分组

    ```java
    // 以对象中的某一个属性作为key, 含有该属性的对象对位key进行分组
    Map<String, List<Person>> personGroupByName = persons.stream()
     												 .collect(Collectors.groupingBy(Person::getName));
    ```

  * **partitioningBy()**          根据条件分组

    ```java
    // 将结果集分成符合条件的集合与不符合条件的集合
    Map<Boolean, List<Person>> patitioningByAge = persons.stream()
        .collect(Collectors.partitioningBy(person -> person.getAge() > 15));
    ```

  * **joining()**                       拼接

    ```java
    String namesStr = persons.stream().map(Person::getName).collect(Collectors.joining(","));
    ```

  * **summarizing()**             统计结果为基础类型

    ```java
    IntSummaryStatistics statistics = persons.stream().collect(Collectors.summarizingInt(Person::getAge));
                    System.out.println(statistics.getMax());
                    System.out.println(statistics.getMin());
                    System.out.println(statistics.getAverage());
    ```

  * **自定义collector 来实现grouping 和 toList**

    ```java
    // 自定义toList()
    persons.stream()
            .collect(Collector.of(
                // 给 supplier 提供容器
                () -> new ArrayList(),
                (list, person) -> {
                    // System.out.println("Accumulator :" + person);
                    list.add(person);
                },
                // 处理并行流
                (left, right) -> {
                    // System.out.println("Combiner :" + left);
                    left.addAll(right);
                    return left;
                },
                // 设置结果处理方式 默认
                Collector.Characteristics.IDENTITY_FINISH));
    // 自定义grouping 
    HashMap<String, List<Person>> groupHashMap = persons.stream().parallel()
                .collect(Collector.of(
                    HashMap::new,
                    (map, person) -> {
                        System.out.println("Accumulator : " + person);
                        map.computeIfAbsent(person.getName(),(k) -> new ArrayList<>()).add(person);
                    },
                    (left, right) -> {
                        System.out.println("Combiner : " + left);
                        right.forEach((key, value) -> left.merge(key, value, 
                                                                 (list, newList) -> {
                                                                     list.addAll(newList);
                                                                     return list;
                                                                 }));
                        return left;
                    },
                    Collector.Characteristics.IDENTITY_FINISH));
    ```

  ### ParallelStream 

  <img src="D:\桌面文件\javaStream\并行流.jpg" alt="并行流操作过程" style="zoom: 33%;" >

  **并行流**在开始时,**Spliterator分割迭代器**将数据分割成多个片段,分割的过程通常采用**递归**的方式进行,以平衡子任务负载平衡,提高资源利用率, 然后**Fork/Join**框架将这些片段分配到多个线程和处理器核心上进行并行处理,处理完成后结果将会被**汇总合并**, 核心是分解**Fork**和结果合并**Join**.

  ```java
  // 并行流采用多线程并发处理结果顺序会变乱
  List.of("x","c","s","n","b").parallerlStream()
      	.map(String::toLowerCase)
      	.forEach(item ->{
              System.out.println("item: " + item + "->" + " Thread: " + Thread.currentThread().getName())
          });
  //forEachOrdered可以保持数据结果的出现顺序
  List.of("x","c","s","n","b").parallerlStream()
      	.map(String::toLowerCase)
      	.forEachOrdered(System.out::println);
  /**
  	保持结果维持其出现顺序的原理是处理并行流的时候对于有序数据源比如list, Spliterator会对数据源进行递归分割, 分割通常是基于逻辑上的而非物理上的复制数据,而是通过划分数据源的索引范围来实现, 每次分割都会产生一种新的Spliterator实例,该实例内部维护了指向源数据的索引范围这种分割机制可以让数据的出现顺序得以保持, 然后Fork/Join框架接手将分配后的数据块分配给不同的子任务执行, 其依据Spliterator维护的顺序信息来调度方法的执行顺序,意味着即使某个子任务较早完成, 如果其关联的方法执行顺序还未到来,系统将缓存数据并暂停该方法,直到所有前序任务都已完成,这种机制确保了在并行处理情况下每个操作也会按照原始数据的出现顺序执行.
  	但是这么做也可能牺牲一些并行执行的效率
  */
  ```

  <img src="D:\桌面文件\javaStream\使并行流有序方法原理.jpg" style="zoom:33%;" >

  