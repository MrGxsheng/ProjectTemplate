### 依赖

```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-security</artifactId>
            <version>2.7.1</version>
        </dependency>

        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt-api</artifactId>
            <version>0.11.5</version>
        </dependency>

        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt-impl</artifactId>
            <version>0.11.5</version>
            <scope>runtime</scope>
        </dependency>

        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt-jackson</artifactId>
            <version>0.11.5</version>
            <scope>runtime</scope>
        </dependency>

        <dependency>
            <groupId>cn.hutool</groupId>
            <artifactId>hutool-all</artifactId>
            <version>5.8.5</version>
        </dependency>
```

<img src="https://hasdsd-markdown.oss-cn-beijing.aliyuncs.com/img/image-20230306165743794.png"/>

### SecurityConfig

```java
@RequiredArgsConstructor //用 final 代替AU注解
@EnableGlobalMethodSecurity(prePostEnabled = true) // 开启授权 之后可以使用@preAuthorize注解  并且可以省掉Config 注解

public class SecurityConfig extends WebSecurityConfigurerAdapter {
	
    
    private final JwtAuthenticationTokenFilter jwtAuthenticationTokenFilter;
    private final AuthenticationEntryPoint authenticationEntryPoint;
    private final AccessDeniedHandler accessDeniedHandler;

    // 使用这个加密密码
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    // 将authenticationManagerBean注入到容器中
    @Bean
    @Override
    public AuthenticationManager authenticationManagerBean() throws Exception {
        return super.authenticationManagerBean();
    }

    // 配置，我看不懂
    @Override
    protected void configure(HttpSecurity http) throws Exception {
//        super.configure(http);
        // 必要配置
        http.csrf().disable()
                .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS) // session设置 前后端用不上
                .and() // 结束上述设置
                .authorizeRequests() // 放行设置
                .antMatchers("/user/login", "/user/register").anonymous() // 未登录可访问
                .anyRequest().authenticated(); // 类 '/**' 全部拦截

        // 添加过滤器，过滤jwt伪造及过期
        http.addFilterBefore(jwtAuthenticationTokenFilter, UsernamePasswordAuthenticationFilter.class);

        // 添加异常处理
        http.exceptionHandling()
                .accessDeniedHandler(accessDeniedHandler)
                .authenticationEntryPoint(authenticationEntryPoint);

        // 允许跨域
        http.cors();
    }
}
```

### UserDetailsService

```java
@Service
@RequiredArgsConstructor
public class UserDetailServiceImpl implements UserDetailsService {

    private final IUserService userService;

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        LambdaQueryWrapper<User> wrapper = new LambdaQueryWrapper<>();
        wrapper.eq(User::getUsername, username);
        User user = userService.getOne(wrapper);
        if (Objects.isNull(user)) {
            throw new RuntimeException("用户名或密码错误");
        }
        						//如果需要传入多个参数的集合需要使用Arraylist
        return new LoginUser(user, Collections.singletonList("test")); // todo 从数据库中获取权限信息
    }
}
```

### UserDetailsImpl （LoginUser）

```java
@Data
@NoArgsConstructor
public class UserDetailsImpl implements UserDetails {

    private User user;
	
    //权限的集合
    private List<String> permissions;

    public UserDetailsImpl(User user) {
        this.user = user;
    }

    public UserDetailsImpl(User user, List<String> permissions) {
        this.user = user;
        this.permissions = permissions;
    

    //授权
    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        //函数式编程
           return 		permissions.stream().
               			map(SimpleGrantedAuthority::new).
               			collect(Collectors.toList());
    }

    // 获取密码
    @Override
    public String getPassword() {
        return user.getPassword();
    }

    // 获取用户名
    @Override
    public String getUsername() {
        return user.getUsername();
    }

    // 是否在期
    @Override
    public boolean isAccountNonExpired() {
        return true;
    }

    // true放行
    @Override
    public boolean isAccountNonLocked() {
        return true;
    }

    // true放行
    @Override
    public boolean isCredentialsNonExpired() {
        return true;
    }
    
	// 是否可用
    @Override
    public boolean isEnabled() {
        return true;
    }
}
```



### 关于生成token

基于登录

```java
    private final AuthenticationManager authenticationManager;
   
public String login(@RequestBody User user) {
        // 内部使用 UserDetailsService 查询用户，正确返回信息 错误返回空（大概）
        UsernamePasswordAuthenticationToken usernamePasswordAuthenticationToken = new UsernamePasswordAuthenticationToken(user.getUsername(), user.getPassword());
        // 取出刚才返回信息的内部存储的 UserDetails
        Authentication authenticate = authenticationManager.authenticate(usernamePasswordAuthenticationToken);

        if (Objects.isNull(authenticate)) {
            throw new RuntimeException("用户名或密码错误");
        }

        LoginUser principal = (LoginUser) authenticate.getPrincipal();
        System.out.println(principal);

        // 返回一个token，在 JwtAuthenticationTokenFilter 中根据这个token获取权限和信息
        return JwtUtil.createToken(principal.getUsername(), principal.getPassword());
    }
```

或者？

```java
    @Autowired
    private AuthenticationManager authenticationManager;

    @Override
    public Map<String, String> getToken(String username, String password) {
        // 先通过username 和 password 进行登录验证
        UsernamePasswordAuthenticationToken usernamePasswordAuthenticationToken = new UsernamePasswordAuthenticationToken(username,password);

        // 如果登陆失败，会自动处理
        Authentication authenticate = authenticationManager.authenticate(usernamePasswordAuthenticationToken);

        UserDetailsImpl loginUser = (UserDetailsImpl) authenticate.getPrincipal();
        User user = loginUser.getUser();
        String Jwt = JwtUtil.createJWT(user.getId().toString());

        Map<String , String > map = new HashMap<>();
        map.put("error_message","success");
        map.put("token",Jwt);
        return map;
    }
```

### JWT认证过滤

```java
@Component
public class JwtAuthenticationTokenFilter extends OncePerRequestFilter {
    @Autowired
    private UserMapper userMapper;

    @Override
    protected void doFilterInternal(HttpServletRequest request, @NotNull HttpServletResponse response, @NotNull FilterChain filterChain) throws ServletException, IOException {
        String token = request.getHeader("Authorization");
		
        //如果没有token的话直接放行
        if (!StringUtils.hasText(token)) {
            filterChain.doFilter(request, response);
            return;
        }


        String userid;
        try {
            //解析token
            Claims claims = JwtUtil.parseJWT(token);
            userid = claims.getSubject();
        } catch (Exception e) {
            throw new RuntimeException(e);
        }

        User user = userMapper.selectById(Integer.parseInt(userid));

        if (user == null) {
            throw new RuntimeException("用户名未登录");
        }

        //TODO 获取权限信息封装到Authenticetion
        UserDetailsImpl loginUser = new UserDetailsImpl(user);
        UsernamePasswordAuthenticationToken authenticationToken =
                new UsernamePasswordAuthenticationToken(loginUser, null, null);

        //存入SecurityContextHolder	
        SecurityContextHolder.getContext().setAuthentication(authenticationToken);

        //放行
        filterChain.doFilter(request, response);
    }
}
```

### 关于权限注解@preAuthorize的例子

```java
@RestController
public class HelloController{
	
	@RequestMapping("/hello")
	@PreAuthorize("hasAuthority('test')")
	public String hello){
		return "hello;
	}
}
```

### 自定义用户名和密码

```java
# 编写配置文件
spring:
    security:
        user:
            name: user
            password: 123456
```

### 自定义失败处理

如果是认证过程中出现的异常会被封装成**AuthenticationException**然后调用**AuthenticationEntryPoint**对象的方法去进行异常处理

如果是授权过程中出现的异常会被封装成**AccessDeniedException**然后调用**AccessDeniedHandler**对象的方法去进行异常处理

```java
@Component
public class AuthenticationEntryPointImpl implements AuthenticationEntryPoint {
    @Override
    public void commence(HttpServletRequest request, HttpServletResponse response, AuthenticationException authException) throws MyException {
        //ResponseResult result = new ResponseResult(HttpStatus.UNAUTHORIZED.value(),"用户名未验证");
        String json = JSONUtil.toJsonStr("认证失败");
        WebUtil.renderString(response, json);
    }
}
```

```java
@Component
public class AccessDeniedHandlerImpl implements AccessDeniedHandler {
    @Override
    public void handle(HttpServletRequest request, HttpServletResponse response, AccessDeniedException accessDeniedException) throws MyException {
        String json = JSONUtil.toJsonStr("权限不足");
        WebUtil.renderString(response, json);
    }
}
```

### 自定义权限检验方法

在SPEL表达式中使用@ex相当于获取容器中bean的名字为ex的对象，然后在调用这个对象的hasAuthority方法

@preAuthorize("@ex.hasAuthority('string str')") 

默认为类的名字(应该)

```java
@Component("xx")
public class Ex {
    public static boolean hasAuthority(String auth){
        Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
        LoginUser loginUser = (LoginUser) authentication.getPrincipal();
        // 我觉得可以这样写
        Collection<? extends GrantedAuthority> authorities = authentication.getAuthorities();
        // 他的写法
        List<String> permissions = loginUser.getPermissions();

        // todo 自定义验证

        return true;
    }
}

```

