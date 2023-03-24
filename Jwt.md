### 焯焯焯

### 依赖

```xml
<dependency>
    <groupId>com.auth0</groupId>
    <artifactId>java-jwt</artifactId>
    <version>3.8.3</version>
</dependency>
```



### JwtInterceptor

拦截器

```java
package cn.sdadgz.web_springboot.config;

import cn.sdadgz.web_springboot.service.IUserService;
import cn.sdadgz.web_springboot.utils.IdUtil;
import cn.sdadgz.web_springboot.utils.JwtUtil;
import cn.sdadgz.web_springboot.utils.StrUtil;
import cn.sdadgz.web_springboot.entity.User;
import cn.sdadgz.web_springboot.mapper.UserMapper;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;
import org.springframework.web.servlet.HandlerInterceptor;
import org.springframework.web.servlet.ModelAndView;

import javax.annotation.Resource;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.util.Objects;

@Component
@Slf4j
public class JwtInterceptor implements HandlerInterceptor {

    @Resource
    private UserMapper userMapper;

    @Resource
    private IUserService userService;

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        // ip
        String ip = IdUtil.getIp(request);

        //放行OPTIONS请求
        // nginx拦过了，自信去掉
//        String method = request.getMethod().toUpperCase();
//        if ("OPTIONS".equals(method)) {
//            return true;
//        }

        //获取token
        String token = request.getHeader("token");
        //token不存在直接报错
        if (token == null) {
            log.error("访问者ip：{}，token不存在", ip);
            throw new BusinessException("499", "Token不存在");
        }
        //获取签发对象id，不存在直接报错
        String userid = null; //获取签发对象的Userid
        try {
            userid = JwtUtil.getAudience(token);//获取token中的签发对象
        } catch (Exception e) {
            log.error("访问者ip：{}，token校验失败", ip);
            throw new DangerousException("499", "数据校验失败", ip, StrUtil.NO_USER);
        }
        //带着token验证载荷username是否正确
        User realUser = null;
        realUser = userService.getUserById(Integer.parseInt(userid));
        if (realUser == null) { // 用户不存在
            log.error("访问者ip：{}，token认证错误", ip);
            throw new DangerousException("499", "认证错误", request, Integer.parseInt(userid));
        }
        //姓名不匹配返回
        if (!Objects.equals(realUser.getName(), JwtUtil.getClaimByName(token, "username").asString())) {
            log.error("访问者ip：{}，token认证错误", ip);
            throw new DangerousException("499", "认证错误", request, Integer.parseInt(userid));
        }
        //检查是否过期
        if (JwtUtil.checkDate(token)) {
            log.error("访问者ip：{}，token过期", ip);
            throw new BusinessException("499", "token过期");
        }
        //布尔值验证
        return JwtUtil.verifyToken(token, userid, realUser.getPassword());
    }

    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception {
        HandlerInterceptor.super.postHandle(request, response, handler, modelAndView);
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {
        HandlerInterceptor.super.afterCompletion(request, response, handler, ex);
    }
}
```



### JwtInterceptorConfig

拦截器配置

```java
package cn.sdadgz.web_springboot.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.InterceptorRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

import javax.annotation.Resource;

@Configuration
public class JwtInterceptorConfig implements WebMvcConfigurer {

    @Resource
    JwtInterceptor interceptor;

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(interceptor).addPathPatterns("/**").order(-1) // 似乎是小的先走
                .excludePathPatterns(
                        "/user/login", // 用户登录 *
                        "/user", // 用户注册 *
                        "/blog/*/blogs", // 博客s
                        "/blog/*/blog/*", // 博客
                        "/img/*/banner", // 主页banner
                        "/img/*/background", // 背景图片
                        "/file/sdadgz/page", // 我的仓库
                        "/ip", // 获取ip
                        "/toy/**", // 小玩具
                        "/ipBan/**", // 这个就当是压测nginx接口了
                        "/static/**");
    }
}
```

### JwtUtils

```java
import com.auth0.jwt.JWT;
import com.auth0.jwt.JWTVerifier;
import com.auth0.jwt.algorithms.Algorithm;
import com.auth0.jwt.interfaces.Claim;
import com.auth0.jwt.interfaces.DecodedJWT;
import java.util.Date;
import java.util.HashMap;
import java.util.Map;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

/**
 * JWT工具类
 *
 * @author lcr
 */
public class JwtUtil {

  /**
   * 日志
   */
  private static final Logger logger = LoggerFactory.getLogger(JwtUtil.class);
  /**
   * 密钥
   */
  private static final String SECRET = "my_secret";

  /**
   * 过期时间 单位为秒
   **/
  private static final long EXPIRATION = 1800L;

  /**
   * 生成用户token,设置token超时时间
   */
  public static String createToken() {
    //过期时间
    Date expireDate = new Date(System.currentTimeMillis() + EXPIRATION * 1000);
    // 存放head值
    Map<String, Object> map = new HashMap<>();
    map.put("alg", "HS256");
    map.put("typ", "JWT");
    // 加密方式 header payload采用的BASE64加密方式（对称加密） secret采用HMAC（哈希加密）
    String token = JWT.create()
        // 添加头部
        .withHeader(map)
        //可以将基本信息放到claims中
        // 添加载荷PAYLOAD
        // 用户编号
        .withClaim("id", "0001")
        // 用户名称
        .withClaim("userName", "lcr")
        // 超时设置,设置过期的日期
        .withExpiresAt(expireDate)
        // 签发时间
        .withIssuedAt(new Date())
        // sign签名 SECRET加密 采用散列加密
        .sign(Algorithm.HMAC256(SECRET));
    return token;
  }

  /**
   * 校验token并解析token
   */
  public static Map<String, Claim> verifyToken(String token) {
    DecodedJWT jwt = null;
    try {
      // 校验header payload signature是否合法 校验过期时间
      JWTVerifier verifier = JWT.require(Algorithm.HMAC256(SECRET)).build();
      jwt = verifier.verify(token);
    } catch (Exception e) {
      logger.error(e.getMessage());
      logger.error(" 签名验签失败 !");
      //解码异常则抛出异常
      return null;
    }
    return jwt.getClaims();
  }

}

```

