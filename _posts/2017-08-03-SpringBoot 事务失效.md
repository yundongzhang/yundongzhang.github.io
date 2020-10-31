---
title: StringBoot 事务失效
description: JDK1.8 + SpringBoot2.x 出现事务失效
date: 2017-08-03 23:08:02
categories:
 - Bug
tags:
 - SpringBoot
 - Web
 - SQL
---

#### 事物失效的几种场景

- <a href="#数据库不支持事务">数据库不支持事务</a>
- <a href="#注解放在了私有方法上">注解放在了私有方法上</a>
- <a href="#类内部调用">类内部调用</a>
- <a href="#未捕获异常">未捕获异常</a>
- <a href="#多线程场景">多线程场景</a>
- <a href="#传播属性设置问题">传播属性设置问题</a>

#####  <a id="数据库不支持事务">数据库不支持事务</a>

> 这个就不用多讲了，一般来说使用 MYSQL 的同学数据库引擎都默认使用 Innodb。如果你选用的是其他引擎如：MyISAM 数据库引擎就天然不支持事务。具体事项出门左转。

#####  <a id="注解放在了私有方法上">注解放在了私有方法上</a>

```java
public class UserService{
  /**
  * 根据用户名称获取用户姓名
  */
  @Transactional(rollbackFor = Exception.class)
  private String getUserNameByUserId(Long UserId){
    return "张三";
  }
}
```

总结: SpringBoot 的事务管理器不支持在私有方法上添加事务!

#####  <a id="类内部调用">类内部调用</a>

```java
// 事务生效
public class UserService{
  /**
  * 根据用户名称获取用户姓名
  */
  @Transactional(rollbackFor = Exception.class)
  public String getUserNameByUserId(Long userId){ 
    return getUserByUserId.getUserName();
  }
  
  public User getUserByUserId(Long userId){
    return new User();
  }
}
// 事务不生效
public class UserService{
  /**
  * 根据用户名称获取用户姓名
  */
  public String getUserNameByUserId(Long userId){ 
    return getUserByUserId.getUserName();
  }
  
  @Transactional(rollbackFor = Exception.class)
  public User getUserByUserId(Long userId){
    return new User();
  }
}
```

总结: 同一个类中两个方法相互调用，事务注解应该添加在入口的方法上。**如：同一个一个类中 A 方法调用 B，那么注解就应该添加在 A 方法上，如果 B 方法调拨 A 那么注解就应该添加在 B 方法上**。

#####  <a id="未捕获异常">未捕获异常</a>

```java
public class UserService{
  /**
  * 根据用户名称获取用户姓名
  */
  @Transactional(rollbackFor = Exception.class)
  public String getUserNameByUserId(Long userId){ 
    try{
      return getUserByUserId.getUserName();
    }catch(Exception e){
    }
  }
}
```

总结: 异常不能自己捕捉，否则 SpringBoot 事务管理器就没法捕捉到异常。无法回滚。如果必须要添加异常捕捉一定要抛出异常

```java
public class UserService{
  /**
  * 根据用户名称获取用户姓名
  */
  @Transactional(rollbackFor = Exception.class)
  public String getUserNameByUserId(Long userId){ 
    try{
      return getUserByUserId.getUserName();
    }catch(Exception e){
      throw new RuntimeException(); // 手动抛出异常
    }
  }
}
```

#####  <a id="多线程场景">多线程场景</a>

```java
package net.isyundong.service.impl;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.UUID;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.atomic.AtomicInteger;
import net.isyundong.entity.A01;
import net.isyundong.entity.A02;
import net.isyundong.mapper.A01Mapper;
import net.isyundong.service.IA01Service;
import com.baomidou.mybatisplus.extension.service.impl.ServiceImpl;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.transaction.PlatformTransactionManager;
import org.springframework.transaction.TransactionDefinition;
import org.springframework.transaction.TransactionStatus;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.transaction.support.DefaultTransactionDefinition;


@Service
public class A01ServiceImpl extends ServiceImpl<A01Mapper, A01> implements IA01Service {

    /**
     * 创建线程池
     */
    private static final ExecutorService threadPool = Executors.newFixedThreadPool(4);
		
  	/**
  	 * 手动抛出异常判断用	
  	 */
    private static AtomicInteger ai = new AtomicInteger(1);

    @Autowired
    private PlatformTransactionManager transactionManager;

    @Autowired
    private A02ServiceImpl a02Service;

 
    @Override
    public void multiThread() {
      	// 用于记录事务
        List<TransactionStatus> transactionStatuss = Collections.synchronizedList(new ArrayList<TransactionStatus>());
        try {
            DefaultTransactionDefinition def = new DefaultTransactionDefinition();
            def.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRES_NEW);
            TransactionStatus transactionStatus = transactionManager.getTransaction(def);
            transactionStatuss.add(transactionStatus);
            A01 a1 = new A01();
            a1.setT01(randomString());
            save(a1);

            A02 a2 = new A02();
            a2.setT01(randomString());
            a02Service.save(a2);
        } catch (Exception e) {
            e.printStackTrace();
            for (TransactionStatus status : transactionStatuss) {
                status.setRollbackOnly();
            }
        }

        threadPool.execute(() -> {
            DefaultTransactionDefinition def = new DefaultTransactionDefinition();
            def.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRES_NEW);
            TransactionStatus transactionStatus = transactionManager.getTransaction(def);
            transactionStatuss.add(transactionStatus);

            try {
                A01 a111 = new A01();
                a111.setT01(randomString());
                save(a111);
                if (ai.incrementAndGet() % 3 == 0) {
                    int i = 0 / 0;
                }
            } catch (Exception e) {
                e.printStackTrace();
                for (TransactionStatus status : transactionStatuss) {
                    status.setRollbackOnly();
                }
            }
        });
    }

    public static String randomString() {
        return UUID.randomUUID().toString().replaceAll("-", "").substring(0, 5).toUpperCase();
    }

}

```



#####  <a id="传播属性设置问题">传播属性设置问题</a>

| 参数              | 解释                                                         |
| ----------------- | ------------------------------------------------------------ |
| **REQUIRED**      | 支持当前事务，如果当前没有事务，就新建一个事务。这是最常见的选择 |
| **SUPPORTS**      | 支持当前事务，如果当前没有事务，就以非事务方式执行           |
| **MANDATORY**     | 支持当前事务，如果当前没有事务，就抛出异常                   |
| **REQUIRES_NEW**  | 新建事务，如果当前存在事务，把当前事务挂起                   |
| **NOT_SUPPORTED** | 以非事务方式执行操作，如果当前存在事务，就把当前事务挂起     |
| **NEVER**         | 以非事务方式执行，如果当前存在事务，则抛出异常               |
| **NESTED**        | 支持当前事务，如果当前事务存在，则执行一个嵌套事务，如果当前没有事务，就新建一个事 |



---

[个人博客](isyundong.net)



