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
```



### SecurityConfig

```java
//@RequiredArgsConstructor
//@EnableGlobalMethodSecurity(prePostEnabled = true) // 开启授权 之后可以使用@preAuthorize注解
@Configuration
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {
	
    //不行的话 去掉final 加@Au注释
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
        return new LoginUser(user, Collections.singletonList("test")); // todo 从数据库中获取权限信息
    }
}
```

### UserDetailsSerciveImpl

```java
@Data
@NoArgsConstructor
@AllArgsConstructor
public class UserDetailsImpl implements UserDetails {

    private User user;

    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        return null;
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

