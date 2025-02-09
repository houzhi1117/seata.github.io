# 常见问题

## Q: java.lang.NoSuchMethodError: com.fasterxml.jackson.databind.jsontype.TypeSerializer.typeId(Ljava/lang/Object;Lcom/fasterxml/jackson/core/JsonToken;)

```
A: undolog序列化配置为jackson时，jackson版本需要为2.9.9+
```

## Q: seata高可用

```
A: 0.6版本开始支持，tc使用db模式共享全局事务会话信息，注册中心使用非file的seata支持的第三方注册中心
```

## Q: undo_log表log_status=1的记录

```
A: 场景：分支事务a注册TC后，a的本地事务提交前发生了全局事务回滚
   后果：全局事务回滚成功，a资源被占用掉，产生了资源悬挂问题
   防悬挂措施：a回滚时发现回滚undo还未插入，则插入一条log_status=1的undo记录，a本地事务（业务写操作sql和对应undo为一个本地事务）提交时会因为undo表主键冲突而提交失败。
```

## Q: 隔离性

```
A: 因seata一阶段本地事务已提交，为防止其他事务脏读脏写需要加强隔离。
    1.脏读 select语句加for update，代理方法增加@GlobalLock或@GlobalTransaction
    2.脏写 必须使用@GlobalTransaction
    注：如果你查询的业务的接口没有GlobalTransactional 包裹，也就是这个方法上压根没有分布式事务的需求，这时你可以在方法上标注@GlobalLock 注解，并且在查询语句上加 for update。
        如果你查询的接口在事务链路上外层有GlobalTransactional注解，那么你查询的语句只要加for update就行。设计这个注解的原因是在没有这个注解之前，需要查询分布式事务读已提交的数据，但业务本身不需要分布式事务。
        若使用GlobalTransactional注解就会增加一些没用的额外的rpc开销比如begin 返回xid，提交事务等。GlobalLock简化了rpc过程，使其做到更高的性能。
```

## Q: 脏数据回滚失败如何处理

```
A: 
    1.脏数据需手动处理，根据日志提示修正数据或者将对应undo删除（可自定义实现FailureHandler做邮件通知或其他）
    2.关闭回滚时undo镜像校验，不推荐该方案。
    注：建议事前做好隔离保证无脏数据
```

## Q: 分支事务注册时全局事务状态不是begin

```
A:  
    异常：Could not register branch into global session xid = status = Rollbacked（还有Rollbacking、AsyncCommitting等等二阶段状态） while expecting Begin
    描述：分支事务注册时，全局事务状态需是一阶段状态begin，非begin不允许注册。属于seata框架层面正常的处理，用户可以从自身业务层面解决。
    有以下几种情况会出现该异常（可继续补充）
    1.分支事务是异步，全局事务无法感知它的执行进度，全局事务已进入二阶段，该异步分支才来注册
    2.服务a rpc 服务b超时（dubbo、feign等默认1秒超时），a上抛异常给tm，tm通知tc回滚，但是b还是收到了请求（网络延迟或rpc框架重试），然后去tc注册时发现全局事务已在回滚
    3.tc感知全局事务超时(@GlobalTransactional(timeoutMills = 默认60秒))，主动变更状态并通知各分支事务回滚，此时有新的分支事务来注册
```