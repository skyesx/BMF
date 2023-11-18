## 总目标

我们有各类处理技术复杂度问题的框架，但为什么没有一个处理业务复杂度的框架呢？本框架致力于形成一套通用的业务逻辑模块化框架，并使得业务开发更为便捷高效。

但我们的框架能提供什么能力，其实不太好理解，需要由一定的编程经验，接下来我从一个普通的业务代码的不断重构过程来说明本框架计划解决以及是怎么解决这些问题的。

### 阶段一、所有业务代码都贯穿于业务代码流程中

举个电商的例子，下单时

*   充值服务有无限库存，因此不需要查询库存信息
*   但普通实物商品需要查询库存信息

    那么阶段一的代码就是在判断是否需要查询库存的逻辑位置，写上

```java
class CheckoutQueryInventoryActivity{

	private InventoryService inventoryService;

	public boolean checkInventorySufficent(Order order){
		if(!needSourcingInventory(order.getProduct())){
			return true;
		}

		SourcingResult result = inventoryService.sourcingInventory(order);	
		return result.isSufficent();
	}

	
	public boolean needSourcingInventory(Product product){
		boolean queryInventory = true;
		if(isTopup(product)){
			queryInventory = false;
		}
		return queryInventory;
	}
}


```

这里的代码问题是什么呢，有以下几点：

*   没有遵守开闭原则，新增一种商品类型是一个高频修改类型，但每次新增类型都要修改本段代码
*   充值服务要定制很多点，例如还要定制履约的实现（普通商品走物流，充值服务对接电信服务商）等等，这些代码散布在代码流程的各个位置，我们很难清楚快速地从代码中了解 充值业务 有什么特性

### 阶段二、使用接口完成开闭改造，将同一个业务的逻辑集中

针对上面的问题，我们可以简单的改造一下

```java
class CheckoutQueryInventoryActivity{

	public static interface Extend{
		//return null means this extend can not check this product
		Boolean needSourcingInventory(Product product);
	}

	private inventoryService;
	private List<Extend> extends;

	public boolean checkInventorySufficent(Order order){
		
		boolean needSourcing=
			extends.stream.map(e->e.needSourcingInventory(order.getProduct))
				.filter(result->result != null)
				.findFirst().orElse(true);
		if(!needSourcing){
			return true;
		}

		SourcingResult result = inventoryService.sourcingInventory(order);	
		return result.isSufficent();
	}
}

class Topup extends Business implements 
CheckoutQueryInventoryActivity.Extend, CheckoutFulfillmentActivity.Extend,{
	public Boolean needSourcingInventory(Product product){
		if(isTopup(product)){ return false; }
		else { return null;}
	}
}
```

我们从业务代码可以看到，上面两个问题已经解决了：

*   新增一个一个类似Topup的业务，无需改动CheckoutQueryInventoryActivity，只需要额外新增一个Business的实现即可
*   同时，业务自身所有定制的逻辑都收纳到了同一个类里面，我们可以快速地看到这个业务定制了什么功能

但这里的实现还有问题，其实决定的是否要查库存的，可能不止商品类型，充值业务可能有一些优惠活动，假设叫 抢购，其是有预算控制/次数控制的，那么此时不能不查库存，要查询一个 优惠库存（或者说渠道库存）。

理论上这个充值业务除了叠加这个 “抢购”活动外，可还可能叠加各种各样的业务规则，例如有一些供应商就是充值的总次数有限制要设置库存，有一些地区就是跟其他的地区不一样，就是要设置库存，可能控制查库存的场景可能可以叠加很多，那究竟要用哪一个来作最终决策呢?

以上仅仅描述了充值业务，但实际上还有 普通物流商品、酒店业务、机票业务 各种各样的业务，他们都会共用上面同一个扩展点，这些业务自身可能也还会叠加一些特定的外场景，那按照之前的实现，且不论执行效率，我们自己还能愉快地排查问题么

### 阶段三、找到扩展点逻辑分割的维度，各自独自处理冲突

从上面的描述大家可以发现，普通商品、酒店、机票、充值这些维度它是不可能叠加出现的，但优惠活动、服务区域等等实际上是可叠加到前述商品中的。同时，能最终决定 充值业务自身 在不同叠加场景下如何表现的，只能是充值业务自己的团队。因此我们引入两个概念，扩展点主业务，以及扩展点叠加业务的概念

*   主业务：主业务间要做到 互斥、且不重不漏（MECE），但叠加业务可叠加到主业务上，叠加的最终表现有主业务自身指定
*   叠加业务：可叠加到主业务上的业务

```java
class CheckoutQueryInventoryActivity{

	public static interface Extend{
		//return null means this extend can not check this product
		Boolean needSourcingInventory(Product product);
	}

	private inventoryService;
	private List<Extend> verticalExtends;

	public boolean checkInventorySufficent(Order order){
		
		boolean needSourcing=
			verticalExtends.stream.map(e->e.needSourcingInventory(order.getProduct))
				.filter(result->result != null)
				.findFirst().orElse(true);
		if(!needSourcing){
			return true;
		}

		SourcingResult result = inventoryService.sourcingInventory(order);	
		return result.isSufficent();
	}
}

class Topup extends VerticalBusiness implements CheckoutQueryInventoryActivity.Extend, CheckoutFulfillmentActivity.Extend{

	private List<HorizontalBusiness> overlyingBusiness = Arrays.asList(FlashSale.getInstance(),Region.getInstance());

	public Boolean needSourcingInventory(Product product){
		if(!isTopup(product)){ return null; }

		return overlyingBusiness.stream()
			.filter(b->b instanceof CheckoutQueryInventoryActivity.Extend)
			.map(b->(CheckoutQueryInventoryActivity.Extend)b)
			.map(b->b.needSourcingInventory(product))
			.filter(isNeed->isNeed != null && isNeed)
			.findAny().orElse(false);
	}
}
```

以上实现一定层度解决了扩展点实现爆炸的情况下，难以调试，复杂易错的问题（若还有复杂度问题或者其他原因可以垂直业务扩展点内再套用垂直业务扩展点），但我们还有场景问题，以上我们使用了商品的业务类型作为主业务，但这总是所有扩展点最好的选择么

很明显，其具有适用场景的区分，例如在优惠域其可能倾向于不同的优惠工具作为主业务划分的依据，而支付域则可能倾向于使用支付渠道作为主业务的划分。因为这样才能更好更均匀地分割扩展点实现

同时，在交易里作为主业务的 产品类型 其在 优惠域、支付域 也仅仅只是一个叠加业务而已。

### 阶段四、不同场景下使用不同主业务的分类

因此，我们应该允许一个业务在不同的域/扩展点钟切换 主业务/叠加业务 的身份

```java
class CheckoutQueryInventoryActivity{

	public static interface Extend{
		//return null means this extend can not check this product
		Boolean needSourcingInventory(Product product);
	}

	private inventoryService;

	//只注入VerticalBusiness<ProductType>类型的CheckoutQueryInventoryActivityExtend实现
	private List<Extend> verticalExtends;

	public boolean checkInventorySufficent(Order order){
		boolean needSourcing=
			verticalExtends.stream.map(e->e.needSourcingInventory(order.getProduct))
				.filter(result->result != null)
				.findFirst().orElse(true);
		if(!needSourcing){
			return true;
		}

		SourcingResult result = inventoryService.sourcingInventory(order);	
		return result.isSufficent();
	}
}

class Topup extends VerticalBusiness<ProductType> implements CheckoutQueryInventoryActivity.Extend, CheckoutFulfillmentActivity.Extend,{
		
	private List<HorizontalBusiness> overlyingBusiness = Arrays.asList(FlashSale.getInstance(),Region.getInstance());

	public Boolean needSourcingInventory(Product product){
		if(!isTopup(product)){ return null; }

		return overlyingBusiness.stream()
			.filter(b->b instanceof CheckoutQueryInventoryActivity.Extend)
			.map(b->(CheckoutQueryInventoryActivity.Extend)b)
			.map(b->b.needSourcingInventory(product))
			.filter(isNeed->isNeed != null && isNeed)
			.findAny().orElse(false);
	}
}

class VerticalBusinessType{}

//代表根据产品类型做主业务划分
class ProductType extends VerticalBusinessType{}

```

如上，我们给VerticalBusiness加上一个泛化参数，生成一个业务类的时候，我们继承自由主业务分类规则所具化的VerticalBusiness。

这样在例如CheckoutQueryInventoryActivity的实例初始化verticalExtends的时候，就可以选择特定的主业务的实现注入进去了

按照惯例，这里还是会有问题。class Topup如果实现了很多扩展点，那么Topup这个类就会变得超级大，甚至于难以阅读

### 阶段五、使用场景化的业务对象进行扩展点实现的分类

将Topup这个类按照场景拆分变成不同对象，TopupCart、TopupCheckout，TopupPayment，TopupFulfillment以场景的流程分割业务对象包含的逻辑 以及减少单个对象的复杂度

但如果我们想要快速复用类似Topup的功能，做一个水电煤缴费，那应该怎么办？

### 阶段六、生成业务模板，快速复制相似主业务，并可作细微调整

使用JAVA原生的继承关系实现，叠加业务、主业务均可用相似形式实现

```java
class CheckoutQueryInventoryActivity{

	public static interface Extend{
		//return null means this extend can not check this product
		Boolean needSourcingInventory(Product product);
	}

	private inventoryService;
	private List<Extend> verticalExtends;

	public boolean checkInventorySufficent(Order order){
		boolean needSourcing=
			verticalExtends.stream.map(e->e.needSourcingInventory(order.getProduct))
				.filter(result->result != null)
				.findFirst().orElse(true);
		if(!needSourcing){
			return true;
		}

		SourcingResult result = inventoryService.sourcingInventory(order);	
		return result.isSufficent();
	}
}

abstract class Digital extends VerticalBusiness<ProductType> implements CheckoutQueryInventoryActivity.Extend, CheckoutFulfillmentActivity.Extend,{
		
	private List<HorizontalBusiness> overlyingBusiness = Arrays.asList(FlashSale.getInstance(),Region.getInstance());

	public abstract boolean isBusinessMatch(Product product);

	public Boolean needSourcingInventory(Product product){
		if(!isBusinessMatch(product)){ return null; }

		return overlyingBusiness.stream()
			.filter(b->b instanceof CheckoutQueryInventoryActivity.Extend)
			.map(b->(CheckoutQueryInventoryActivity.Extend)b)
			.map(b->b.needSourcingInventory(product))
			.filter(isNeed->isNeed != null && isNeed)
			.findAny().orElse(false);
	}
}

class Topup extends Digital{
	@override
	public boolean isBusinessMatch(Product product){
		return isTopup(product);
	}
}

class Utility extends Digital{
	@override
	public boolean isBusinessMatch(Product product){
		return isTopup(product);
	}
}

class VerticalBusinessType{}

//代表根据产品类型做主业务划分
class ProductType extends VerticalBusinessType{}

```



这里有一个问题，主业务、叠加业务 自身的参数/上下文应该如何传递？

### 阶段七、业务上下文强类型化

* 首先应该是一个强类型对象，不能是一个MAP
* 这个强对象应该跟正整个场景处理过程维护一个总的，
* 还是每个扩展点都自行维护一个？
* 


```java
class CheckoutQueryInventoryActivity{

	public static interface Extend{
		//return null means this extend can not check this product
		Boolean needSourcingInventory(Product product);
	}

	private inventoryService;
	private List<Extend> verticalExtends;

	public boolean checkInventorySufficent(Order order){
		boolean needSourcing=
			verticalExtends.stream.map(e->e.needSourcingInventory(order.getProduct))
				.filter(result->result != null)
				.findFirst().orElse(true);
		if(!needSourcing){
			return true;
		}

		SourcingResult result = inventoryService.sourcingInventory(order);	
		return result.isSufficent();
	}
}

abstract class Digital extends VerticalBusiness<ProductType> implements CheckoutQueryInventoryActivity.Extend, CheckoutFulfillmentActivity.Extend,{
		
	private List<HorizontalBusiness> overlyingBusiness = Arrays.asList(FlashSale.getInstance(),Region.getInstance());

	public abstract boolean isBusinessMatch(Product product);

	public Boolean needSourcingInventory(Product product, BusinessContext context){
		if(!isBusinessMatch(product)){ return null; }
		
		DigitalContext digitalContext = context.getSubContext(Digital.getInstance().getBusinessContext());
		if(digitalContext.getEmail() != null) {
		    return true;
		}

		return overlyingBusiness.stream()
			.filter(b->b instanceof CheckoutQueryInventoryActivity.Extend)
			.map(b->(CheckoutQueryInventoryActivity.Extend)b)
			.map(b->b.needSourcingInventory(product,context))
			.filter(isNeed->isNeed != null && isNeed)
			.findAny().orElse(false);
	}
	
	public Class<?> getBusinessContext(){
	    return DigitalContext.class;
	}
	
	public static class DigitalContext implements BusinessContext{
	    private String email;
	    private String phone;
	}
}

class Topup extends Digital{
	@override
	public boolean isBusinessMatch(Product product){
		return isTopup(product);
	}
}

class Utility extends Digital{
	@override
	public boolean isBusinessMatch(Product product){
		return isUtility(product);
	}
}

class VerticalBusinessType{}

//代表根据产品类型做主业务划分
class ProductType extends VerticalBusinessType{}

```




随着业务越来越深入复杂，一个大系统里的不同域可能需要拆分，例如同一个业务在 搜索、详情页、下单、支付、履约等等位置都有定制，但其都归属于不同的系统。我们要如何像之前一样，快速集中地看到某一个业务具体是怎样的？

### 阶段八、构建全局业务可视化系统

构建独立的全局业务可视化系统，可视化各个域各个场景流程，并将流程中的扩展点予以展示。

同时也可以在场景流程中展示业务的全局样貌。

最后这个可视化系统形成后：

*   即使是业务/产品也能够快速知道全系统链路里，当前有什么可定制点，快速理清变化点，设计出业务全貌
*   开发基于 业务/产品 上述的梳理，也能够直接找到代码修改点，无需额外的逻辑转换（DDD目标之一）
*   能可视化地、迅速地排查出某个业务与其他业务的逻辑冲突点，帮助提前快速发现问题

解决了业务可视化问题后，我们就需要解决全局业务的 运维问题了

### 阶段九、构建全局业务的发布、灰度、统计系统
通过类动态加载、框架埋点等手段，可以实现业务与平台的发布分离、全局业务维度的灰度放量测试以及做到业务维度的运维统计


