---
title: TDD
date: 2019-10-29
tags:
 - TDD
 - mock
 - stub
---

> martin fowler关于测试的一些文章总结

## TDD

缘起于Kent Back极限编程(XP Extreme Programming)方法论中的一种方法-测试驱动开发(Test-Driven Development).
遵循以下三个步骤:
* 首先编写功能相关的测试
* 编写功能代码直到通过测试
* 重构新旧代码,保持结构整洁

TDD能够带来两个好处:
* 有助于写出自测良好的代码
* 能够帮助开发者将接口和实现分离.因为首先编写测试会迫使一个开发者将代码实现为面向接口(否则不容易编写测试用例)

TDD的第三个步骤很关键,否则容易写出 messy aggregation of code fragments.

## 何时需要mock

如果完全不使用mock:
* 测试集的执行时间缓慢.有可能会调用大量的网络服务(数据库/rpc接口等等)
* 测试覆盖率低.一些错误条件或者异常可能测试不到
* 测试对系统敏感.可能系统负载高或者内存被其他人使用都容易影响测试结果

如果全部使用mock:
* 一些mock系统严重依赖于reflection,因此如果两个类之间交互时使用mock可能比直接交互还慢
* mock过多可能会使setup code异常复杂并且和具体代码实现紧密耦合
* mock过多会创建大量的interface类,创建目的仅仅是为了能够被mocking.导致过度抽象

所以建议在架构边界进行mock.例如mock数据库,web服务器,外部服务.

并且这会迫使你去思考边界在哪,并且将边界实现为多态.因此你可以将边界外的组件独立部署

>good architectures are inherently testable

martin fowler自己的习惯是不使用mocking tools,自己写边界的mock.因为mocking tools大部分有自己的domain languange,而且好多功能也用不着

## mock stub spy

接口:
```
interface Authorizer {
  public Boolean authorize(String username, String password);
}

```

stub:
用于测试已经授权或者未授权时的行为
```
public class AcceptingAuthorizerStub implements Authorizer {
  public Boolean authorize(String username, String password) {
	return true;
	}
}
```

spy:
spy用来监视caller的行为,涉及代码内部的逻辑.例如测试结尾通过验证authorizeWasCalled是true来验证authorize确实被调用.
spy监视的内部动作越多,测试集会越耦合于代码,形成fragile tests
```
public class AcceptingAuthorizerSpy implements Authorizer {
  public boolean authorizeWasCalled = false;

  public Boolean authorize(String username, String password) {
	authorizeWasCalled = true;
	return true;
  }
```

mock:
```
public class AcceptingAuthorizerVerificationMock implements Authorizer {
  public boolean authorizeWasCalled = false;

  public Boolean authorize(String username, String password) {
	authorizeWasCalled = true;
	return true;
  }

  public boolean verify() {
	return authorizedWasCalled;
  }
}
```

fake:
```
  public class AcceptingAuthorizerFake implements Authorizer {
	  public Boolean authorize(String username, String password) {
		return username.equals("Bob");
	  }
  }

```
martin fowler习惯是使用stub和spy,其他的基本不用


## mocks stubs区别

* 结果验证不同,stub是状态验证(state verification),mock是行为验证(behavior verification)
* 本质上是测试和设计结合层面的不同

test的四个流程:setup-exercise-verify-teardown

正常测试方法:从仓库中获取货物然后填充到订单中
```
public class OrderStateTester extends TestCase {
  private static String TALISKER = "Talisker";
  private static String HIGHLAND_PARK = "Highland Park";
  private Warehouse warehouse = new WarehouseImpl();

  protected void setUp() throws Exception {
    warehouse.add(TALISKER, 50);
    warehouse.add(HIGHLAND_PARK, 25);
  }
  public void testOrderIsFilledIfEnoughInWarehouse() {
    Order order = new Order(TALISKER, 50);
    order.fill(warehouse);
    assertTrue(order.isFilled());
    assertEquals(0, warehouse.getInventory(TALISKER));
  }
  public void testOrderDoesNotRemoveIfNotEnough() {
    Order order = new Order(TALISKER, 51);
    order.fill(warehouse);
    assertFalse(order.isFilled());
    assertEquals(50, warehouse.getInventory(TALISKER));
  }

```

jMock:
```
public class OrderInteractionTester extends MockObjectTestCase {
  private static String TALISKER = "Talisker";

  public void testFillingRemovesInventoryIfInStock() {
    //setup - data
    Order order = new Order(TALISKER, 50);
    Mock warehouseMock = new Mock(Warehouse.class);
    
    //setup - expectations
    warehouseMock.expects(once()).method("hasInventory")
      .with(eq(TALISKER),eq(50))
      .will(returnValue(true));
    warehouseMock.expects(once()).method("remove")
      .with(eq(TALISKER), eq(50))
      .after("hasInventory");

    //exercise
    order.fill((Warehouse) warehouseMock.proxy());
    
    //verify
    warehouseMock.verify();
    assertTrue(order.isFilled());
  }

  public void testFillingDoesNotRemoveIfNotEnoughInStock() {
    Order order = new Order(TALISKER, 51);    
    Mock warehouse = mock(Warehouse.class);
      
    warehouse.expects(once()).method("hasInventory")
      .withAnyArguments()
      .will(returnValue(false));

    order.fill((Warehouse) warehouse.proxy());

    assertFalse(order.isFilled());
  }
```
有几点不同:
* warehouse使用mock而不是真实的class
* setup阶段除了设置数据,还设置了warehouse期望被调用的方式以及返回值.最后通过verify方法验证是否按setup调用
* warehouse不使用assert验证.验证的是行为而不是状态

衍生出两种不同的流派:classic and mockist TDD

* classic setup阶段需要大量的协助对象创建.而mockist不需要
* mockist是test isolation,所有的第三方都是mock,测试不会因为第三方的bug而fragile.但classic test不只是一个单元测试,也是部分的集成测试
* mockist关注测试对象的行为,会和代码实现耦合更重
* mockist支持outside-in的方法,而关注domain model设计方式的更加倾向于classic testing


## 参考链接

* https://martinfowler.com/bliki/TestDrivenDevelopment.html
* https://blog.cleancoder.com/uncle-bob/2014/05/10/WhenToMock.html
* https://blog.cleancoder.com/uncle-bob/2014/05/14/TheLittleMocker.html
* https://martinfowler.com/articles/mocksArentStubs.html