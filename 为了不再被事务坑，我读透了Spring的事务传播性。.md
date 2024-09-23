
在之前文章中，我们已经被事务坑了两次：

[mq发送消息之后，业务代码回滚，导致发了一条中奖消息给用户！！](https://juejin.cn/post/7362537451170758692)

[我又被Spring的事务坑了，用户兑奖之后，什么东西都没收到！！](https://juejin.cn/post/7397025359525756928)

所以作者对Spring的事务深恶痛绝，这次我们来再次对spring的事务发起进攻，还是用用户中奖这个例子去解释这个Spring的事务（用户：麻烦给一下出场费），防止以后再次出现这样的情况，这次我们的攻击点就是spring的事务的七种传播性。

Spring框架中的事务传播性是指当一个事务方法被另一个事务方法调用时，如何处理这种嵌套调用的情况。

虽然在上一篇文章中我们说到：**一个Transactional注解方法就是一个mysql里面的事务**，但是这里大家不要把Spring的事务和mysql的隔离级别搞混了。

我们先来看一个简单的例子，有一个接口是用户中奖的接口：

```
    /**
     * 中奖
     */
    @GetMapping("/winning")
    public String winning(@RequestParam Integer userId) {
        return userService.winning(userId);
    }
```

![image-20240808160641579](https://i-blog.csdnimg.cn/blog_migrate/18ea56afde49f11e2d98c96055a553e1.png)

这里面是他的实现类，在这里，除了往user表中插入了一条数据，还往中奖记录表中添加了一条数据。我们这里抛出了一个异常用来模拟我们在业务处理中遇到的异常。

```
    @Override
    public String winning(Integer userId) {
        User user = new User();
        user.setId(userId);
        user.setUsername("user" + userId);
        userDao.insert(user);
        winningService.winning2(userId);
        throw new RuntimeException("fu*k Transactional");
    }
```

![image-20240808160734077](https://i-blog.csdnimg.cn/blog_migrate/10a1eb23a14f39c0146f25b6e8771d90.png)

```
    @Override
    public void winning2(Integer userId) {
        Winning winning = new Winning();
        winning.setUserId(userId);
        winningDao.insert(winning);
    }
```

![image-20240808160801799](https://i-blog.csdnimg.cn/blog_migrate/c0e7d5926ba133d06021c8ea826a0edc.png)

下面是两张表的样子（demo演示，不会有这么简单的表的）：

中奖用户表

![image-20240805195234627](https://i-blog.csdnimg.cn/blog_migrate/80625c5fd8e55f1bfe9db6a97c3b94d7.png)

中奖记录表

![image-20240805195254275](https://i-blog.csdnimg.cn/blog_migrate/67d642cca0bc74b077130d4b2cedb264.png)

这个时候我们来模拟用户中奖：

![image-20240805200815719](https://i-blog.csdnimg.cn/blog_migrate/ad6b60fa09d8be0e397f99585ec04306.png)

![image-20240805200914028](https://i-blog.csdnimg.cn/blog_migrate/2fcff6860c02552ebaeaad3784cac543.png)

可以看到，我们的后台已经报异常了，但是这个时候我们看我们的表数据就会发现，居然两条数据都插入进去了：

![image-20240805201151898](https://i-blog.csdnimg.cn/blog_migrate/2a9361603feccfac6119a25f7656aa54.png)

这就有问题了呀，兄弟，哥们！！就会出现我们上面的情况：

[我又被Spring的事务坑了，用户兑奖之后，什么东西都没收到！！](https://juejin.cn/post/7397025359525756928)

## 1. PROPAGATION_REQUIRED (默认值)

   - 如果当前存在事务，则加入该事务；如果当前没有事务，则创建一个新的事务。

   ### 如果当前存在事务，则加入该事务：

   这个是Transactional注解的默认值，也就是说你就写一个@Transactional，那么默认就是这个，所以我们把方法改为下面的试试：

   ```
    @Override
    @Transactional
    public String winning(Integer userId) {
        User user = new User();
        user.setId(userId);
        user.setUsername("user" + userId);
        userDao.insert(user);
        winningService.winning2(userId);
        throw new RuntimeException("fu*k Transactional");
    }
    
    @Override
    @Transactional
    public void winning2(Integer userId) {
        Winning winning = new Winning();
        winning.setUserId(userId);
        winningDao.insert(winning);
    }

   ```

   这个时候我们调用接口会出现什么情况呢？我们来试一下

​    ![image-20240806195407438](https://i-blog.csdnimg.cn/blog_migrate/e74b66ff01187ebc9762c2f026c0aeb8.png)

   没错，他回滚了。也就是说没有执行，也就是说在外面的winning方法中是一整个大事务，只要事务里面报错了，则都会不commit事务。

   ![image-20240806195636896](https://i-blog.csdnimg.cn/blog_migrate/4f2ac46b5dbed417003f19b2572f2d28.png)

  其实把异常放到这里也是可以全部回滚的：

```
    @Override
    @Transactional
    public String winning(Integer userId) {
        User user = new User();
        user.setId(userId);
        user.setUsername("user" + userId);
        userDao.insert(user);
        winningService.winning2(userId);
        return "ok";
    }

    @Override
    @Transactional
    public void winning2(Integer userId) {
        Winning winning = new Winning();
        winning.setUserId(userId);
        winningDao.insert(winning);
        throw new RuntimeException("fu*k Transactional2");
    }
```

 但是估计有时候大家也是这么写的：

   ```
@Override
@Transactional
public String winning(Integer userId) {
    User user = new User();
    user.setId(userId);
    user.setUsername("user" + userId);
    userDao.insert(user);
    winning2(userId);
    throw new RuntimeException("fu*k Transactional");
}

@Autowired
private WinningDao winningDao;

public void winning2(Integer userId) {
    Winning winning = new Winning();
    winning.setUserId(userId);
    winningDao.insert(winning);
}
   ```

   这样写的后果就是事务也会回滚，（什么鬼？这个不是自己调用自己的方法吗，上次你不是说这种情况会失效吗？

   [我又被Spring的事务坑了，用户兑奖之后，什么东西都没收到！！](https://juejin.cn/post/7397025359525756928#heading-3)）

   盆友，稍安勿躁，且听我细细道来

   之所以上面的方法也会回滚，是因为在外面本来就开启了一个事务，在调用自己的方法过程中其实都是在一个事务里面执行的，这并没有什么不妥，也照样走的是spring的代理，因为是winning是通过controller层调用进来的，所以自然是走了spring的代理，所以说，我们下面这样写结果也是会回滚的：

   ```
@Override
@Transactional
public String winning(Integer userId) {
    User user = new User();
    user.setId(userId);
    user.setUsername("user" + userId);
    userDao.insert(user);
    winning2(userId);
    throw new RuntimeException("fu*k Transactional");
}

@Autowired
private WinningDao winningDao;

@Transactional
public void winning2(Integer userId) {
    Winning winning = new Winning();
    winning.setUserId(userId);
    winningDao.insert(winning);
}
   ```

   那什么情况下才符合我们之前说过的[我又被Spring的事务坑了，用户兑奖之后，什么东西都没收到！！](https://juejin.cn/post/7397025359525756928#heading-3)第三种情况自己调用自己就会失效呢？

   这还不简单，你只要了解其中的原理就可以写出来了：

   ![image-20240808161628514](https://i-blog.csdnimg.cn/blog_migrate/c78262c8ce9996dcc25188c0433f787e.png)

   ```
@Override
public void  winningNoTransactional(Integer userId) {
    winning(userId);
}
   ```

   ![image-20240808161653760](https://i-blog.csdnimg.cn/blog_migrate/285bfa5104a3ac60245e1849ac448456.png)

   可以看到我们上面这样写，虽然winning有一个事务，但是他的上一层是自己类里面的方法调用自己的，相当于没走spring的代理，有的@Transactional和没的一样，所以自然不会生效了

   ![image-20240808161933637](https://i-blog.csdnimg.cn/blog_migrate/b2162c05e1dd21f8decc7d1105b2ea20.png)

   ### 如果当前没有事务，则创建一个新的事务:

   ```
@Override
public String winning(Integer userId) {
    User user = new User();
    user.setId(userId);
    user.setUsername("user" + userId);
    userDao.insert(user);
    winningService.winning(userId);
    throw new RuntimeException("fu*k Transactional");
}

	@Override
    @Transactional
    public void winning(Integer userId) {
        Winning winning = new Winning();
        winning.setUserId(userId);
        winningDao.insert(winning);
    }
   ```

   那么这样的会回滚吗？答案是都插进去了：

   ![image-20240808155415889](https://i-blog.csdnimg.cn/blog_migrate/cfa06f67fe6dcd77918ea679e7b0ac54.png)

   所以这里我们要注意一下，这样的写法不行的，因为第一个没有事务，第二个事务里面没有报异常，所以后面报了异常事务是不会回滚的，我们可以这样写：

   ```
@Override
@Transactional
public void winning(Integer userId) {
    Winning winning = new Winning();
    winning.setUserId(userId);
    winningDao.insert(winning);
    throw new RuntimeException("fu*k Transactional2");
}
   ```

   ![image-20240808155742825](https://i-blog.csdnimg.cn/blog_migrate/70c8a97fc5a0e0c5eb81bccd8c523efc.png)

   这个结果是user表插入成功，但是中奖表没有插入成功，但是这个符合我们的这个事务传播行为：**如果当前没有事务，则创建一个新的事务**

 ## 2.PROPAGATION_SUPPORTS

   - 如果当前存在事务，则加入该事务；如果当前没有事务，则以非事务的方式继续执行。

   ### 如果当前存在事务，则加入该事务

   ```
@Override
@Transactional
public String winning(Integer userId) {
    User user = new User();
    user.setId(userId);
    user.setUsername("user" + userId);
    userDao.insert(user);
    winningService.winning2(userId);
    throw new RuntimeException("fu*k Transactional");
}

     @Override
    @Transactional(propagation = Propagation.SUPPORTS)
    public void winning2(Integer userId) {
        Winning winning = new Winning();
        winning.setUserId(userId);
        winningDao.insert(winning);
    }
   ```

这个结果显而易见是不会插入成功的：

![image-20240808164050974](https://i-blog.csdnimg.cn/blog_migrate/860b84efbb7101ec5e258b027a06820a.png)

### 如果当前没有事务，则以非事务的方式继续执行

```
    @Override
    public String winning(Integer userId) {
        User user = new User();
        user.setId(userId);
        user.setUsername("user" + userId);
        userDao.insert(user);
        winningService.winning2(userId);
        throw new RuntimeException("fu*k Transactional");
    }
    
    
        @Override
    @Transactional(propagation = Propagation.SUPPORTS)
    public void winning2(Integer userId) {
        Winning winning = new Winning();
        winning.setUserId(userId);
        winningDao.insert(winning);
        throw new RuntimeException("fu*k Transactional2");

    }
```

可以看到是成功插入了，因为winning没有事务，winning2的传播行为也不会加入事务，即使抛出了异常，也不会回滚。

![image-20240808164451029](https://i-blog.csdnimg.cn/blog_migrate/a8f59a7d5c19b56efaeeabf6a9d36607.png)

## 3.PROPAGATION_MANDATORY

   - 如果当前存在事务，则加入该事务；如果当前没有事务，则抛出异常。

### 如果当前存在事务，则加入该事务.

这个和第一点第二点是一样的（讲个锤子）

### 如果当前没有事务，则抛出异常

```
    @Override
    public String winning(Integer userId) {
        User user = new User();
        user.setId(userId);
        user.setUsername("user" + userId);
        userDao.insert(user);
        winningService.winning2(userId);
        throw new RuntimeException("fu*k Transactional");
    }

    @Override
    @Transactional(propagation = Propagation.MANDATORY)
    public void winning2(Integer userId) {
        Winning winning = new Winning();
        winning.setUserId(userId);
        winningDao.insert(winning);
        throw new RuntimeException("fu*k Transactional2");

    }
```

这个传播行为很有用，例如你写一个公用的方法给其他同事调用的话可以致使他的方法必须拥有事务。

![image-20240808165614936](https://i-blog.csdnimg.cn/blog_migrate/55e1cd7e6e4b910ee316248a28f5083b.png)

但是结果呢？显而易见，第一张表插入了，第二张没插入（因为第二个方法都没进来就报错了，第一个没有事务，自然插入成功了）

![image-20240808165807029](https://i-blog.csdnimg.cn/blog_migrate/0a42b3fa446fdf68c62fbfd010523b58.png)

##  4.PROPAGATION_REQUIRES_NEW

   - 创建一个新的事务，并且挂起当前的事务（如果存在的话）。

```
    @Override
    @Transactional
    public String winning(Integer userId) {
        User user = new User();
        user.setId(userId);
        user.setUsername("user" + userId);
        userDao.insert(user);
        winningService.winning2(userId);
        return "ok";
    }

    @Override
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void winning2(Integer userId) {
        Winning winning = new Winning();
        winning.setUserId(userId);
        winningDao.insert(winning);
        throw new RuntimeException("fu*k Transactional2");
    }
```

这个结果就是两个都没插入，这个很好理解，但是可能大家不理解**挂起**的概念

解释：

1. winning 方法：当调用 winning 方法时，一个新的事务（记为事务A）开始。
2. 插入用户：在事务A中，User 对象被创建并插入数据库。
3. 调用 winning2 方法：接着调用 winning2 方法，这将启动一个新的事务（记为事务B），并且事务A被挂起。
4. 插入 Winning 记录：在事务B中，Winning 对象被创建并插入数据库。
5. 抛出异常：在事务B中抛出 RuntimeException，导致事务B回滚，撤销所有更改。
6. 恢复事务A：事务B结束后，事务A被恢复。

如果 winning2 方法中的异常没有被捕获，并且传播到了 winning 方法中，那么 winning 方法中的事务A也会被回滚，所以如果我们想 winning 方法里面的user真正插入的话，我们就可以这样写：

```
@Override
@Transactional
public String winning(Integer userId) {
    try {
        User user = new User();
        user.setId(userId);
        user.setUsername("user" + userId);
        userDao.insert(user);

        winningService.winning2(userId);
    } catch (Exception e) {
        log.error("Exception occurred in winning method: ", e);
        // 这里可以选择记录错误信息，或者做一些其他的错误处理
        // 但是不需要在这里回滚事务，因为Spring会自动处理
    }
    return "ok";
}

@Override
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void winning2(Integer userId) {
    Winning winning = new Winning();
    winning.setUserId(userId);
    winningDao.insert(winning);
    throw new RuntimeException("fu*k Transactional2");
}
```

这样大家是不是就理解了**创建一个新的事务，并且挂起当前的事务**这句话了

## 5.PROPAGATION_NOT_SUPPORTED

   - 以非事务方式执行操作，并挂起当前事务（如果存在的话）。

```
    @Override
    @Transactional
    public String winning(Integer userId) {
        User user = new User();
        user.setId(userId);
        user.setUsername("user" + userId);
        userDao.insert(user);
        winningService.winning2(userId);
        return "ok";
    }

  @Override
    @Transactional(propagation = Propagation.NOT_SUPPORTED)
    public void winning2(Integer userId) {
        Winning winning = new Winning();
        winning.setUserId(userId);
        winningDao.insert(winning);
        throw new RuntimeException("fu*k Transactional2");
    }
```

**执行流程**

1. winning 方法开始：当调用 winning 方法时，一个新的事务（记为事务A）开始。
2. 插入用户：在事务A中，User 对象被创建并插入数据库。
3. 调用 winning2 方法：接着调用 winning2 方法。由于 winning2 方法使用了 Propagation.NOT_SUPPORTED，这意味着：
4. 如果当前存在事务（即事务A），则该方法不在任何事务中执行。
5. 如果当前不存在事务，则该方法同样不在任何事务中执行。
6. 插入 Winning 记录：在 winning2 方法中，Winning 对象被创建并插入数据库。由于 winning2 方法不在事务中执行，因此数据库操作直接提交，不会等待事务结束。
7. 抛出异常：在 winning2 方法中抛出 RuntimeException。
8. 异常处理：由于 winning2 方法不在事务中执行，异常直接抛出给调用者 winning 方法。
9. 异常传播：如果 winning 方法没有捕获这个异常，那么异常会继续向上层传播，导致事务A回滚。

所以结果是，winning表插入了，user表没有：

![image-20240808173551550](https://i-blog.csdnimg.cn/blog_migrate/9ae4a898e2bf79e64f5d405e91260cda.png)

如果为了确保 User 数据能够被正确插入，同时避免事务A因 winning2 方法中的异常而回滚，我们也可以在 winning 方法中捕获异常，以确保事务A能够正常提交。

## 6.PROPAGATION_NEVER

   - 以非事务方式执行，如果当前存在事务，则抛出异常。

![image-20240808173844001](https://i-blog.csdnimg.cn/blog_migrate/01667dcdfc905649ed165dfd4cd506d1.png)

这个传播行为也很有意思，如果当前存在事务，则抛出`IllegalTransactionStateException`异常

和第三点PROPAGATION_MANDATORY完全相反的，而且是直接以非事务方式执行

## 7.PROPAGATION_NESTED

   - 如果当前存在事务，则在嵌套事务内执行；如果当前没有事务，则其行为类似于`PROPAGATION_REQUIRED`。

### 如果当前存在事务，则在嵌套事务内执行

```
    @Override
    @Transactional
    public String winning(Integer userId) {
        User user = new User();
        user.setId(userId);
        user.setUsername("user" + userId);
        userDao.insert(user);
        winningService.winning2(userId);
        return "ok";
    }

  @Override
    @Transactional(propagation = Propagation.NESTED)
    public void winning2(Integer userId) {
        Winning winning = new Winning();
        winning.setUserId(userId);
        winningDao.insert(winning);
        throw new RuntimeException("fu*k Transactional2");
    }
```

结果是两个表都没插入，但是什么叫嵌套事务呢？

嵌套事务是事务winning的一个子事务（记为事务winning2），它与事务winning共享相同的资源，但有自己的保存点。

### 如果当前没有事务，则其行为类似于`PROPAGATION_REQUIRED`

也就是说类似于我们的第一种：**如果当前没有事务，则创建一个新的事务**

```
    @Override
    public String winning(Integer userId) {
        User user = new User();
        user.setId(userId);
        user.setUsername("user" + userId);
        userDao.insert(user);
        winningService.winning2(userId);
        return "ok";
    }
    
    
    @Override
    @Transactional(propagation = Propagation.NESTED)
    public void winning2(Integer userId) {
        Winning winning = new Winning();
        winning.setUserId(userId);
        winningDao.insert(winning);
        throw new RuntimeException("fu*k Transactional2");
    }
```

结果就是user新建了，winning表没有新建

![image-20240808194941233](https://i-blog.csdnimg.cn/blog_migrate/b9fa08403121026a547a5036df86a412.png)

相信仔细看完的同学已经注意到了里面有很多名词：**加入事务，挂起事务，嵌套事务**

这里做一个简单的总结，更详细的这里不做研究（性价比不高，很少用得到，上面7总如果开发用到了，根据实际情况调整即可）

**加入事务**：`REQUIRED` 加入当前事务，或者创建一个新的事务。

**挂起事务**：`REQUIRES_NEW` 挂起当前事务，创建一个新的事务执行方法。

**嵌套事务**：`NESTED` 在现有事务内创建一个子事务，可以独立回滚至 **Savepoint**。


> ![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/31dc8042deb97d9fdd67c146bd3fac11.png)

