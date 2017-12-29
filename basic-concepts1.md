## Java中的equals和hashCode

quals()和hashCode()方法是用来在同一类中做比较用的，尤其是在容器里如set存放同一类对象时用来判断放入的对象是否重复。

这里我们首先要明白一个问题：
    equals()相等的两个对象，hashcode()一定相等，equals()不相等的两个对象，却并不能证明他们的hashcode()不相等。换句话说，equals()方法不相等的两个对象，hashCode()有可能相等。
	  
   在这里hashCode就好比字典里每个字的索引，equals()好比比较的是字典里同一个字下的不同词语。就好像在字典里查“自”这个字下的两个词语“自己”、“自发”，如果用equals()判断查询的词语相等那么就是同一个词语，比如equals()比较的两个词语都是“自己”，那么此时hashCode()方法得到的值也肯定相等；如果用equals()方法比较的是“自己”和“自发”这两个词语，那么得到结果是不想等，但是这两个词都属于“自”这个字下的词语所以在查索引时相同，即：hashCode()相同。如果用equals()比较的是“自己”和“他们”这两个词语的话那么得到的结果也是不同的，此时hashCode() 得到也是不同的
       
   反过来：hashcode()不等，一定能推出equals()也不等；hashcode()相等，equals()可能相等，也可能不等。在object类中，hashcode()方法是本地方法，返回的是对象的地址值，而object类中的equals()方法比较的也是两个对象的地址值，如果equals()相等，说明两个对象地址值也相等，当然hashcode() 也就相等了.
     

官方注释解释：   
    在 Java 应用程序执行期间，在同一对象上多次调用 hashCode 方法时，必须一致地返回相同的整数，前提是对象上 equals 比较中所用的信息没有被修改。从某一应用程序的一次执行到同一应用程序的另一次执行，该整数无需保持一致。     
如果根据 equals(Object) 方法，两个对象是相等的，那么在两个对象中的每个对象上调用 hashCode 方法都必须生成相同的整数结果。     
以下情况不 是必需的：如果根据 equals(java.lang.Object) 方法，两个对象不相等，那么在两个对象中的任一对象上调用 hashCode 方法必定会生成不同的整数结果。但是，程序员应该知道，为不相等的对象生成不同整数结果可以提高哈希表的性能。     
实际上，由 Object 类定义的 hashCode 方法确实会针对不同的对象返回不同的整数。（这一般是通过将该对象的内部地址转换成一个整数来实现的，但是 JavaTM 编程语言不需要这种实现技巧。）     

   当equals方法被重写时，通常有必要重写 hashCode 方法，以维护 hashCode 方法的常规协定，该协定声明相等对象必须具有相等的哈希码。

   实际中这在项目中经常被用到，下面是本人所经历的项目中的一个例子，

java```
@Service
public class HandlerRegistry {
	
	private Map<MapKeyPair<TradeTypeEnum, HandlerTypeEnum>, VoucherHandler> config = new HashMap<MapKeyPair<TradeTypeEnum, HandlerTypeEnum>, VoucherHandler>(6);
	
	public void addHandler(TradeTypeEnum tradeType, HandlerTypeEnum handlerType,VoucherHandler handler) {
		MapKeyPair<TradeTypeEnum, HandlerTypeEnum> key = new MapKeyPair<TradeTypeEnum, HandlerTypeEnum>(tradeType, handlerType);
		config.put(key, handler);
	}

	
	public VoucherHandler getHandler(TradeTypeEnum tradeType, HandlerTypeEnum handlerType) {
		MapKeyPair<TradeTypeEnum, HandlerTypeEnum> key = new MapKeyPair<TradeTypeEnum, HandlerTypeEnum>(tradeType, handlerType);
		return config.get(key);
	}
		
	
}


class MapKeyPair<V, K> {

	private V tradeType;

	private K handlerType;

	public MapKeyPair(V tradeType, K handlerType) {
		this.tradeType = tradeType;
		this.handlerType = handlerType;
	}

	public V getTradeType() {
		return tradeType;
	}

	public void setTradeType(V tradeType) {
		this.tradeType = tradeType;
	}

	public K getHandlerType() {
		return handlerType;
	}

	public void setHandlerType(K handlerType) {
		this.handlerType = handlerType;
	}

	@Override
	public int hashCode() {
		final int prime = 31;
		int result = 1;
		result = prime * result + ((handlerType == null) ? 0 : handlerType.hashCode());
		result = prime * result + ((tradeType == null) ? 0 : tradeType.hashCode());
		return result;
	}

	@Override
	public boolean equals(Object obj) {
		if (this == obj)
			return true;
		if (obj == null)
			return false;
		if (getClass() != obj.getClass())
			return false;
		MapKeyPair<?, ?> other = (MapKeyPair<?, ?>) obj;
		if (handlerType == null) {
			if (other.handlerType != null)
				return false;
		} else if (!handlerType.equals(other.handlerType))
			return false;
		if (tradeType == null) {
			if (other.tradeType != null)
				return false;
		} else if (!tradeType.equals(other.tradeType))
			return false;
		return true;
	}
}
```


