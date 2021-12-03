
# Spring Security用户名密码登陆和token授权登陆两种实现 #

最近研究了一下Spring boot的web工程里通过Spring security做登陆验证。因为要满足授权登陆和用户名密码登陆两种方式，因此有一些配置和自定义的验证方式需要添加。这里简单说一下，主要留着备忘，之后有继续对这个框架研究再继续完善这份文档。

简单先说说spring security对于登陆验证的流程。

security本身带有一系列的拦截器，对于web资源的请求，都会根据security的配置被拦截器所拦截。

如果这个请求不带有授权信息，那么会被拦截，跳转到指定的页面。

授权信息是在用户登陆的过程当中生成的。我们将将怎么处理用户的登陆请求，并且生成登陆授权信息。

**配置文件**

配置文件先给出来：

```
@Configuration
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    @Value("${http.login.fail.url}")
    private String loginFailUrl;

    @Value("${http.logout.success.url}")
    private String logoutSuccessUrl;

    @Value("${http.white.url.active}")
    private String whiteActiveUrl;

    @Autowired
    private Environment env;
   
    
    /**
     * 此处给AuthenticationManager添加登陆验证的逻辑。
     * 这里添加了两个AuthenticationProvider分别用于用户名密码登陆的验证以及token授权登陆两种方式。
     * 在处理登陆信息的过滤器执行的时候会调用这两个provider进行登陆验证。
     */
    @Autowired
    public void configGlobal(AuthenticationManagerBuilder auth) throws Exception {
    	auth.authenticationProvider(customAuthenticationProvider())
            .authenticationProvider(tokenAuthenticationProvider())
            .eraseCredentials(true);
    }
    
    //用户名和密码登陆处理
    @Bean
    public CustomAuthenticationProvider customAuthenticationProvider() {
        return new CustomAuthenticationProvider();
    }
    
    //token登陆处理
    @Bean
    public TokenAuthenticationProvider tokenAuthenticationProvider() {
    	return new TokenAuthenticationProvider();
    }

    /**
     * 添加token登陆验证的过滤器
     */
    @Bean
    public TokenAuthenticationFilter tokenAuthenticationFilter() throws Exception {
    	TokenAuthenticationFilter filter = new TokenAuthenticationFilter();
        filter.setAuthenticationManager(authenticationManager());
        return filter;
    }
    
    /**
     * 添加用户名和密码登陆验证的过滤器
     */
    @Bean
    public CustomAuthenticationFilter customAuthenticationFilter() throws Exception {
    	CustomAuthenticationFilter filter = new CustomAuthenticationFilter();
        filter.setAuthenticationManager(authenticationManager());
        return filter;
    }

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.csrf().disable();

        String[] whiteActiveUrls = whiteActiveUrl.split(",");
        List<String> whiteUrls = new ArrayList<String>();

        for (int i = 0; i < whiteActiveUrls.length; i++) {
            String item = whiteActiveUrls[i];
            item = env.getProperty("http.white.urls." + item);
            if (StringUtils.isEmpty(item)) {
                continue;
            }
            whiteUrls.add(item);
        }
        whiteUrls.add("/static/**");

        // 白名单
        http.authorizeRequests().antMatchers(whiteUrls.toArray(new String[whiteUrls.size()])).permitAll().requestMatchers(CorsUtils::isPreFlightRequest);
        http.headers().frameOptions().disable();

        // 定义登陆成功之后的页面跳转
  		http.formLogin().loginPage("/login").defaultSuccessUrl("/index").successForwardUrl("/index");
        
        //分别添加 token 过滤器和 用户名密码过滤器。
        http.addFilterBefore(tokenAuthenticationFilter(), UsernamePasswordAuthenticationFilter.class);
        http.addFilterBefore(customAuthenticationFilter(), UsernamePasswordAuthenticationFilter.class);
        
        // 登出相关url
        http.logout().logoutSuccessUrl(logoutSuccessUrl);
		// 要求对request都进行是否授权的验证
        http.authorizeRequests().anyRequest().authenticated();
    }

}
```

通过配置文件可见定义了两个登陆验证的过滤器，过滤器会对我们指定的url进行拦截，通过我们定义的验证逻辑判断登陆信息是否正确。如果正确将授权信息添加到服务器。

**用户名和密码登陆**

先来看看用户名和密码的登陆逻辑。其实spring security本身是有用户名和密码验证的过滤器。要实现基本继承原本的逻辑即可。

先定义一个AuthenticationToken，他的实例包含了用户登陆时的信息，封装后用于校验。

```
public class CustomAuthenticationToken extends UsernamePasswordAuthenticationToken {

    /**
     * 
     */
    private static final long serialVersionUID = -1076492615339314113L;

    /**
     * {@inheritDoc}
     */
    public CustomAuthenticationToken(Object principal, Object credentials) {
        super(principal, credentials);
    }

    /**
     * {@inheritDoc}
     */
    public CustomAuthenticationToken(Object principal, Object credentials,
            Collection<? extends GrantedAuthority> authorities) {
        super(principal, credentials, authorities);
    }

}
```

过滤器的定义

```
public class CustomAuthenticationFilter extends UsernamePasswordAuthenticationFilter {


    public CustomAuthenticationFilter() {
        //父类中定义了拦截的请求URL，/login的post请求，直接使用这个配置，也可以自己重写
        super(); 
        //添加了自定义的登陆失败处理器，配置文件中直接配置failureUrl没能直接生效
        super.setAuthenticationFailureHandler(new LoginFailureHandler());
    }

    /**
     * 这里主要是把 request中的用户名和密码参数取出来，封装CustomAuthenticationToken，然后getAuthenticationManager().authenticate(authRequest)进行校验
     */
    @Override
    public Authentication attemptAuthentication(HttpServletRequest request, HttpServletResponse response)
    	throws AuthenticationException {
    	
    	if (!request.getMethod().equals(HttpMethod.POST.name())) {
    		throw new AuthenticationServiceException("Authentication method not supported: " + request.getMethod());
        }

    	String username = obtainUsername(request).trim();
    	String password = obtainPassword(request).trim();
    	final int userNameLen = 32;
        if (username.length() == 0 || username.length() > userNameLen || password.length() == 0
                || password.length() > userNameLen) {
            throw new BadCredentialsException("username or password is wrong!");
        }

        CustomAuthenticationToken authRequest = new CustomAuthenticationToken(username, password);
        setDetails(request, authRequest);
        Authentication authentication = getAuthenticationManager().authenticate(authRequest);
        return authentication;

    }

    @Override
    protected String obtainUsername(HttpServletRequest request) {
        String username = super.obtainUsername(request);
        return username == null ? "" : username;
    }

    /**
     * {@inheritDoc}
     */
    @Override
    protected String obtainPassword(HttpServletRequest request) {
        String password = super.obtainPassword(request);
        return password == null ? "" : password;
    }

}
```

具体的校验逻辑定义

```
public class CustomAuthenticationProvider implements AuthenticationProvider {

    @Autowired
    private UserServiceFeignClient userServiceFeignClient;

    // 根用户拥有全部的权限
    private final List<GrantedAuthority> authorities = Arrays.asList(new SimpleGrantedAuthority("CAN_SEARCH"), new SimpleGrantedAuthority("CAN_EXPORT"), new SimpleGrantedAuthority("CAN_IMPORT"), new SimpleGrantedAuthority("CAN_BORROW"), new SimpleGrantedAuthority("CAN_RETURN"), new SimpleGrantedAuthority("CAN_REPAIR"), new SimpleGrantedAuthority("CAN_DISCARD"), new SimpleGrantedAuthority("CAN_EMPOWERMENT"), new SimpleGrantedAuthority("CAN_BREED"));

    //这里通过用户名和密码的检查匹配判断登陆是否成功。
    @Override
    public Authentication authenticate(Authentication authentication) throws AuthenticationException {
    	if (authentication.isAuthenticated()) {
    		return authentication;
    	}
    	
    	String username = authentication.getName();
        User user = null;
        try {
        	user = userServiceFeignClient.getUserByUsername(username);
        } catch (BusinessException e) {
        	 throw new BadCredentialsException("该用户不存在，用户名: " + username);
        }
        if (user == null) {
            throw new BadCredentialsException("该用户不存在，用户名: " + username);
        }
        String password;
		try {
			password = encode((String)authentication.getCredentials());
		} catch (NoSuchAlgorithmException e) {
			throw new BadCredentialsException("密码错误");
		}
        if (!password.equals(user.getPassword())) {
            throw new BadCredentialsException("密码错误");
        }

        //这个是自定义用户登陆信息
        SecurityUser securityUser = new SecurityUser();

        securityUser.setUsername(user.getUsername());
        securityUser.setPassword(user.getPassword());
        securityUser.setUid(user.getUid());
		
        
        return new CustomAuthenticationToken(securityUser, authentication.getCredentials(), authorities);
    }

    public static String encode(String str) throws NoSuchAlgorithmException{
		MessageDigest instance = MessageDigest.getInstance("MD5");
		byte[] digest = instance.digest(str.getBytes());
		StringBuffer sb = new StringBuffer();
		for (byte b : digest) {
			int j = b & 0xff;
			String hexString = Integer.toHexString(j);
			if (hexString.length() < 2) {
				hexString = "0" + hexString;
			}
			sb.append(hexString);
		}
		return sb.toString();
	}
    
    //这里定义provider是否被调用，需要执行结果为true才会执行验证逻辑
    @Override
    public boolean supports(Class<?> authentication) {
    	return CustomAuthenticationToken.class.isAssignableFrom(authentication);
    }

}
```

这个就是用户名和密码登陆校验的逻辑。

而token登陆的逻辑基本一致，只是校验token即可，不校验用户名密码，下面直接把代码粘贴出来。

**token授权**

AuthenticationToken的定义

```
public class TokenAuthenticationToken extends UsernamePasswordAuthenticationToken {

    private static final long serialVersionUID = -6231962326068951783L;


    public TokenAuthenticationToken(Object principal) {
        super(principal, "");
    }


    public TokenAuthenticationToken(Object principal, Collection<? extends GrantedAuthority> authorities) {
        super(principal, "", authorities);
    }

}
```

FIlter定义

```
public class TokenAuthenticationFilter extends AbstractAuthenticationProcessingFilter {


    private String tokenParameter = "token";


    
    /**
     * @param defaultFilterProcessesUrl
     */
    public TokenAuthenticationFilter() {
        super("/quicklogin");
        super.setAuthenticationFailureHandler(new LoginFailureHandler());
    }

    /**
     * {@inheritDoc}
     */
    @Override
    public Authentication attemptAuthentication(HttpServletRequest request, HttpServletResponse response)
    	throws AuthenticationException {
    	
    	if (!request.getMethod().equals(HttpMethod.POST.name())) {
    		throw new AuthenticationServiceException("Authentication method not supported: " + request.getMethod());
        }

    	String token = obtainToken(request);
        if (token == null || token.length() == 0) {
        	throw new BadCredentialsException("uid or token is null.");
        }

        TokenAuthenticationToken authRequest = new TokenAuthenticationToken(token);

        authRequest.setDetails(authenticationDetailsSource.buildDetails(request));
  
        return this.getAuthenticationManager().authenticate(authRequest);

    }

    protected String obtainToken(HttpServletRequest request) {
        String token = request.getParameter(this.tokenParameter);
        return token == null ? "" : token.trim();
    }

}
```

authProvider的定义

```
public class TokenAuthenticationProvider implements AuthenticationProvider {

    @Autowired
    private UserServiceFeignClient userServiceFeignClient;
	
    // 根用户拥有全部的权限
    private final List<GrantedAuthority> authorities = Arrays.asList(new SimpleGrantedAuthority("CAN_SEARCH"), new SimpleGrantedAuthority("CAN_EXPORT"), new SimpleGrantedAuthority("CAN_IMPORT"), new SimpleGrantedAuthority("CAN_BORROW"), new SimpleGrantedAuthority("CAN_RETURN"), new SimpleGrantedAuthority("CAN_REPAIR"), new SimpleGrantedAuthority("CAN_DISCARD"), new SimpleGrantedAuthority("CAN_EMPOWERMENT"), new SimpleGrantedAuthority("CAN_BREED"));

    
	@Override
	public Authentication authenticate(Authentication authentication) throws AuthenticationException {
		
		if (authentication.isAuthenticated()) {
			return authentication;
		}
		//获取过滤器封装的token信息
		TokenAuthenticationToken authenticationToken = (TokenAuthenticationToken) authentication;
		User user = userServiceFeignClient.getUserTokenByToken((String)authenticationToken.getPrincipal());
//        不通过
		if (user == null) {
        	 throw new BadCredentialsException("授权token无效，请重新登陆");
        }
        SecurityUser securityUser = new SecurityUser();

        securityUser.setUsername(user.getUsername());
        securityUser.setPassword(user.getPassword());
        securityUser.setUid(user.getUid());
        
        TokenAuthenticationToken authenticationResult = new TokenAuthenticationToken(securityUser, authorities);

        return authenticationResult;
	}

	@Override
	public boolean supports(Class<?> authentication) {
		return TokenAuthenticationToken.class.isAssignableFrom(authentication);
	}

}
```

登陆失败信息的获取
前文我们自定义了一个登陆失败的handler如下。只是定义了失败后跳转的URL。

```
@Component
public class LoginFailureHandler extends SimpleUrlAuthenticationFailureHandler {
	public LoginFailureHandler() {
		super("/login?error=true");
	}
}
```

前端获取登陆失败的信息

```
<p th:if="${param.error}" th:text="${session?.SPRING_SECURITY_LAST_EXCEPTION?.message}" ></p>
```

————————————————

版权声明：本文为CSDN博主「markfengfeng」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。

原文链接：https://blog.csdn.net/markfengfeng/article/details/100171744