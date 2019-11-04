---
layout: post
title: spring状态机的实现
date: 2019-11-04
categories:
    - Spring
comments: true
permalink: spring-statemachine.html
---

在平时开发中，很多业务都会依赖状态的变化，例如订单有`下单>支付>发货>确认收货`。单据有`创建>审核通过>记账`，平时在开发中多在代码中写死并通过`if/else`判断。曾经尝试按状态模式写过状态机，效果不太理想。最近发现spring早已提供了状态机的功能，写个demo试一试。

# 快速开始
1. 定义订单的状态

```
public enum OrderState {

    //待付款
    NEW,

    //已付款，等待发货
    PAID,

    //已发货
    SHIPPED,

    //确认收货、完成
    COMPLETED;

}
```

2. 定义订单的事件（用户动作）

```
public enum OrderEvent {
  //下单
  CREATE,

  //付款
  PAY,

  //发货
  SHIP,

  //确认收货
  RECEIPT;
}
```

3. 配置状态

```
@Configuration
@EnableStateMachine
public class StateMachineConfig extends EnumStateMachineConfigurerAdapter<OrderState, OrderEvent> {

  @Override
  public void configure(StateMachineStateConfigurer<OrderState, OrderEvent> states)
      throws Exception {
    states
        .withStates()
        .initial(OrderState.NEW)// 初始状态
        .end(OrderState.COMPLETED)
        .states(EnumSet.allOf(OrderState.class));
  }

}
```

4. 配置状态变化

```
public void configure(StateMachineTransitionConfigurer<OrderState, OrderEvent> transitions)
  throws Exception {
transitions
	.withExternal()
	.source(OrderState.NEW).target(OrderState.PAID)// 指定状态来源和目标
	.event(OrderEvent.PAY)// 指定触发事件
	.and()
	.withExternal()
	.source(OrderState.PAID).target(OrderState.SHIPPED)
	.event(OrderEvent.SHIP)
	.and()
	.withExternal()
	.source(OrderState.SHIPPED).target(OrderState.COMPLETED)
	.event(OrderEvent.RECEIPT);
}
```

5. 配置监听器

```
@Override
public void configure(StateMachineConfigurationConfigurer<OrderState, OrderEvent> config)
  throws Exception {
config
	.withConfiguration()
	.listener(new OrderStateMachineListener());
}
```

6. 编写监听器

```
public class OrderStateMachineListener extends StateMachineListenerAdapter<OrderState, OrderEvent> {

  private static final Logger LOGGER = LoggerFactory.getLogger(OrderStateMachineListener.class);

  @Override
  public void transition(Transition<OrderState, OrderEvent> transition) {
    if (transition.getSource() == null && transition.getTarget().getId() == OrderState.NEW) {
      LOGGER.info("用户下单，等待支付");
      return;
    }
    if (transition.getSource().getId() == OrderState.NEW
        && transition.getTarget().getId() == OrderState.PAID) {
      LOGGER.info("用户完成支付，等待发货");
      return;
    }

    if (transition.getSource().getId() == OrderState.PAID
        && transition.getTarget().getId() == OrderState.SHIPPED) {
      LOGGER.info("卖家已发货");
      return;
    }

    if (transition.getSource().getId() == OrderState.SHIPPED
        && transition.getTarget().getId() == OrderState.COMPLETED) {
      LOGGER.info("用户确认收货");
      return;
    }
  }
}
```

7. 启动类测试

```
@SpringBootApplication
public class Application implements CommandLineRunner {

	public static void main(String[] args) {
		SpringApplication.run(Application.class, args);
	}

	@Autowired
	private StateMachine<OrderState, OrderEvent> stateMachine;

	@Override
	public void run(String... args) {
		stateMachine.start();
		stateMachine.sendEvent(OrderEvent.PAY);
		stateMachine.sendEvent(OrderEvent.SHIP);
		stateMachine.sendEvent(OrderEvent.RECEIPT);
	}

}
```

启动后输出如下：

```
c.g.e.s.state.OrderStateMachineListener  : 用户下单，等待支付
o.s.s.support.LifecycleObjectSupport     : started org.springframework.statemachine.support.DefaultStateMachineExecutor@601cbd8c
o.s.s.support.LifecycleObjectSupport     : started SHIPPED COMPLETED PAID NEW  / NEW / uuid=46b71fd2-27ee-4444-9984-65367bba8a77 / id=null
c.g.e.s.state.OrderStateMachineListener  : 用户完成支付，等待发货
c.g.e.s.state.OrderStateMachineListener  : 卖家已发货
c.g.e.s.state.OrderStateMachineListener  : 用户确认收货
o.s.s.support.LifecycleObjectSupport     : stopped org.springframework.statemachine.support.DefaultStateMachineExecutor@601cbd8c
```

# 参考资料

http://blog.didispace.com/spring-statemachine

https://nicky-chen.github.io/2018/12/19/state-machine