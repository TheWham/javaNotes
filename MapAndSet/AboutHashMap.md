##                                                                              HashMap的基本用法

* #### 关于构造器

  1. 基本构造器 `HashMap<T,K> parameter = new HashMap<>()`
  2. 简单点的     `var parameter = new HashMap<T,k>() `**注意要带上参数类型**
  3. `HashMap(Map<? extends K,? extends V> m) parameter`  **构造一个新的 `HashMap` ，其映射与指定的 `Map`相同。**

* #### 关于函数方法

  1. `V` `put(K key , V value)` **用来添加元素的**

     `parameter.put(key,value)`

  2. `V ` `get(Object key)` **返回键映射的值如果没有则返回null**

     `parameter.get(key)` 

  3. `Set<K>`  `ketSet()`**返回此映射中包含的键的`Set`视图**

     `Set<k> parameter = map.keySet()`

     ```java
     for(K ans : map.keySet())
         System.out.println(ans + map.get(ans));
     ```

  4. `Set<Map.Entry<String,Integer>>`   `entrySet()` **返回此映射中包含的映射的`Set`图.**

      还可以单独拿出来在遍历的时候使用

     ```java
     for(HashMap<K,V> parameter : map.entry())
         System.out.println(parameter.getKey() + parameter.getValue())
     ```

  5. `Collection<V>` `values()` 返回此映射种包含的值的`collections`视图

     ```java
     for(Collection<V> value : values)
         System.out.println(value); //但是不能通过value得到key
     ```

  6. `int size()` **返回次映射种键-值映射的数量**

  7. `V remove(Object key)` **从此映射中删除指定键的映射（如果存在)**

  8. `boolean`  `containKey(Object key)` **如果此映射包含指定键的映射，则返回 `true` **

  9. `boolean`  `containValue(Object value)`**如果此映射将一个或多个键映射到指定值，则返回 `true` 。**

  10. `boolean`  `isEmpty()` **如果此映射不包含键 - 值映射，则返回 `true` 。**

  11. `void` `putAll(Map<? extends K,? extends V> map)` **将指定映射中的所有映射复制到此映射。**

* #### 关于遍历

  + 迭代器

    ```java
    Iterator<HashMap.Entry<K,V>> iter = map.entrySet();
    while(iter.hasNext())
    {
        var load = iter.next();
        System.out.println(load.getKey() + load.getValue());
        //不能搞iter.next().getKey()+" "+iter.next().getValue() 这样iter就遍历两次了
    }
    /*同理遍历键和值也是如此*/
    Iterator<K> iter = map.keySet().iterator();
    Iterator<V> iter = map.values().iterator();
    ```

  + for-each  (Map 本身不能用for-each遍历，但是可以转换)

    ```java
    /*遍历Key*/
    for(var key : mp.ketSet())
        System.out.println(key + mp.get(key));
    /*遍历值*/
    for(var value : mp.values())
        System.out.println(value);
    /*遍历键和值*/
    for(Map.Entry<K,V> ans : mp.entrySet())
         System.out.println(ans.getKey() + ans.getValue());
    ```

  + Stream

    ```java
    ```

    

