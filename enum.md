### ENUM

```java
package enumtest;

/**
 * 性别枚举
 */

public enum SexEnum {
    BOY(0,"男"),
    GIRL(1,"女"),
    OTHERS(2,"保密");

    private Integer code;
    private String description;

    SexEnum(Integer code, String description) {
        this.code = code;
        this.description = description;
    }

    public Integer getCode() {
        return code;
    }

    public void setCode(Integer code) {
        this.code = code;
    }

    public String getDescription() {
        return description;
    }

    public void setDescription(String description) {
        this.description = description;
    }
}

```

### 调用

```java
	SexEnum.BOY.getCode();
	SexEnum.BOY.getDescription();
```

