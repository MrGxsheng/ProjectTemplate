### 依赖

```xml
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
```

### Result

```java
package com.te_springboot.common;

import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

/**
 * 结果集
 *
 * <p>
 * fw练习生
 * </p>
 *
 * @author sheng
 * @since 2023/3/16 16:49:26
 */


@Data
@AllArgsConstructor
@NoArgsConstructor
public class Result {
    private String code;
    private String msg;
    private Object data;
    // 不需要返回值
    public static Result success() {
        return new Result(Constants.CODE_200, "操作成功", null);
    }
    // 需要返回值
    public static Result success(Object data) {
        return new Result(Constants.CODE_200, "", data);
    }
    // 需要返回状态
    public static Result error(String code, String msg) {
        return new Result(code, msg, null);
    }
    // 不需要返回状态
    public static Result error() {
        return new Result(Constants.CODE_500, "系统错误", null);
    }
}



```



### Constants

```java
package com.te_springboot.common;

/**
 * 结果集
 *
 * <p>
 * fw练习生
 * </p>
 *
 * @author sheng
 * @since 2023/3/16 16:52:01
 */

public interface Constants {
    String CODE_200 = "200";    //成功
    String CODE_400 = "400";    //参数错误
    String CODE_401 = "401";    // 用户的锅
    String CODE_404 = "404";    //未找到
    String CODE_457 = "457";    // 上传文件异常
    String CODE_465 = "465";    // 重复的文章标题
    String CODE_485 = "485";    // 用户名或密码错误
    String CODE_498 = "498";    // 权限不足
    String CODE_499 = "499";    // token认证失败
    String CODE_500 = "500";    //服务器内部错误
}

```

