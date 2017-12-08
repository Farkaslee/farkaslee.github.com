### 为什么要使用Bean Validation？

 当我们实现某个接口时，都需要对入参数进行校验

该方法输入的四个参数都是必填项。用代码进行参数验证带来几个问题
需要写大量的代码来进行参数验证。
需要通过注释来直到每个入参的约束是什么。
每个程序员做参数验证的方式不一样，参数验证不通过抛出的异常也不一样。

### Bean Validation是一个通过配置注解来验证参数的框架：

具体的bean：

```java

public class TradeMessageGson {
	
	@NotNull(message = "{message.userId.notNll}")
	private Long userId;
	
	@NotNull(message="{message.accountId.notNll}")
	private Long accountId;
	
	@NotEmpty(message="{message.prodCategory.notNll}")
	private String prodCategory;
	
	@NotNull(message="{message.productId.notNll}")
	private Long productId;

	@NotEmpty(message="{message.channelId.notNll}")
	private String channelId;

	@NotEmpty(message="{message.tradeType.notNll}")
	private String tradeType;
	
	@NotEmpty(message="{message.requestId.notNll}")
	private String requestId;
	
	@NotNull(message="{message.payload.notNll}")
	private Object payload;

	private String groupId;
  }
  
```
### 相应的验证方法
```java

import java.util.Iterator;
import java.util.Set;

import javax.validation.ConstraintViolation;
import javax.validation.Validator;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

@Service
public class ValidationService {

	@Autowired
	private Validator validator;

	public <T> void validate(T object) throws BizUncheckedException {
		Set<ConstraintViolation<T>> violations = validator.validate(object);
		if (!violations.isEmpty()) {
			StringBuilder result = new StringBuilder();
			Iterator<ConstraintViolation<T>> it = violations.iterator();
			while (it.hasNext()) {
				ConstraintViolation<T> violation = it.next();
				String errorMessage = violation.getMessage();
				if (result.length() > 0)
					result.append(",");
				result.append(errorMessage);
			}
			BizUncheckedException exception = new BizUncheckedException(ExceptionCode.VALIDATION_FAILURE, result.toString());
			throw exception;
		}
	}
}
```
指导参考
https://www.ibm.com/developerworks/cn/java/j-lo-jsr303/
