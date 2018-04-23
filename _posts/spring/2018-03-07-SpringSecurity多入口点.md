---
title: SpringSecurity多入口点
date: 2018-03-07 15:03:02
category: SpringSecurity
toc: true
classes: narrow
excerpt: "我们经常需要将普通用户和管理员用户做在同一个系统中，Spring Security 也并不也需要我们做太多的工作就可以支持。"
---

## 概述

我们经常需要将普通用户和管理员用户做在同一个系统中，Spring Security 也并不也需要我们做太多的工作就可以支持。

我们可以配置多个 HttpSecurity 实例，就像我们可以在xml文件中配置多个 \<http\> 一样。关键在于多次扩展 **WebSecurityConfigurationAdapter** 。

在这篇快速教程中，我们将介绍如何  在Spring Security应用程序中定义多个入口点。

这主要需要通过多次扩展 **WebSecurityConfigurerAdapter** 类来在XML配置文件或多个 **HttpSecurity** 实例中定义多个http块。

<!--more-->

## Maven的依赖

首先我们需要以下依赖项：


```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
    <version>1.5.2.RELEASE</version>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-thymeleaf</artifactId>
    <version>1.5.2.RELEASE</version>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <version>1.5.2.RELEASE</version>
</dependency>    
<dependency>
    <groupId>org.springframework.security</groupId>
    <artifactId>spring-security-test</artifactId>
    <version>4.2.2.RELEASE</version>
</dependency>
```

## 多入口点

### 具有多个HTTP元素的多个入口点

首先配置用户的数据来源，这里简单起见使用内存用户：

```java
@Configuration
@EnableWebSecurity
public class MultipleEntryPointsSecurityConfig {

    @Bean
    public UserDetailsService userDetailsService() throws Exception {
        InMemoryUserDetailsManager manager = new InMemoryUserDetailsManager();
        manager.createUser(User
          .withUsername("user")
          .password("userPass")
          .roles("USER").build());
        manager.createUser(User
          .withUsername("admin")
          .password("adminPass")
          .roles("ADMIN").build());
        return manager;
    }
}
```

现在，让我们看看我们如何在我们的安全配置中定义多个入口点。

我们将在此处使用基本身份验证驱动的示例，并且我们将充分利用Spring Security支持在我们的配置中定义多个HTTP元素的事实。

在使用Java配置时，定义多个安全域的方法是使用多个@Configuration类来扩展WebSecurityConfigurerAdapter类.

每个类都有自己的安全配置。这些类可以是静态的，并放置在主配置中。

让我们定义一个具有三个入口点的配置，每个入口点具有不同的权限和认证模式：

1. 一个用于使用HTTP基本认证的管理用户
2. 一个用于使用表单身份验证的常规用户
3. 一个用于不需要认证的访客用户

### 第一个入口

第一个入口 **/admin/\*\*** 保护为仅允许具有ADMIN角色的用户,并且需要具有使用 **authenticationEntryPoint()** 方法设置的 **BasicAuthenticationEntryPoint** 类型入口点的HTTP基本认证:

```java
@Configuration
@Order(1)
public static class App1ConfigurationAdapter extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.antMatcher("/admin/**")
            .authorizeRequests().anyRequest().hasRole("ADMIN")
            .and().httpBasic().authenticationEntryPoint(authenticationEntryPoint());
    }

    @Bean
    public AuthenticationEntryPoint authenticationEntryPoint(){
        BasicAuthenticationEntryPoint entryPoint =
          new BasicAuthenticationEntryPoint();
        entryPoint.setRealmName("admin realm");
        return entryPoint;
    }
}
```

每个静态类上的 **@Order** 注释指示配置将被视为找到与请求的URL匹配的顺序。每个 **@Order** 的值必须是唯一的。

BasicAuthenticationEntryPoint类型的bean需要设置属性 **RealmName**

### 第二个入口，使用相同的http元素

以 **/user/\*\*** 开头的链接需要具有USER角色的用户才能访问，USER用户使用的表单的认证方式

```java
@Configuration
@Order(2)
public static class App2ConfigurationAdapter extends WebSecurityConfigurerAdapter {
    protected void configure(HttpSecurity http) throws Exception {
        http.antMatcher("/user/**")
            .authorizeRequests().anyRequest().hasRole("USER")
            .and()
            // formLogin configuration
            .and()
            .exceptionHandling()
            .defaultAuthenticationEntryPointFor(
              loginUrlauthenticationEntryPointWithWarning(),
              new AntPathRequestMatcher("/user/private/**"))
            .defaultAuthenticationEntryPointFor(
              loginUrlauthenticationEntryPoint(),
              new AntPathRequestMatcher("/user/general/**"));
    }

    @Bean
	public AuthenticationEntryPoint loginUrlauthenticationEntryPoint(){
	    return new LoginUrlAuthenticationEntryPoint("/userLogin");
	}

	@Bean
	public AuthenticationEntryPoint loginUrlauthenticationEntryPointWithWarning(){
	    return new LoginUrlAuthenticationEntryPoint("/userLoginWithWarning");
	}
}
```

的确，除了authenticationEntryPoint()方法之外，另一种定义入口点的方法是使用defaultAuthenticationEntryPointFor()方法。这可以根据RequestMatcher对象定义多个匹配不同条件的入口点。

在这种情况下，入口点都是LoginUrlAuthenticationEntryPoint类型，并使用不同的登录页面URL：**/userLogin** 和 **/userLoginWithWarning** 同时都作为登录页面，并且尝试访问 **/user/private** 时会显示警告。

此配置还需要定义 /userLogin  和 /userLoginWithWarning MVC映射和两个带有标准登录表单的页面。

对于表单身份验证，请务必记住，配置的任何URL（例如登录处理URL）也需要遵循 **/user/\*\*** 格式，否则该url为可访问。

如果没有角色权限的用户尝试访问受保护的URL ，则上述两种配置都将重定向到 **/403** 。

即使它们处于不同的静态类中，也要小心地址url的名称，否则一个会覆盖另一个。


### 第三个入口点 不需要登录

最后定义 **/guest/\*\***，它允许所有类型包括未经认证的用户访问：

```java
@Configuration
@Order(3)
public static class App3ConfigurationAdapter extends WebSecurityConfigurerAdapter {

    protected void configure(HttpSecurity http) throws Exception {
        http.antMatcher("/guest/**").authorizeRequests().anyRequest().permitAll();
    }
}
```

### XML配置

```xml
<!-- 第一个入口点 -->
<security:http pattern="/admin/**" use-expressions="true" auto-config="true">
    <security:intercept-url pattern="/**" access="hasRole('ROLE_ADMIN')"/>
    <security:http-basic entry-point-ref="authenticationEntryPoint" />
</security:http>

<bean id="authenticationEntryPoint"
  class="org.springframework.security.web.authentication.www.BasicAuthenticationEntryPoint">
     <property name="realmName" value="admin realm" />
</bean>

<!-- 第二个入口点 -->
<security:http pattern="/user/general/**" use-expressions="true" auto-config="true"
  entry-point-ref="loginUrlAuthenticationEntryPoint">
    <security:intercept-url pattern="/**" access="hasRole('ROLE_USER')" />
    //form-login configuration      
</security:http>

<bean id="loginUrlAuthenticationEntryPoint"
  class="org.springframework.security.web.authentication.LoginUrlAuthenticationEntryPoint">
  <constructor-arg name="loginFormUrl" value="/userLogin" />
</bean>

<security:http pattern="/user/private/**" use-expressions="true" auto-config="true"
  entry-point-ref="loginUrlAuthenticationEntryPointWithWarning">
    <security:intercept-url pattern="/**" access="hasRole('ROLE_USER')"/>
    //form-login configuration
</security:http>

<bean id="loginUrlAuthenticationEntryPointWithWarning"
  class="org.springframework.security.web.authentication.LoginUrlAuthenticationEntryPoint">
    <constructor-arg name="loginFormUrl" value="/userLoginWithWarning" />
</bean>

<!-- 第三个入口点 -->
<security:http pattern="/**" use-expressions="true" auto-config="true">
    <security:intercept-url pattern="/guest/**" access="permitAll()"/>  
</security:http>
```

### Controller代码

```java
@Controller
public class PagesController {

    @RequestMapping("/admin/myAdminPage")
    public String getAdminPage() {
        return "multipleHttpElems/myAdminPage";
    }

    @RequestMapping("/user/general/myUserPage")
    public String getUserPage() {
        return "multipleHttpElems/myUserPage";
    }

    @RequestMapping("/user/private/myPrivateUserPage")
    public String getPrivateUserPage() {
        return "multipleHttpElems/myPrivateUserPage";
    }

    @RequestMapping("/guest/myGuestPage")
    public String getGuestPage() {
        return "multipleHttpElems/myGuestPage";
    }

    @RequestMapping("/multipleHttpLinks")
    public String getMultipleHttpLinksPage() {
        return "multipleHttpElems/multipleHttpLinks";
    }
}
```

html页面写在resources下的templates中，这里就省略了。

### 初始化应用程序

```java
@SpringBootApplication
public class MultipleEntryPointsApplication {
    public static void main(String[] args) {
        SpringApplication.run(MultipleEntryPointsApplication.class, args);
    }
}
```
