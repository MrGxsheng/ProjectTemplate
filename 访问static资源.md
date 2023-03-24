### 配置文件

```yaml
  mvc:
    static-path-pattern: '/static/**' # 静态资源访问
  web:
    resources:
      static-locations: file:'http://localhost:9000/static/img/'
```

### MyWebMvcConfig

```java
package com.te_springboot.config;

import org.springframework.stereotype.Component;
import org.springframework.web.servlet.config.annotation.ResourceHandlerRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

/**
 * ??不懂
 *
 * <p>
 * fw练习生
 * </p>
 *
 * @author sheng
 * @since 2023/3/21 16:37:13
 */

@Component
public class MyWebMvcConfig implements WebMvcConfigurer {
    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        registry.addResourceHandler("/static/**")
                .addResourceLocations("classpath:/static/");
    }
}
```

