### 定时任务设置

* **主要是用到@Scheduled注解 cron来设置触发时间**

```java
package com.sky.task;


import com.sky.entity.Orders;
import com.sky.mapper.OrderMapper;
import com.sky.service.OrderService;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.scheduling.annotation.Schedules;
import org.springframework.stereotype.Component;

import java.time.LocalDateTime;
import java.util.List;

@Component
@Slf4j
public class OrderTask {
    @Autowired
    private OrderMapper orderMapper;

    @Scheduled(cron = "0 * * * * ?") // 每分钟检查一次
    public void cancelOrdersRegular(){
        log.info("定时处理支付超时订单");
        LocalDateTime time = LocalDateTime.now().plusMinutes(-15);
        List<Orders> ordersList = orderMapper.updateStatusToCancelOrders(Orders.PENDING_PAYMENT, time);
        if (ordersList.isEmpty()){
            return;
        }
        ordersList.forEach(orders -> {
            orders.setStatus(Orders.CANCELLED);
            orders.setCancelTime(LocalDateTime.now());
            orders.setCancelReason("订单支付超时");
            orderMapper.updateUserOrders(orders);
        });
    }

    @Scheduled(cron = "0 0 1 * * ?") // 每天凌晨1点检查前天的订单
    public void checkDeliveryOrder(){
        log.info("定时处理派送中的订单");
        LocalDateTime time = LocalDateTime.now().plusMinutes(-60);
        List<Orders> ordersList = orderMapper.updateStatusToCancelOrders(Orders.DELIVERY_IN_PROGRESS, time);
        if (ordersList.isEmpty()){
            return;
        }
        ordersList.forEach(orders -> {
            orders.setStatus(Orders.COMPLETED);
            orderMapper.updateUserOrders(orders);
        });
    }
}

```

