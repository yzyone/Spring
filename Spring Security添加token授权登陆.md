
# Spring Security添加token授权登陆 #

需求：其他项目要跳转一个springboot+springsecurity项目，需要授权特定token登陆。
实现思路：添加过滤器，拦截请求中的token，传递给Authentication鉴权，如果这个请求不带有授权信息，被拦截后，会跳转登录页面。

参考代码：
配置：

```
@Configuration
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {
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
    *核心代码逻辑
    **/
     @Override
    protected void configure(HttpSecurity http) throws Exception {
        
        //添加 token 过滤器。
        http.addFilterBefore(tokenAuthenticationFilter(), UsernamePasswordAuthenticationFilter.class);
    }
}
```

token授权:

过滤器定义：

注：参考AbstractAuthenticationProcessingFilter和UsernamePasswordAuthenticationToken去实现一下就好了。

```
public class TokenAuthenticationFilter extends AbstractAuthenticationProcessingFilter {


    private String tokenParameter = "token";


    
    /**
     * @param defaultFilterProcessesUrl
     */
    public TokenAuthenticationFilter() {
        super("/quicklogin");//放行的请求url
        super.setAuthenticationFailureHandler(new LoginFailureHandler());//失败页面
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

authProvider的定义:

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

TokenAuthenticationToken：

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

参考链接：https://blog.csdn.net/markfengfeng/article/details/100171744
————————————————

版权声明：本文为CSDN博主「student100000」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。

原文链接：https://blog.csdn.net/student100000/article/details/107755350