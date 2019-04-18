# spring-boot-shiro
将一个spring mvc +shiro项目转化成spring boot + shiro项目

以下记录此次转遇到的坑

  1 自定义静态资源路径问题

之前以为静态资源只要在配置文件中spring.mvc.static-path-pattern（访问路径）以及spring.resources.static-locations（资源实际路径）即可。但是这样配置之后，访问静态资源返回码为404.
之后仔细看了文档，才明白自定义静态资源需要写相关的配置类。
可是按照网上说的增加如下配置类

	@Configuration
	@EnableWebMvc
	public class WebConfig extends WebMvcConfigurerAdapter {
			@Override
			public void addResourceHandlers(ResourceHandlerRegistry registry) {
					registry.addResourceHandler("/t-static/**")
									.addResourceLocations("classpath:/templates/backstage/")
									.setCacheControl(CacheControl.maxAge(1, TimeUnit.HOURS).cachePublic());
			}
			}
可是本人使用的是spring-boot 2.4,WebMvcConfigurerAdapter已经被弃用了。之后查了下api文档，于是改成了如下代码：
		
		@Configuration
	public class ResourceConfig implements  WebMvcConfigurer{
			@Override
			public void addResourceHandlers(ResourceHandlerRegistry registry) {
					registry.addResourceHandler("/t-static/**")
									.addResourceLocations("classpath:/templates/backstage/")
									.setCacheControl(CacheControl.maxAge(1, TimeUnit.HOURS).cachePublic());
			}
	}
404问题就解决了

  2 jar 包冲突
 
 问题1：The bean 'propertySourcesPlaceholderConfigurer', defined in class path resource 
[org/springframework/boot/autoconfigure/session/JdbcSessionConfiguration$SpringBootJdbcHttpSessionConfiguration.class], 
could not be registered. 
A bean with that name has already been defined in class path resource 
[org/springframework/boot/autoconfigure/context/PropertyPlaceholderAutoConfiguratio.class] and overriding is disabled.

根据spring boot 提示在配置文件中加了
	spring.main.allow-bean-definition-overriding=true

但是又报以下的错了

org.springframework.boot.autoconfigure.session.NonUniqueSessionRepositoryException: Multiple session repository candidates are available, set the 'spring.session.store-type' property accordingly。
根据spring boot 提示在配置文件中加了
	spring.session.store-type=none可以解决以上问题。
	
不过根据网上的各种答案，有说是jar包问题。那么我仔细检查了下pom.xml 文件，发现在自己写的common tool jar中有引入<groupId>org.springframework.session</groupId>
<artifactId>spring-session</artifactId>

而项目中没有使用到spring-sessio，所以不包含这个依赖即可。

3 shiro 自定义filter 

解决以上问题之后，项目成功启动。配置了shiro的filter。若下：

    @Bean
    public UserFormAuthenticationFilter userFormAuthenticationFilter() {
        return new UserFormAuthenticationFilter();
    }
		
    @Bean("shiroFilter")
    public ShiroFilterFactoryBean shiroFilter() {
        ShiroFilterFactoryBean shiroFilterFactoryBean = new ShiroFilterFactoryBean();
        shiroFilterFactoryBean.setSecurityManager(securityManager());
        shiroFilterFactoryBean.setLoginUrl("/login");
        Map<String, Filter> filters = new LinkedHashMap<>();
        filters.put("anon", new AnonymousFilter());
        filters.put("authc", userFormAuthenticationFilter());
        shiroFilterFactoryBean.setFilters(filters);

        //注意此处使用的是LinkedHashMap，是有顺序的，shiro会按从上到下的顺序匹配验证，匹配了就不再继续验证
        //所以上面的url要苛刻，宽松的url要放在下面，尤其是"/**"要放到最下面，如果放前面的话其后的验证规则就没作用了。
      
        Map<String, String> filterChainDefinitionMap = new LinkedHashMap<>();
        filterChainDefinitionMap.put("/swagger-ui.html", "anon");
        filterChainDefinitionMap.put("/swagger-resources/**", "anon");
        filterChainDefinitionMap.put("/v2/api-docs", "anon");
        filterChainDefinitionMap.put(" /register.html", "anon");
        filterChainDefinitionMap.put("/sys/login**", "anon");
        filterChainDefinitionMap.put("/captcha.jpg**", "anon");
        filterChainDefinitionMap.put("/t-static/**", "anon");
        filterChainDefinitionMap.put("/database/success", "anon");
        filterChainDefinitionMap.put("/**", "authc");

        shiroFilterFactoryBean.setFilterChainDefinitionMap(filterChainDefinitionMap);
        return shiroFilterFactoryBean;
    }
打开login页面，login 页面的静态资源老是重定向（静态资源返回302）。静态资源是mapping到了/t-static/下。查看log，静态资源并没有通过filter，或者说shiroFilter没有起到作用。之后网上查找了一番，才知道自定义的UserFormAuthenticationFilter不应该使用spring进行管理，因为spring管理之后，在启动时便已经注册了。从而会影响shiroFilter的过滤。所以解决办法如下：

1 删除自定义的UserFormAuthenticationFilter的注入
2 将 filters.put("authc", userFormAuthenticationFilter()); 改为filters.put("authc", new UserFormAuthenticationFilter());

				 @Bean("shiroFilter")
    public ShiroFilterFactoryBean shiroFilter() {
        ShiroFilterFactoryBean shiroFilterFactoryBean = new ShiroFilterFactoryBean();
        shiroFilterFactoryBean.setSecurityManager(securityManager());
        shiroFilterFactoryBean.setLoginUrl("/login");
        Map<String, Filter> filters = new LinkedHashMap<>();
        filters.put("anon", new AnonymousFilter());
        filters.put("authc", new UserFormAuthenticationFilter());
        shiroFilterFactoryBean.setFilters(filters);

        //注意此处使用的是LinkedHashMap，是有顺序的，shiro会按从上到下的顺序匹配验证，匹配了就不再继续验证
        //所以上面的url要苛刻，宽松的url要放在下面，尤其是"/**"要放到最下面，如果放前面的话其后的验证规则就没作用了。
      
        Map<String, String> filterChainDefinitionMap = new LinkedHashMap<>();
        filterChainDefinitionMap.put("/swagger-ui.html", "anon");
        filterChainDefinitionMap.put("/swagger-resources/**", "anon");
        filterChainDefinitionMap.put("/v2/api-docs", "anon");
        filterChainDefinitionMap.put(" /register.html", "anon");
        filterChainDefinitionMap.put("/sys/login**", "anon");
        filterChainDefinitionMap.put("/captcha.jpg**", "anon");
        filterChainDefinitionMap.put("/t-static/**", "anon");
        filterChainDefinitionMap.put("/database/success", "anon");
        filterChainDefinitionMap.put("/**", "authc");

        shiroFilterFactoryBean.setFilterChainDefinitionMap(filterChainDefinitionMap);
        return shiroFilterFactoryBean;
    }


 
 
