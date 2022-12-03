---
title: Vue+SpringBoot+Cas 前后端分离方式对接Cas
categories: 
- 笔记
tags: [SpringBoot,Vue,CAS,单点登录,前后端分离]
date: 2022-11-29 00:40:49
updated: 2022-11-29 00:40:50
cover: https://sm.ms/delete/NjEArfsZqUMKXQcb8nw7LhVSk5
---
#SpringBoot #Vue #CAS  #单点登录 #前后端分离 
# 前言
因为组里来了新的前端，所以新开的项目就都准备用前后端分离的方式来做，原本还挺爽的，只用写自己接口就完事了，但是到了对接甲方CAS的时候就出了问题。
# CAS认证流程
官网给出的认证流程如下，
![](https://s2.loli.net/2022/11/30/yH9ki1vE2KgaWlD.png)

TGC，TGT，ST之类的概念就不说了，网上一搜一大堆，简单来说就是访问网站后没登陆就跳到cas认证中心，登录完会在浏览器存一个cookie叫TGC（用来标识你登陆过了，以及后续获取ticket），跳转到你的服务上，并且url后面会携带一个ticket，也就是ST，client接收到这次请求后（一般是通过filter拦截地址，而地址默认是/cas/login，当然也可以自己设置），会把ticket发送给cas server去验证，验证通过后返回xml结构的用户信息。
# 后端流程
前后端分离对接CAS的流程一般是前端负责跳转登录页面，并且接收登录之后返回的ticket，将其传给后端，后端在接收到ticket之后进行验证，验证成功后返回给前端一个标识，用来判断接下来的请求是否已经认证。
后端认证的话有两种方式，一种是通过cas-client-core或者Spring Security进行配置，配置完之后只需要给前端/login接口和/logout接口即可，当然这个地址也可以用过setFilterProcessesUrl来自定义。
另一种方式自己写一个接口，直接请求CAS Server的验证票据接口，认证成功后返回给前端一个token，证明已经认证成功。
# 一、Spring Security代码实现
# 1. Spring Security配置
```java
@Configuration  
@EnableWebSecurity  
@EnableGlobalMethodSecurity(prePostEnabled = true, securedEnabled = true)  
public class CasSecurityConfig extends WebSecurityConfigurerAdapter {  
    @Value("${cas.server-host-url}")  
    private String serverHostUrl;  
    @Value("${cas.server-host-login-url}")  
    private String serverHostLoginUrl;  
    @Value("${cas.client-host-url}")  
    private String clientHostUrl;  
    @Autowired  
    private CustomUserDetailsService customUserDetailsService;  
    @Autowired  
    private CustomAuthenticationEntryPoint unAuthenticationEntrypoint;  
    @Autowired  
    private LogoutSuccessHandlerImpl logoutSuccessHandler;  
    @Autowired  
    private CasAuthenticationSuccessHandler casAuthenticationSuccessHandler;  
    @Autowired  
    private JwtAuthenticationTokenFilter authenticationTokenFilter;  
  
  
    @Override  
    protected void configure(HttpSecurity httpSecurity) throws Exception {  
        httpSecurity  
                .cors().disable()  
                .csrf().disable()  
                // 过滤请求  
                .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS).and()  
                .authorizeRequests()  
                .antMatchers(  
                        HttpMethod.GET,  
                        "/*.html",  
                        "/**/*.js",  
                        "/profile/**",  
                        "/error",  
                        "/api/**",  
                        "/**/*.css",  
                        "/**/*.png",  
                        "/**/*.jpg",  
                        "/**/*.jpeg",  
                        "/**/*.gif",  
                        "/**/*.ico",  
                        "/font/**"  
                ).permitAll()  
                .antMatchers("/swagger-ui.html").anonymous()  
                .antMatchers("/swagger-resources/**").anonymous()  
                .antMatchers("/*/api-docs").anonymous()  
                .antMatchers("/druid/**").anonymous()  
                // 除上面外的所有请求全部需要鉴权认证  
                .anyRequest().authenticated()  
                .and()  
                .headers().frameOptions().disable();  
        httpSecurity.exceptionHandling()  
                .authenticationEntryPoint(unAuthenticationEntrypoint).and()  
                .addFilterAt(casAuthenticationFilter(), CasAuthenticationFilter.class)  
                .addFilterBefore(authenticationTokenFilter, CasAuthenticationFilter.class)  
                .addFilterBefore(logoutFilter(), LogoutFilter.class)  
                .addFilterBefore(singleSignOutFilter(), CasAuthenticationFilter.class);  
    }  
  
    @Override  
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {  
        super.configure(auth);  
        auth.authenticationProvider(casAuthenticationProvider());  
    }  
  
    @Bean  
    public CasAuthenticationEntryPoint authenticationEntryPoint() {  
        CasAuthenticationEntryPoint casAuthenticationEntryPoint = new CasAuthenticationEntryPoint();  
        casAuthenticationEntryPoint.setLoginUrl(serverHostLoginUrl);  
        casAuthenticationEntryPoint.setServiceProperties(serviceProperties());  
        return casAuthenticationEntryPoint;  
    }  
  
    @Bean  
    @Override    public AuthenticationManager authenticationManagerBean() throws Exception {  
        return super.authenticationManagerBean();  
    }  
  
    @Bean  
    public ServiceProperties serviceProperties() {  
        ServiceProperties serviceProperties = new ServiceProperties();  
        serviceProperties.setService(clientHostUrl + "/login");  
        serviceProperties.setAuthenticateAllArtifacts(true);  
        return serviceProperties;  
    }  
  
    @Bean  
    public TicketValidator ticketValidator() {  
        return new Cas20ProxyTicketValidator(serverHostUrl);  
    }  
  
    @Bean  
    public CasAuthenticationProvider casAuthenticationProvider() {  
        CasAuthenticationProvider provider = new CasAuthenticationProvider();  
        provider.setServiceProperties(this.serviceProperties());  
        provider.setTicketValidator(this.ticketValidator());  
        provider.setAuthenticationUserDetailsService(customUserDetailsService);  
        provider.setKey("CAS_PROVIDER_KEY");  
        return provider;  
    }  
  
    @Bean  
    public CasAuthenticationFilter casAuthenticationFilter() throws Exception {  
        CasAuthenticationFilter casAuthenticationFilter = new CasAuthenticationFilter();  
        casAuthenticationFilter.setAuthenticationManager(authenticationManager());  
        casAuthenticationFilter.setFilterProcessesUrl("/login");  
        casAuthenticationFilter.setAuthenticationSuccessHandler(casAuthenticationSuccessHandler);  
        return casAuthenticationFilter;  
    }  
  
  
    @Bean  
    public SecurityContextLogoutHandler securityContextLogoutHandler() {  
        return new SecurityContextLogoutHandler();  
    }  
  
    @Bean  
    public LogoutFilter logoutFilter() {  
        LogoutFilter logoutFilter = new LogoutFilter(logoutSuccessHandler,  
                securityContextLogoutHandler());  
        logoutFilter.setFilterProcessesUrl("/logout");  
        return logoutFilter;  
    }  
  
    @Bean  
    public SingleSignOutFilter singleSignOutFilter() {  
        SingleSignOutFilter singleSignOutFilter = new SingleSignOutFilter();  
        singleSignOutFilter.setIgnoreInitConfiguration(true);  
        return singleSignOutFilter;  
    }  
  
}
```
# 2. 修改登录跳转
因为项目是前后端分离的方式，肯定是不会出现用户自行访问后端接口地址，然后跳转登录页面的情况，所以就需要修改authenticationEntryPoint，使其返回一个登录地址，让前端去跳转。
```java
@Component  
public class CustomAuthenticationEntryPoint implements AuthenticationEntryPoint {  
  
    // 配置文件中的CAS服务器地址  
    @Value("${cas.server-host-login-url}")  
    private String serverHostLoginUrl;  
  
    // 配置文件中的本应用前端地址  
    @Value("${cas.client-host-url}")  
    private String clientHostUrl;  
  
    /**  
     * 处理未登录或登录超时的逻辑，因为项目前后端分离，此处直接返回一串地址，跳转至CAS端进行登录，   
     */  
    @Override  
    public void commence(HttpServletRequest request, HttpServletResponse response, AuthenticationException authException) throws IOException, ServletException {  
        // 构造未登录情况需要跳转的login页面url  
        // 登录地址（指定的一个后台controller接口）  
        String encodeUrl = URLEncoder.encode(clientHostUrl + "/login", "utf-8");  
        // CAS认证中心页面地址，参数service带上登录地址，登录成功后会带上ticket跳转回service指定的地址  
        String redirectUrl = serverHostLoginUrl + "?service=" + encodeUrl;  
        response.setStatus(HttpServletResponse.SC_OK);  
        response.setHeader("content-type", "application/json;charset=UTF-8");  
        response.setCharacterEncoding("UTF-8");  
        PrintWriter out = response.getWriter();  
        // 返回与前端约定的格式，前端能获取到redirectUrl跳转即可  
        Result<String> unauthorized = Result.unauthorized(redirectUrl);  
        out.write(JSON.toJSONString(unauthorized));  
    }  
  
}
```
前端设置了全局拦截器后，这样不管请求什么接口，一旦状态码为401，就会自动跳转到登录地址。
# 3. 实现基于Jwt认证
```java
@Component  
public class JwtAuthenticationTokenFilter extends OncePerRequestFilter {  
    @Autowired  
    private TokenService tokenService;  
  
    @Override  
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain chain)  
            throws ServletException, IOException {  
        CasUserInfo casUserInfo = tokenService.getCasUser(request);  
        Authentication authentication = SecurityContextHolder.getContext().getAuthentication();  
        if (ObjectUtils.isNotEmpty(casUserInfo) && ObjectUtils.isEmpty(authentication)) {  
            tokenService.verifyToken(casUserInfo);  
            UsernamePasswordAuthenticationToken authenticationToken = new UsernamePasswordAuthenticationToken(casUserInfo, null, casUserInfo.getAuthorities());  
            authenticationToken.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));  
            SecurityContextHolder.getContext().setAuthentication(authenticationToken);  
        }  
        chain.doFilter(request, response);  
    }  
}
```
# 3. 登录成功处理
这一步是为了修改原本的成功跳转，改为生成JWT token，并存储在cookie中。
```java
@Service  
public class CasAuthenticationSuccessHandler extends SavedRequestAwareAuthenticationSuccessHandler {  
   private RequestCache requestCache = new HttpSessionRequestCache();  
  
   @Autowired  
   private TokenService tokenService;  
   @Value("${cas.web-url}")  
   private String webUrl;  
  
   /**  
    * 令牌有效期（默认30分钟）  
    */  
   @Value("${token.expireTime}")  
   private int expireTime;  
  
   @Override  
   public void onAuthenticationSuccess(HttpServletRequest request, HttpServletResponse response,  
                              Authentication authentication) throws ServletException, IOException {  
      String targetUrlParameter = getTargetUrlParameter();  
      if (isAlwaysUseDefaultTargetUrl()  
            || (targetUrlParameter != null && StringUtils.hasText(request.getParameter(targetUrlParameter)))) {  
         requestCache.removeRequest(request, response);  
         super.onAuthenticationSuccess(request, response, authentication);  
         return;      }  
      clearAuthenticationAttributes(request);  
      CasUserInfo userDetails = (CasUserInfo) authentication.getPrincipal();  
      String token = tokenService.createToken(userDetails);  
      //往Cookie中设置token  
      Cookie casCookie = new Cookie(Constants.TOKEN, token);  
      casCookie.setMaxAge(expireTime * 60);  
      casCookie.setPath("/");  
      response.addCookie(casCookie);  
      response.setContentType("application/json;charset=utf-8");  
      PrintWriter writer = response.getWriter();  
      writer.write(JSONObject.toJSONString(Result.success("登录成功",null)));  
      //登录成功后跳转到前端登录页面  
//    getRedirectStrategy().sendRedirect(request, response, webUrl);  
   }  
}
```
# 4. 单点登出
刚开始对接的时候，不知道是前端使用了异步请求导致后端无法正常重定向，还是本身就不好实现，登出跳转总是出问题，所以干脆改成返回json，再由前端跳转，和登录的步骤基本一致
```java
@Configuration  
public class LogoutSuccessHandlerImpl implements LogoutSuccessHandler {  
    @Autowired  
    private TokenService tokenService;  
    @Value("${cas.server-host-url}")  
    private String casServerHostUrl;  
    @Value("${cas.client-host-url}")  
    private String casClientHostUrl;  
  
    /**  
     * 退出处理  
     *  
     * @return  
     */  
    @Override  
    @SneakyThrows    
    public void onLogoutSuccess(HttpServletRequest request, HttpServletResponse response, Authentication authentication) {  
        CasUserInfo casUserInfo=tokenService.getCasUser(request);  
        if (ObjectUtils.isNotEmpty(casUserInfo)) {  
            // 删除用户缓存记录  
            tokenService.delLoginUser(casUserInfo.getToken());  
        }  
        String url = casServerHostUrl + "/logout?service=" + casClientHostUrl;  
        JSONObject jsonObject = new JSONObject();  
        jsonObject.put("logoutUrl", url);  
        ServletUtils.renderString(response, JSON.toJSONString(Result.success("退出成功",jsonObject)));  
//        response.sendRedirect(this.casServerHostUrl + "/logout?service=" + this.casClientHostUrl);  
    }  
}
```
# 5. 前端工作
前端需要做的是，根据后端返回的json跳转到登录页面，还有将登录成功后得到的ticket传给后端，以及从cookie里拿token并放在header里面。
# 二、serviceValidate配置
这种方式就是直接去请求CAS Server提供的一个票据验证的接口，后端这边没啥要做的，就是请求接口，解析返回的xml数据，并且给前端一个token即可。这边直接拿JeecgBoot里面的代码看一下。
```java
@Slf4j  
@RestController  
@RequestMapping("/sys/cas/client")  
public class CasClientController {  
  
   @Autowired  
   private ISysUserService sysUserService;  
   @Autowired  
    private ISysDepartService sysDepartService;  
   @Autowired  
    private RedisUtil redisUtil;  
  
   @Value("${cas.prefixUrl}")  
    private String prefixUrl;  
  
  
   @GetMapping("/validateLogin")  
   public Object validateLogin(@RequestParam(name="ticket") String ticket,  
                        @RequestParam(name="service") String service,  
                        HttpServletRequest request,  
                        HttpServletResponse response) throws Exception {  
      Result<JSONObject> result = new Result<>();  
      log.info("Rest api login.");  
      try {  
         String validateUrl = prefixUrl+"/p3/serviceValidate";  
         String res = CasServiceUtil.getStValidate(validateUrl, ticket, service);  
         log.info("res."+res);  
         final String error = XmlUtils.getTextForElement(res, "authenticationFailure");  
         if(StringUtils.isNotEmpty(error)) {  
            throw new Exception(error);  
         }  
         final String principal = XmlUtils.getTextForElement(res, "user");  
         if (StringUtils.isEmpty(principal)) {  
               throw new Exception("No principal was found in the response from the CAS server.");  
           }  
         log.info("-------token----username---"+principal);  
          //1. 校验用户是否有效  
         SysUser sysUser = sysUserService.getUserByName(principal);  
         result = sysUserService.checkUserIsEffective(sysUser);  
         if(!result.isSuccess()) {  
            return result;  
         }  
         String token = JwtUtil.sign(sysUser.getUsername(), sysUser.getPassword());  
         // 设置超时时间  
         redisUtil.set(CommonConstant.PREFIX_USER_TOKEN + token, token);  
         redisUtil.expire(CommonConstant.PREFIX_USER_TOKEN + token, JwtUtil.EXPIRE_TIME*2 / 1000);  
  
         //获取用户部门信息  
         JSONObject obj = new JSONObject();  
         List<SysDepart> departs = sysDepartService.queryUserDeparts(sysUser.getId());  
         obj.put("departs", departs);  
         if (departs == null || departs.size() == 0) {  
            obj.put("multi_depart", 0);  
         } else if (departs.size() == 1) {  
            sysUserService.updateUserDepart(principal, departs.get(0).getOrgCode());  
            obj.put("multi_depart", 1);  
         } else {  
            obj.put("multi_depart", 2);  
         }  
         obj.put("token", token);  
         obj.put("userInfo", sysUser);  
         result.setResult(obj);  
         result.success("登录成功");  
  
      } catch (Exception e) {  
         //e.printStackTrace();  
         result.error500(e.getMessage());  
      }  
      return new HttpEntity<>(result);  
   }  
  
  
}
```
需要注意的一点是，不同协议的票据验证地址是不一样的，比如CAS3.0的是/p3/serviceValiate，CAS2.0的则没有前面的p3，而1.0的地址则是/validate，并且接收的参数和返回的结构也有一些差别。