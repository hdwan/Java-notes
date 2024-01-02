# Spring Security

Spring Security是一个功能强大且高度可定制的**身份验证和访问控制**框架。Spring Security致力于为Java应用程序提供身份验证和授权的能力。像所有Spring项目一样，Spring Security的真正强大之处在于它可以轻松扩展以满足定制需求的能力。

![image-20231223160231071](typora文档图片/image-20231223160231071.png)

Spring Security两大重要核心功能：**用户认证（Authentication）**和**用户授权（Authorization）**。

- **用户认证Authentication**：验证某个用户是否为系统中的合法主体，也就是说**用户能否访问该系统**。用户认证一般要求用户提供用户名和密码。系统通过校验用户名和密码来完成认证过程。
- **用户授权Authorization**：验证某个用户**是否有权限执行某个操作**。在一个系统中，不同用户所有的权限是不同的。比如对一个文件来说，有的用户只能进行读取，有的用户既能读取，又能修改。一般来说，系统会为不同的用户分配不同的角色，而每个角色则对应一系列的权限。

## 概述

Spring Security**本质**上就是一个过滤器链，通过过滤器拦截请求并进行处理。

### 一般流程

1. 用户登录时，前端将用户输入的**用户名、密码信息传输到后台**，后台**用一个类对象将其封装起来**，通常使用的是`UsernamePasswordAuthenticationToken`这个类。
2. 程序负责**验证**这个类对象。验证方法是调用Service根据username从数据库中取用户信息到实体类的实例中，比较两者的密码，如果密码正确就成功登陆，同时把包含着用户的用户名、密码、所具有的权限等信息的类对象放到`SecurityContextHolder`（安全上下文容器，类似Session）中去。
3. 用户访问一个资源的时候，**首先判断是否是受限资源。如果是的话还要判断当前是否未登录，没有的话就跳到登录页面。**
4. 如果用户已经登录，访问一个受限资源的时候，程序要根据url去数据库中取出该资源所对应的所有可以访问的角色，然后拿着当前用户的所有角色一一对比，判断用户是否可以访问（这里就是和**权限**相关）。

## 入门项目

### 默认认证

创建一个 spring boot 项目，主要安装两个依赖：

![image-20231223211333994](typora文档图片/image-20231223211333994.png)

```xml
 <dependency>
     <groupId>org.springframework.boot</groupId>
     <artifactId>spring-boot-starter-security</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

**解决跨域**

```java
package com.springsecurity.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.CorsRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

// 解决跨域
@Configuration
public class CorsConfig implements WebMvcConfigurer {
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/**")
                .allowedOriginPatterns("*")
                .allowedMethods("GET", "HEAD", "POST", "PUT", "DELETE", "OPTIONS")
                .allowCredentials(true)
                .maxAge(3600)
                .allowedHeaders("*");
    }
}

```

**控制层设置接口**

```java
@RestController
@RequestMapping("/test")
public class Test {

    @GetMapping("/test")
    public String test() {
        System.out.println("收到请求");
        return "success";
    }
}
```

**访问后端**

在浏览器中输入：`http://localhost:8888/test/test`

会自动进行认证，调转到登录页面：

![image-20231223220712578](typora文档图片/image-20231223220712578.png)

用户名默认为：`user`，密码后端自动生成：

![image-20231223220740532](typora文档图片/image-20231223220740532.png)

这是引入了SpringSecurity依赖之后默认的登陆页面，即实现了“认证”。

### 默认认证实现原理

仅仅只需要引入一个依赖，就开启了认证，如何实现的？

首先，Spring Security 中 `认证、授权` 等功能都是基于 `过滤器` 完成的。

1. 观察控制台日志输出：

    ![image-20231228171757163](typora文档图片/image-20231228171757163.png)

    项目启动时候的**临时密码**是**org.springframework.boot.autoconfigure.security.servlet**包下的`UserDetailsServiceAutoConfiguration`类生成的，部分源码如下：

    ```java
        private String getOrDeducePassword(SecurityProperties.User user, PasswordEncoder encoder) {
            String password = user.getPassword();
            if (user.isPasswordGenerated()) {
                logger.warn(String.format("%n%nUsing generated security password: %s%n%nThis generated password is for development use only. Your security configuration must be updated before running your application in production.%n", user.getPassword()));
            }
    
            return encoder == null && !PASSWORD_ALGORITHM_PATTERN.matcher(password).matches() ? "{noop}" + password : password;
        }
    ```

2. 从源码可以看出，密码来自于 `SecurityProperties` 的内部类 `User`：`SecurityProperties.User user`，查看源码：

    ![image-20231228194946471](typora文档图片/image-20231228194946471.png)

    `User` 是 `SecurityProperties` 的静态内部类，默认的用户名为“user”，且密码的生成方式是UUID。

    **此外**：该类上添加了注解 `@ConfigurationProperties`：

    - 该注解的作用：获取配置文件中的属性值，类似于 `@Value` 注解。
    - 前缀 `prefix` ：定义了哪些外部属性将绑定到类的字段上。

    所以我们可以在配置文件中自己设置用户名和密码：

    ```yaml
    spring:
      security:
        user:
          password: admin
          name: admin
    ```

    分析`User`静态类的源码可以发现，属性 `passwordGenerated` 的值默认为true，即密码不指定的话默认是自动生成的，如果我们在配置文件中重新设置了用户密码，则创建`User`的时候进行setter注入就会将配置文件的用户名密码注入，并将`passwordGenerated`设置为`false`，此时`UserDetailsServiceAutoConfiguration`类的就不会打印密码了，如下图。

    ![image-20231228200352185](typora文档图片/image-20231228200352185.png)

![image-20231228201113115](typora文档图片/image-20231228201113115.png)

继续看 `UserDetailsServiceAutoConfiguration`，该类以 `AutoConfiguration` 结尾，即自动配置类，该类上添加的注解：

```java
注释：
Auto-configuration for a Spring Security in-memory AuthenticationManager. Adds an InMemoryUserDetailsManager with a default user and generated password. This can be disabled by providing a bean of type AuthenticationManager, AuthenticationProvider or UserDetailsService.

自动配置内存中的Spring Security AuthenticationManager。使用默认用户和生成的密码添加InMemoryUserDetailsManager。可以通过提供AuthenticationManager、AuthenticationProvider或UserDetailsService类型的bean来禁用。

@Configuration(
    proxyBeanMethods = false
)
@ConditionalOnClass({AuthenticationManager.class})
@ConditionalOnBean({ObjectPostProcessor.class})
@ConditionalOnMissingBean(
    value = {AuthenticationManager.class, AuthenticationProvider.class, UserDetailsService.class, AuthenticationManagerResolver.class},
    type = {"org.springframework.security.oauth2.jwt.JwtDecoder", "org.springframework.security.oauth2.server.resource.introspection.OpaqueTokenIntrospector", "org.springframework.security.oauth2.client.registration.ClientRegistrationRepository"}
)
public class UserDetailsServiceAutoConfiguration {
```

**关于自动配置类**：Spring Boot的自动配置功能是**通过条件注解实现**的。这种方式可以根据一定的条件来判断是否需要自动配置某些组件，在程序启动时**自动装配这些组件到Spring容器中**。这样,我们在使用Spring Boot时，只需要添加相应的依赖，就可以直接使用这些组件了，无需手动配置。

所以 该类是自动配置类，可以使用默认的用户和生成的密码。

注解 `@ConditionalOnXxxxx` 和 `@ConditionalOnMissingXxxxx` 表示项目中包含`Xxxxx` 和不包含 `Xxxxx` 的时候才满足条件。

注意：`@ConditionalOnMissingXxxxx`的参数value表示数组里所有Xxxxx都不在项目中才满足条件。

所以该类在类路径下存在 `AuthenticationManager` 、在Spring 容器中存在`Bean` `ObjectPostProcessor` 并且不存在`Bean` `AuthenticationManager` , `AuthenticationProvider` , `UserDetailsService` 的情况下生效。

因此可以手动实现value数组中任何一个使得条件不生效，从而不使用 `UserDetailsServiceAutoConfiguration` 自动配置类。（注释中也说明了）

所以一般自己实现 `UserDetailsService` 接口。

接着分析 `UserDetailsServiceAutoConfiguration` 的源码：

```java
public class UserDetailsServiceAutoConfiguration {

	private static final String NOOP_PASSWORD_PREFIX = "{noop}";

	private static final Pattern PASSWORD_ALGORITHM_PATTERN = Pattern.compile("^\\{.+}.*$");

	private static final Log logger = LogFactory.getLog(UserDetailsServiceAutoConfiguration.class);

	@Bean
	@Lazy
	public InMemoryUserDetailsManager inMemoryUserDetailsManager(SecurityProperties properties,
			ObjectProvider<PasswordEncoder> passwordEncoder) {
		SecurityProperties.User user = properties.getUser();
		List<String> roles = user.getRoles();
		return new InMemoryUserDetailsManager(
				User.withUsername(user.getName()).password(getOrDeducePassword(user, passwordEncoder.getIfAvailable()))
						.roles(StringUtils.toStringArray(roles)).build());
	}

	private String getOrDeducePassword(SecurityProperties.User user, PasswordEncoder encoder) {
		String password = user.getPassword();
		if (user.isPasswordGenerated()) {
			logger.warn(String.format(
					"%n%nUsing generated security password: %s%n%nThis generated password is for development use only. "
							+ "Your security configuration must be updated before running your application in "
							+ "production.%n",
					user.getPassword()));
		}
		if (encoder != null || PASSWORD_ALGORITHM_PATTERN.matcher(password).matches()) {
			return password;
		}
		return NOOP_PASSWORD_PREFIX + password;
	}

}
```

该类通过 `@Lazy` （该注解表示只有在需要使用时才会被实例化,以减少启动时间和资源占用）初始化了类名为 `InMemoryUserDetailsManager` 的内存用户管理器：

```java
public class InMemoryUserDetailsManager implements UserDetailsManager, UserDetailsPasswordService {
    .....
}
public interface UserDetailsManager extends UserDetailsService {
    .....
}
```

该管理器实现了 `UserDetailsManager` 接口，通过配置注入了一个默认的 UserDetails 存在内存中，就是我们上面用的那个 user ，每次启动 user 都是动态生成的。而 `UserDetailsManager` 接口又继承了 `UserDetailsService` 接口（注意：接口可以继承接口，实体类不能继承接口，只能实现），所以实际上还是用 `UserDetailsService` 加载用户信息。

下面看 `UserDetailsService` 接口:

```java
public interface UserDetailsService {
    UserDetails loadUserByUsername(String username) throws UsernameNotFoundException;
}
```

一个很简单的接口，只有一个方法：`loadUserByUsername`，即通过用户名来加载用户。这个方法主要用于从系统数据中查询并加载具体的用户到 Spring Security中。返回类型是 `UserDetails`，交给spring。

`UserDetails`：该接口是**提供用户信息的核心接口**。该接口**的实现仅仅存储用户的信息**。后续会将该接口提供的用户信息封装到认证对象 Authentication 中去。 UserDetails 默认提供了：

```java
public interface UserDetails extends Serializable {
	// 用户的权限集， 默认需要添加 ROLE_ 前缀
	Collection<? extends GrantedAuthority> getAuthorities();
	// 用户的加密后的密码， 不加密会使用 {noop} 前缀
	String getPassword();
    // 应用内唯一的用户名
	String getUsername();
    // 账户是否过期
	boolean isAccountNonExpired();
    // 账户是否锁定
	boolean isAccountNonLocked();
    // 凭证是否过期
	boolean isCredentialsNonExpired();
	// 用户是否可用
	boolean isEnabled();

}
```

可以实现该接口以存储更多的用户信息。比如用户的邮箱、手机 号等等。通常我们使用其实现类：

```java
org.springframework.security.core.userdetails.User
```

该类内置一个建造器 UserBuilder 会很方便地帮助我们构建 UserDetails 对象。

**自定义用户管理**

上面提到过默认使用的是`UserDetailsManager` 的实现类 `InMemoryUserDetailsManager` 进行用户管理，我们可以定义自己的 UserDetailsManager Bean，实现我们需要的用户管理逻辑：

```JAVA
@Configuration
public class UserDetailsServiceConfiguration {

    @Bean
    public UserDetailsRepository userDetailsRepository() {
        UserDetailsRepository userDetailsRepository = new UserDetailsRepository();

        // 为了让我们的登录能够运行 这里我们初始化一个用户Felordcn 密码采用明文 当你在密码12345上使用了前缀{noop} 意味着你的密码不使用加密，authorities 一定不能为空 这代表用户的角色权限集合
        UserDetails felordcn = User.withUsername("Felordcn").password("{noop}12345").authorities(AuthorityUtils.NO_AUTHORITIES).build();
        userDetailsRepository.createUser(felordcn);
        return userDetailsRepository;
    }


    @Bean
    public UserDetailsManager userDetailsManager(UserDetailsRepository userDetailsRepository) {
        return new UserDetailsManager() {
            @Override
            public void createUser(UserDetails user) {
                userDetailsRepository.createUser(user);
            }

            @Override
            public void updateUser(UserDetails user) {
                userDetailsRepository.updateUser(user);
            }

            @Override
            public void deleteUser(String username) {
                userDetailsRepository.deleteUser(username);
            }

            @Override
            public void changePassword(String oldPassword, String newPassword) {
                userDetailsRepository.changePassword(oldPassword, newPassword);
            }

            @Override
            public boolean userExists(String username) {
                return userDetailsRepository.userExists(username);
            }

            @Override
            public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
                return userDetailsRepository.loadUserByUsername(username);
            }
        };
    }
}


/**
 * 自己实现 UserDetailsManager
 * @author Daniel
 * @date 2023/12/28
 */
public class UserDetailsRepository {

    /**
     * 替换为 抽象DAO接口可进行用户持久化操作
     */
    private Map<String, UserDetails> users = new HashMap<>();

    /**
     * Create user.
     * @param user the user
     */
    public void createUser(UserDetails user) {
        users.putIfAbsent(user.getUsername(), user);
    }

    /**
     * Update user.
     * @param user the user
     */
    public void updateUser(UserDetails user) {
        users.put(user.getUsername(), user);
    }

    /**
     * Delete user.
     * @param username the username
     */
    public void deleteUser(String username) {
        users.remove(username);
    }

    /**
     * Change password.
     * @param oldPassword the old password
     * @param newPassword the new password
     */
    public void changePassword(String oldPassword, String newPassword) {
        Authentication currentUser = SecurityContextHolder.getContext()
                .getAuthentication();

        if (currentUser == null) {
            // This would indicate bad coding somewhere
            throw new AccessDeniedException(
                    "Can't change password as no Authentication object found in context "
                            + "for current user.");
        }

        String username = currentUser.getName();

        UserDetails user = users.get(username);


        if (user == null) {
            throw new IllegalStateException("Current user doesn't exist in database.");
        }

        // 实现具体的更新密码逻辑
    }

    /**
     * User exists boolean.
     * @param username the username
     * @return the boolean
     */
    public boolean userExists(String username) {
        return users.containsKey(username);
    }

    /**
     * Load user by username user details.
     * @param username the username
     * @return the user details
     * @throws UsernameNotFoundException the username not found exception
     */
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        return users.get(username);
    }

}
```

只需要将 UserDetailsRepository 中的 users 属性替代为抽象的Dao接口，即可使用数据库来管理用户。



### 密码加密

```java
	public InMemoryUserDetailsManager inMemoryUserDetailsManager(SecurityProperties properties,	ObjectProvider<PasswordEncoder> passwordEncoder) {
```

上面的`UserDetailsServiceAutoConfiguration` 的源码里在初始化 `InMemoryUserDetailsManager` 的时候，传入了一个参数：`ObjectProvider<PasswordEncoder>`，这里的 **`PasswordEncoder`** 就是对密码进行编码的工具接口。该接口只有两个功能： 匹配验证、密码编码。

```java
public interface PasswordEncoder {
    String encode(CharSequence rawPassword); // 编码
    boolean matches(CharSequence rawPassword, String encodedPassword); // 匹配 明文和密文
    // 该方法用来判断当前密码是否需要升级，可以看见这个方法是默认的
	// 默认返回值是 false
    default boolean upgradeEncoding(String encodedPassword) {
        return false;
    }
}
```

`PasswordEncoder` 的常见实现类：

- `BCryptPasswordEncoder`：加密密码；
- `DelegatingPasswordEncoder`：委托密码编码器；

#### 委托密码编码器 DelegatingPasswordEncoder

Delegate，委托。即自己不干，交给别人干。

之所以使用 委托 设计模式，而不直接使用具体的加密类，原因：

- 有很多应用程序使用旧的密码编码不容易进行迁移；
- 密码存储的最佳实践就被更改了；
- 而 Spring Security 作为一个框架而言，不能这么轻易地带破坏性的更改。

使用 DelegatingPasswordEncoder 的好处：

- 确保使用的密码编码可以进行规范的正确的密码存储；
- 允许以现代和遗留格式验证密码；
- 允许将来升级编码；

该类的成员属性：

```java
public class DelegatingPasswordEncoder implements PasswordEncoder {
    // 默认包裹 id 的前、后缀
	private static final String PREFIX = "{";
	private static final String SUFFIX = "}";
	private final String idForEncode; // 通过id来匹配具体编码器，即用来匹配 PasswordEncoder的实现类
	private final PasswordEncoder passwordEncoderForEncode; // 实际采用的加密的方案对象，即PasswordEncoder的某种实现类 如BCryptPasswordEncoder
	private final Map<String, PasswordEncoder> idToPasswordEncoder; // 用来维护多个 idForEncode 与具体 PasswordEncoder 实现类的映射关系。 DelegatingPasswordEncoder 初始化时
装载进去，会在初始化时进行一些规则校验。
	private PasswordEncoder defaultPasswordEncoderForMatches = new UnmappedIdPasswordEncoder(); // 默认的密码匹配器，上面的 Map 中都不存在就用它来执行 	matches 方法进行匹配验证。这是一个内部类实现。
	}
```

`DelegatingPasswordEncoder` 中的编码方法：

```java
	@Override
	public String encode(CharSequence rawPassword) {
		return PREFIX + this.idForEncode + SUFFIX + this.passwordEncoderForEncode.encode(rawPassword);
	}
```

可以看出编码规则：`{idForEncode}encodePassword`，即 前缀 + 编码方式 + 后缀 + 原始密码编码后的密码。

`DelegatingPasswordEncoder` 中的**密码匹配**方法：

```java
	@Override
	// rawPassword 是原密码，即用户输入的
    // prefixEncodePassword 可以理解为是保存在数据库中的密码（数据库保存的是密文）
	public boolean matches(CharSequence rawPassword, String prefixEncodedPassword) {
		if (rawPassword == null && prefixEncodedPassword == null) {
			return true;
		}
		String id = extractId(prefixEncodedPassword); // 从 {id} 中获取 id
		PasswordEncoder delegate = this.idToPasswordEncoder.get(id); // 找出id对应的编码类（PasswordEncoder实现类）
		if (delegate == null) { // 找不到就用默认的比对方式
			return this.defaultPasswordEncoderForMatches.matches(rawPassword, prefixEncodedPassword);
		}
        // 找到了就进行比对
		String encodedPassword = extractEncodedPassword(prefixEncodedPassword);
		return delegate.matches(rawPassword, encodedPassword);
	}
```

传入**原始密码和遵循 `{idForEncode}encodePassword` 规则的密码编码串**。通过获取编码方式id ( idForEncode ) 来从 DelegatingPasswordEncoder 中的映射集合 idToPasswordEncoder 中获取具体的 PasswordEncoder 实现类进行匹配校验。找不到就使用默认的`defaultPasswordEncoderForMatches`方法生成的属性 `UnmappedIdPasswordEncoder`。

#### 密码器静态工厂PasswordEncoderFactories

DelegatingPasswordEncoder 在哪实例化的？

`PasswordEncoderFactories`，密码编码器工厂类，专门制造 `PasswordEncoder`。

```java
public final class PasswordEncoderFactories {

	private PasswordEncoderFactories() {
	}

	@SuppressWarnings("deprecation")
	public static PasswordEncoder createDelegatingPasswordEncoder() {
		String encodingId = "bcrypt";
		Map<String, PasswordEncoder> encoders = new HashMap<>();
		encoders.put(encodingId, new BCryptPasswordEncoder());
		encoders.put("ldap", new org.springframework.security.crypto.password.LdapShaPasswordEncoder());
		encoders.put("MD4", new org.springframework.security.crypto.password.Md4PasswordEncoder());
		encoders.put("MD5", new org.springframework.security.crypto.password.MessageDigestPasswordEncoder("MD5"));
		encoders.put("noop", org.springframework.security.crypto.password.NoOpPasswordEncoder.getInstance());
		encoders.put("pbkdf2", new Pbkdf2PasswordEncoder());
		encoders.put("scrypt", new SCryptPasswordEncoder());
		encoders.put("SHA-1", new org.springframework.security.crypto.password.MessageDigestPasswordEncoder("SHA-1"));
		encoders.put("SHA-256",
				new org.springframework.security.crypto.password.MessageDigestPasswordEncoder("SHA-256"));
		encoders.put("sha256", new org.springframework.security.crypto.password.StandardPasswordEncoder());
		encoders.put("argon2", new Argon2PasswordEncoder());
		return new DelegatingPasswordEncoder(encodingId, encoders);
	}

}

```

可以看出，该类是一个静态类，只提供了初始化 `PasswordEncoder` 的方法 `createDelegatingPasswordEncoder`，即创建`DelegatingPasswordEncoder`。

从方法中可以看出 id 是 `bcrypt`，即默认采用的是`bcrypt`进行编码。

另外，可以看出 `"noop"` 对应的是 `NoOpPasswordEncoder`，即不加密密码，直接使用明文和明文对比，所以前面默认认证以及我们实现管理器的时候密码都会加上 `{noop}`，以此来使用 `NoOpPasswordEncoder` 取代默认的密码加密方式。这也是为啥前端输入的密码xxx能和我们设置的{noop}xxx匹配上。

这样做的好处：增加了”**多样性**“，即使以后我们对密码编码规则进行替换，也不会影响以前的用户使用的加密编码规则。

**Spring Security 加载 PasswordEncoder 的规则**

`PasswordEncoderFactories` 又是从哪来的呢？

`WebSecurityConfigurerAdapter`里面可以找到：

```java
static class LazyPasswordEncoder implements PasswordEncoder {
        private ApplicationContext applicationContext;
        private PasswordEncoder passwordEncoder;
        LazyPasswordEncoder(ApplicationContext applicationContext) {
            this.applicationContext = applicationContext;
        }
        public String encode(CharSequence rawPassword) {
            return this.getPasswordEncoder().encode(rawPassword);
        }
        public boolean matches(CharSequence rawPassword, String encodedPassword) {
            return this.getPasswordEncoder().matches(rawPassword, encodedPassword);
        }
        public boolean upgradeEncoding(String encodedPassword) {
            return this.getPasswordEncoder().upgradeEncoding(encodedPassword);
        }
    -------------------------这里----------------
        private PasswordEncoder getPasswordEncoder() {
            if (this.passwordEncoder != null) {
                return this.passwordEncoder;
            } else {
                PasswordEncoder passwordEncoder = (PasswordEncoder)this.getBeanOrNull(PasswordEncoder.class);
                if (passwordEncoder == null) {
                    passwordEncoder = PasswordEncoderFactories.createDelegatingPasswordEncoder();
                }

                this.passwordEncoder = passwordEncoder;
                return passwordEncoder;
            }
        }
        private <T> T getBeanOrNull(Class<T> type) {
            try {
                return this.applicationContext.getBean(type);
            } catch (NoSuchBeanDefinitionException var3) {
                return null;
            }
        }
        public String toString() {
            return this.getPasswordEncoder().toString();
        }
    }
```

在一个实现了 `PasswordEncoder` 接口的静态内部类 `LazyPasswordEncoder` 里面。

该类的 `getPasswordEncoder` 方法中用到了 `PasswordEncoderFactories`。

该方法返回 `PasswordEncoder`，如果现有`PasswordEncoder`非空，则直接返回；否则先从spring的ioc容器中找，找到了就返回；如果找不到就用 `PasswordEncoderFactories` 创建的，即用`PasswordEncoderFactories`的方法`createDelegatingPasswordEncoder`创建 `DelegatingPasswordEncoder`。

上面的**自定义用户管理** 就是先创建了一个 `UserDetailsRepository` 代理默认的 `InMemoryUserDetailsManager` 的功能，将其作为参数（通过bean注解注入容器）传递给`UserDetailsManager`（），再注入spring容器中，这样spring就能找到`PasswordEncoder`，不适用默认的。缺点：只能用一种 PasswordEncoder 进行密码比对，好处就是专一、密码前不用写{id}。

如果不写 {id} 呢？会怎么密码比对呢？

从源码中可以看出：执行DelegatingPasswordEncoder 中的默认方案（找不到匹配的时候），即会抛出java.lang.IllegalArgumentException: There is no PasswordEncoder mapped for the id "null"的异常。

### 关于bcrypt

前面提到过默认的编码方式是`bcrypt`加密。

`bcrypt` 使用的是布鲁斯·施内尔在1993年发布的 Blowfish 加密算法。 `bcrypt` 算法将 salt 随机并混入最终加密后的密码，验证时也无需单独提供之前的 salt ，从而无需单独处理 salt 问题。加密后的格式一般为：  

```tex
$2a$10$/bTVvqqlH9UiE0ZJZ7N2Me3RIgUCdgMheyTgV0B4cMCSokPa.6oCa
```

`$` 是分割符，无意义; `2a` 是 bcrypt 加密版本号； `10` 是 cost 的值；而后的前 22 位是 `salt` 值；再然后的字符串就是密码的密文。

**bcrypt 特点**

- 慢，难以破解。
- 同样的密码每次使用 bcrypt 编码，密码暗文都是不一样的。























































配置类中注入PaswordEncoder的bean对象：

```java
@Bean
public PasswordEncoder password() {
    return new BCryptPasswordEncoder();
}
```

通常而言，在配置类中注入PaswordEncoder的bean对象是必须的，因为Spring Security 要求容器中必须有 PasswordEncoder 实例,才能加密。所以当我们手动加入自定义登录逻辑时，要求必须给容器注入PaswordEncoder的bean对象。

如果不想使用它自带的加密方式，也可以使用自己的。**写一个类实现PasswordEncoder接口。**

注意：同一个字符串，通过加密生成的字符串每次都不一样，但是尽管每次都不一样，也都不会匹配失败。即同一个密码生成的密文每次都不一样，但是无论是哪个密文，最终都能解析成功。

### 自定义账户密码

#### 配置文件中定义

在application配置文件中设置账户密码：

```yaml
spring:
  security:
    user:
      name: apple
      password: 123456
```

这是认证的时候账号密码就变成了配置文件里的。

#### 通过数据库

1. 编写配置类

    ```java
    @Configuration
    public class SecurityConfig extends WebSecurityConfigurerAdapter {
        @Autowired
        private UserDetailsService userDetailsService;
    
        @Override
        protected void configure(AuthenticationManagerBuilder auth) throws Exception {
            auth.userDetailsService(userDetailsService)
                    .passwordEncoder(password());
        }
    
        @Bean
        public PasswordEncoder password() {
            return new BCryptPasswordEncoder();
        }
    }
    
    ```

2. 层

    ```java
    // controller
    @RestController
    @RequestMapping("test")
    public class TestController {
    
        @Autowired
        private UserMapper userMapper;
    
    
        @GetMapping("test")
        public void test() {
            System.out.println("测试");
            System.out.println(userMapper.selectList(null).size());
        }
    
    }
    
    // mapper
    @Mapper
    public interface UserMapper extends BaseMapper<User> {
    }
    
    // pojo
    @Data
    @NoArgsConstructor
    @AllArgsConstructor
    @TableName("user")
    public class User {
        private Integer id;
        private String username;
        private String password;
    }
    
    ```

3. service层

    注意：这个UserDetailsService接口是springsecurity内部提供的，只需要编写对应的实现类即可完成用户认证授权。

    ```java
    package com.sqlsecurity.service;
    
    import com.baomidou.mybatisplus.core.conditions.query.QueryWrapper;
    import com.sqlsecurity.dao.UserMapper;
    import com.sqlsecurity.pojo.User;
    import org.springframework.beans.factory.annotation.Autowired;
    import org.springframework.security.core.GrantedAuthority;
    import org.springframework.security.core.authority.AuthorityUtils;
    import org.springframework.security.core.authority.SimpleGrantedAuthority;
    import org.springframework.security.core.userdetails.UserDetails;
    import org.springframework.security.core.userdetails.UserDetailsService;
    import org.springframework.security.core.userdetails.UsernameNotFoundException;
    import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
    import org.springframework.stereotype.Service;
    
    import java.util.ArrayList;
    import java.util.Collection;
    import java.util.List;
    
    @Service
    public class MyUserDetailsService implements UserDetailsService {
    
        @Autowired
        private UserMapper userMapper;
    
    
        @Override
        public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
            // 授权
            List<GrantedAuthority> authorities = AuthorityUtils.commaSeparatedStringToAuthorityList("manager");
            /**
             * 根据用户名从数据库查询符合要求的用户
             * user 是spring security默认的用户类  包含username password；或者自定义用户类
             * 该项目用的是mybatis-plus
             */
            QueryWrapper<com.sqlsecurity.pojo.User> queryWrapper =  new QueryWrapper<>();
            queryWrapper.eq("username", username);
            User user = userMapper.selectOne(queryWrapper);
            if (user == null) { // 排除用户不存在异常
                throw new UsernameNotFoundException("用户不存在");
            }
    
            return new org.springframework.security.core.userdetails.User(
                    user.getUsername(),
                    user.getPassword(),
                    authorities
            );
        }
    }
    
    ```

4. 配置数据库连接

    ```yaml
    server:
      port: 8888
    spring:
      datasource:
        driver-class-name: com.mysql.cj.jdbc.Driver
        url: jdbc:mysql://localhost:3306/database?serverTimezone=Asia/Shanghai&useUnicode=true&characterEncoding=utf-8
        username: root
        password: xxxxxx
    ```

用数据库中的用户名和密码即可进行登录认证。

1. 系统默认提供一个登录页面和登录接口。

2. 密码校验是由SpringSecurity内部完成。不需要我们来处理。我们只需要**将数据库查出来的用户名和密码交给spring security提供的User类即可**。即service层里的`loadUserByUsername`方法返回值。



#### 自定义登录页面

默认情况下使用的是springsecurity自带的登录页面，我们可以自己实现。

主要是通过配置类中，重写configure（HttpSecurity http）的这个方法：

```java
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.formLogin()//表示进行表单登录
                .loginPage("/login.html")//自定义的登录页面
                .loginProcessingUrl("/login")//传一个登录处理的接口，不管你传的接口地址是什么，都由Security内部完成。当然也可以自己写这个接口，这样就不会用系统来完成登录处理校验用户名密码了。还有就是如果自定义了登录页面，那么登录处理的接口loginProcessingUrl项一定要写，不管是写系统自带的，还是你自己写的处理接口都行，否则报错。
                .usernameParameter("username") //定义登录时的用户名的key，即表单中name的值，默认为username
                .passwordParameter("password") //定义登录时的密码key,即表单中name的值，默认是password
                //设置的这两个用户名、密码的key，如果不自己写登录页面的话，可以不用写，因为系统默认提供的页面就是这个默认值。写了的话，一定要与表单页面中定义的name值一致才行。
                .defaultSuccessUrl("/pages/main")//登录成功跳转到的页面或者路径。当然，如果你不是从登录页面登录的，那么拦截之后会进入到你的请求路径（或页面）中
                .failureUrl("/login.html")//登录失败跳转到的页面
                .permitAll() //指和登录表单相关的接口 都通过，不拦截
                .and()
                .authorizeRequests()//开启授权请求
                .antMatchers("/","/pages/main","/login").permitAll()//设置哪些路径放行，不需要认证	不需要登录可以访问的
                .anyRequest().authenticated()//除开上面的，其他所有请求全部都需要认证。因为还没有用户授权，所以目前所有的接口登录后都能访问。
                .and()
                .csrf().disable();//关闭csrf防护
    }
```

### 授权

不授权的情况下，当用户认证完成后可以访问所有路径。

#### 为路径授权

**示例**：

```java
.antMatchers("/security/test").hasAuthority("admin")
//表示当前登录用户，只有具有权限名称为admin时，才能访问此地址
```

在配置文件重写的`configure`方法中添加授权：

```java
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.formLogin()//表示进行表单登录
                .loginPage("/login.html")//自定义的登录页面
                .loginProcessingUrl("/login")//传一个登录处理的接口，不管你传的接口地址是什么，都由Security内部完成。当然也可以自己写这个接口，这样就不会用系统来完成登录处理校验用户名密码了。还有就是如果自定义了登录页面，那么登录处理的接口loginProcessingUrl项一定要写，不管是写系统自带的，还是你自己写的处理接口都行，否则报错。
                .usernameParameter("username") //定义登录时的用户名的key，即表单中name的值，默认为username
                .passwordParameter("password") //定义登录时的密码key,即表单中name的值，默认是password
                //设置的这两个用户名、密码的key，如果不自己写登录页面的话，可以不用写，因为系统默认提供的页面就是这个默认值。写了的话，一定要与表单页面中定义的name值一致才行。
                .defaultSuccessUrl("/pages/main")//登录成功跳转到的页面或者路径。当然，如果你不是从登录页面登录的，那么拦截之后会进入到你的请求路径（或页面）中
                .failureUrl("/login.html")//登录失败跳转到的页面
                .permitAll() //指和登录表单相关的接口 都通过，不拦截
                .and()
                .authorizeRequests()//开启授权请求
                .antMatchers("/","/pages/main","/login").permitAll()//设置哪些路径放行，不需要认证	
                .antMatchers("/security/test").hasAuthority("admin")//表示当前登录用户，只有具有权限名称为admin时，才能访问此地址
                .anyRequest().authenticated()//除开上面的，其他所有请求全部都需要认证。
                .and()
                .csrf().disable();//关闭csrf防护
    }
```

#### 为用户添加权限

##### 用方法授权

在`UserDetailsService`实现类中，重写方法中增加权限

```java
 @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        // 授权
        List<GrantedAuthority> authorities = AuthorityUtils.commaSeparatedStringToAuthorityList("manager, admin");
```

也可以通过查询数据库获取权限，然后添加进去。

常见授权方法：

- `hasAuthority(String authority)`

    意思是如果当主体具有指定的权限，则返回true，否则返回false。

    注意：该方法只能判断是否有某种权限。

    ```java
    // 配置类中设置访问权限
    .antMatchers("/security/test").hasAuthority("admin")
    
    // UserDetailsService实现类中设置
    List<GrantedAuthority> authorities = AuthorityUtils.commaSeparatedStringToAuthorityList("manager, admin");
    ```

- `hasAnyAuthority(String... authorities)`

    当前主体**具有任意一个权限**，则返回true，否则返回false。

    ```java
    .antMatchers("/security/test").hasAnyAuthority("admin, role1, role1")
    ```

    参数是逗号分隔得字符串。

- `hasRole(String role)`

    当前主体具有指定的角色，则返回true，否则返回false。

    ```java
    .antMatchers("/security/test").hasRole("sale")
    ```

    注意：如果是hasRole，那么**在userDetailsService实现类中的角色名前面一定要添加ROLE_**

    ```java
    List<GrantedAuthority> authorities = AuthorityUtils.commaSeparatedStringToAuthorityList("manager, ROLE_admin");
    ```

- `hasAnyRole(String... roles)`

    当前主**体具备任何一个角色**，则返回true，否则返回false。

    ```java
    .antMatchers("/security/test").hasAnyRole("sale, ROLE_role1")
    ```

##### 用注解授权

也可以**在对应的控制器方法上**，添加对应的注解来进行授权访问。

首先需要在启动类上添加注解：

```java
@EnableGlobalMethodSecurity(securedEnabled=true)
```

在方法上授权的注解：`@Secured`

判断**是否具有某个角色**。注意：匹配的角色字符串需要添加前缀“ROLE_“。

```java
@Secured({"ROLE_role1", "ROLE_role2"})
@GetMapping("test")
public void test() {
    
}
```

注意：该注解表示只要拥有任一角色即可访问，如果需要同时满足多个角色，该注解就无法满足需求。

使用注解：**`@PreAuthorize`**

```java
@PreAuthorize("hasAnyRole('normal','admin')")
```

可以判断是否**具有某个角色**或**权限**，也判断是否**同时具有某些角色**或**权限**

```java
@PreAuthorize("hasRole('ROLE_ADMIN') and hasRole('ROLE_USER')")
```

```java
@PostAuthorize： 在目标方法执行之后进行权限校验。
@PostFilter： 在目标方法执行之后对方法的返回结果进行过滤。
@PreAuthorize： 在目标方法执行之前进行权限校验。
@PreFilter： 在目标方法执行之前对方法参数进行过滤。
@secured： 访问目标方法必须具备相应的角色。
@DenyAll： 拒绝所有访问。
@PermitAll： 允许所有访问。
@RolesAllowed： 访问目标方法必须具备相应的角色。
```



### 记住我

即登录之后在一定期限内免登录。

正常而言，关闭浏览器后，默认cookie会消失。但是如果设置了记住我，那么它会生成一个**叫remember me的cookie**，这个cookie包含了用户信息，且不会消失，直到设置的过期时间到了才会消失。

因此我们关闭后，再次请求，它会带着这个rememberme（就是token）到数据库中查询。

**修改配置文件**

或者新建配置文件

```java
    /**
      * 自动登录 配置类中，注入数据源和配置操作数据库对象
      */
     //注入数据源
     @Autowired
     private DataSource dataSource;
 
     //注入操作数据库的对象JdbcTokenRepositoryImpl,用它来创建token
     // 当然最好选择返回上层接口。我们一般是这样的，因为多态方便后续修改维护。
     @Bean
     public PersistentTokenRepository persistentTokenRepository(){
         JdbcTokenRepositoryImpl jdbcTokenRepository = new JdbcTokenRepositoryImpl();
         //  赋值数据源
         jdbcTokenRepository.setDataSource(dataSource);
         //  自动创建表 , 第一次执行会创建，以后要执行就要删除掉！
         //jdbcTokenRepository.setCreateTableOnStartup(true);//这里我们是自己创建的数据库。所以不要这句
         return jdbcTokenRepository;
     }
 
 
 
 	@Override
     protected void configure(HttpSecurity http) throws Exception {
         ......
          .and()
          .rememberMe()//开启记住我功能
          .tokenRepository(persistentTokenRepository())//把操作数据库对象传进来
          .tokenValiditySeconds(60)//表示自动登录，60s内有效
          .userDetailsService(userDetailsService)//查询数据库的service
         ......
     }
```

`tokenValiditySeconds`设置有效时间：以 `秒` 为单位，如一天表示：60\*60\*24

### 关闭安全认证

更改启动类的注解：

```java
@SpringBootApplication(exclude = {SecurityAutoConfiguration.class})
```

### CSRF功能

前面配置文件中：

![image-20231226105714077](typora文档图片/image-20231226105714077.png)

跨站请求伪造（Cross-site request forgery）。

跨站请求位置默认开启。针对 PATCH，POST，PUT 和 DELETE 方法进行防护。

想要实现该功能，只需要在配置类中开启CSRF的情况下，在前端中设置如下：

```html
<input type="hidden"th:if="${_csrf}!=null"th:value="${_csrf.token}"name="_csrf"/>
```

一般测试的时候，免得在前端还要加上这个代码。都选择关闭CSRF功能。如果不关闭，那么在前端表单登录的代码中一定要加上上面这段。否则自己写的登录页面（属于跨站），POST提交就会被进行防护。

### 防止同时在线

配置类中增加session相关配置：

```java
        //踢下线配置
        http.sessionManagement()
                .maximumSessions(1) // 表示同一个用户最大登录客户端的数量为1
                .maxSessionsPreventsLogin(false) // 阻止登录策略，如果为true，表示已经登录就不允许在别的地方登录了。如果为false，则表示在其他地方登录后，就会踢出之前其他地方登录的该账号。
                .expiredSessionStrategy(new SessionInformationExpiredStrategy() {
                    // 方法一：页面跳转的方式处理
                    //private RedirectStrategy redirectStrategy = new DefaultRedirectStrategy();
                    // 当发现session超时，或者session被踢下线之后，要进行的处理
                    //@Override
                    //public void onExpiredSessionDetected(SessionInformationExpiredEvent event) throws IOException, ServletException {
                    //    redirectStrategy.sendRedirect(event.getRequest(),event.getResponse(),"/forced");
                    //}
 
                    // 方法二：前后端分离的情况下，一般是返回json数据
                    // 可以使用springboot默认的jackson的json处理对象，当然你也可以使用其他json工具
                    private ObjectMapper objectMapper = new ObjectMapper();
                    @Override
                    public void onExpiredSessionDetected(SessionInformationExpiredEvent event) throws IOException, ServletException {
 
                        Map<String, Object> map = new HashMap<>();
                        map.put("code",50009);
                        map.put("data",null);
                        map.put("msg","您已在其他地方进行了登录，请核实是否为本人操作!");
                        String json = objectMapper.writeValueAsString(map);
                        event.getResponse().setContentType("application/json;charset=utf-8");
                        event.getResponse().getWriter().write(json);
                    }
                });
```

不同地方同时登录会被顶掉。

### 认证常规配置

```
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.formLogin()
                .and()
                .authorizeRequests()
                .antMatchers("/test/register", "/test/login").permitAll()
                .anyRequest().authenticated()
                .and().csrf().disable();
    }
```

- 开启授权；
- 登录、注册放行；
- 开启认证；
- 关闭csrf防护；



## 待定

###  过滤器链

从上到下

- `WebAsyncManagerIntegrationFilter`：将SecurityContext集成到Spring MVC中用于管理异步请求处理的WebAsyncManager中。
- `SecurityContextPersistenceFilter`：在当前会话中填充SecurityContext，SecurityContext即Security的上下文对象，里面包含了当前用户的认证及权限信息等。
- `HeaderWriterFilter`：向请求的Header中添加信息
- `CsrfFilter`：用于防止CSRF（跨域请求伪造）攻击。Spring Security会对所有post请求验证是否包含系统生成的CSRF的信息，如果不包含则报错。
- `LogoutFilter`：匹配URL为“/logout”的请求，清除认证信息，实现用户注销功能。
- `UsernamePasswordAuthenticationFilter`：认证操作的过滤器，用于匹配URL为“/login”的POST请求做拦截，校验表单中的用户名和密码。
- `DefaultLoginPageGeneratingFilter`：如果没有配置登陆页面，则生成默认的认证页面
- `DefaultLogoutPageGeneratingFilter`：用于生成默认的退出页面
- `BasicAuthenticationFilter`：用于Http基本认证，自动解析Http请求头中名为Authentication的内容，并获得内容中“basic”开头之后的信息。
- `RequestCacheAwareFilter`：用于缓存HttpServletRequest
- `SecurityContextHolderAwareRequestFilter`：用于封装ServletRequest，让ServletRequest具备更多功能。
- `AnonymousAuthenticationFilter`：对于未登录情况下的处理，当SecurityContextHolder中认证信息为空，则会创建一个匿名用户存入到SecurityContextHolder中
- `SessionManagementFilter`：限制同一用户开启多个会话
- `ExceptionTranslationFilter`：异常过滤器，用来处理在认证授权过程中抛出异常。
- `FilterSecurityInterceptor`：获取授权信息，根据SecurityContextHolder中存储的用户信息判断用户是否有权限访问

**核心过滤器**

`FilterSecurityInterceptor`：是一个**方法级**的权限过滤器, 基本位于过滤链的最底部。

`ExceptionTranslationFilter`：是个异常过滤器，用来处理在认证授权过程中抛出的异常。主要用于处理AuthenticationException（认证）和AccessDeniedException（授权）的异常

`UsernamePasswordAuthenticationFilter` ：对/login 的 POST 请求做拦截，校验表单中用户名，密码。最常用的用户名和密码认证方式的主要处理类,构造了一个UsernamePasswordAuthenticationToken对象实现类。





### 过滤器加载过程

Springboot在整合Spring Security项目时会**自动配置DelegatingFilterProxy**过滤器，若非Springboot工程，则需要**手动配置**该过滤器。

​    

![springsecurity过滤器链](typora文档图片/springsecurity过滤器链.png)

**过滤器如何进行加载的**：

Security在**DelegatingFilterProxy**的doFilter()调用了initDelegat()方法，在该方法中调用了`WebApplicationContext`的getBean()方法，该方法触发FilterChainProxy的doFilterInternal方法，用于获取过滤链中的所有过滤器并进行加载。



### 权限管理相关概念

#### 认证

`authentication`，权限管理系统确认一个主体的身份，允许主体进入系统。简单说就是“主体”证明自己是谁。笼统的认为就是以前所做的**登录操作**。

#### 授权

`authorization` 将操作系统的“权力”授予“主体”，这样主体就具备了操作系统中特定功能的能力。简单来说，授权就是给用户**分配权限**。

#### 主体

`principal` 使用系统的用户或设备或从其他系统远程登录的用户等等。简单说就是谁使用系统谁就是主体。

**`认证`**和**`授权`** 是分开的，无论使用什么样的认证方式。都不会影响授权，这是两个独立的存在，这种独立带来的好处之一，就是可以非常方便地整合一些外部的解决方案。

![image-20231224214814025](typora文档图片/image-20231224214814025.png)

## 认证方式

所谓的**认证**，就是用来判断系统中是否存在某用户，并判断该用户的身份是否合法的过程，解决的其实是用户登录的问题。认证的存在，是为了保护系统中的隐私数据与资源，只有合法的用户才可以访问系统中的资源。

**认证方式**

在Spring Security中，常见的认证方式可以分为HTTP层面和表单层面，常见的认证方式如下:

- HTTP基本认证；

- Form表单认证；

- HTTP摘要认证；


### 认证流程

**认证流程**：

`UsernamePasswordAuthenticationFilter`

- 是最常用的用户名和密码认证方式的主要处理类,构造了一个UsernamePasswordAuthenticationToken对象实现类，将用请求信息封装为Authentication。

`Authentication接口`

-  封装了用户相关信息。

`AuthenticationManager接口`

- 定义了认证Authentication的方法，是认证相关的核心接口，也是发起认证的出发点，因为在实际需求中，我们可能会允许用户使用用户名+密码登录，同时允许用户使用邮箱+密码，手机号码+密码登录，甚至，可能允许用户使用指纹登录，所以说AuthenticationManager一般不直接认证，AuthenticationManager接口的常用实现类ProviderManager 内部会维护一个List列表，存放多种认证方式，实际上这是委托者模式的应用（Delegate）。也就是说，核心的认证入口始终只有一个：AuthenticationManager

AuthenticationManager，ProviderManager ，AuthenticationProvider…

用户名+密码（UsernamePasswordAuthenticationToken），邮箱+密码，手机号码+密码登录则对应了三个AuthenticationProvider

`DaoAuthenticationProvider`

- 用于解析并认证 UsernamePasswordAuthenticationToken 的这样一个认证服务提供者,对应以上的几种登录方式。

`UserDetailsService接口`

- Spring Security 会将前端填写的username 传给 UserDetailService.loadByUserName方法。我们只需要从数据库中根据用户名查找到用户信息然后封装为UserDetails的实现类返回给SpringSecurity 即可，自己不需要进行密码的比对工作，密码比对交由SpringSecurity处理。

`UserDetails接口`

- 提供核心用户信息。通过UserDetailsService根据用户名获取处理的用户信息要封装成UserDetails对象返回。然后将这些信息封装到Authentication对象中。



1. **在Spring Security中认证是由`AuthenticationManager`接口来负责的，接口定义为：**

```java
public interface AuthenticationManager {
    Authentication authenticate(Authentication authentication) throws AuthenticationException;
```

- 返回 `Authentication` 表示认证成功；
- 返回 `AuthenticationException` 异常，表示认证失败；

AuthenticationManager 主要实现类为 ProviderManager，在 ProviderManager 中管理了众多 AuthenticationProvider 实例。在一次完整的认证流程中，Spring Security 允许存在多个 AuthenticationProvider ，用来实现多种认证方式，这些 AuthenticationProvider 都是由 ProviderManager 进行统一管理的。

![image-20231224215130969](typora文档图片/image-20231224215130969.png)

2. **Authentication**

认证以及认证成功的信息主要是由 Authentication 的实现类进行保存的，其接口定义为：

```java
public interface Authentication extends Principal, Serializable {
    Collection<? extends GrantedAuthority> getAuthorities();

    Object getCredentials();

    Object getDetails();

    Object getPrincipal();

    boolean isAuthenticated();

    void setAuthenticated(boolean isAuthenticated) throws IllegalArgumentException;
}
```

- getAuthorities 获取用户权限信息；
- getCredentials 获取用户凭证信息，一般指密码；
- getDetails 获取用户详细信息；
- getPrincipal 获取用户身份信息，用户名、用户对象等；
- isAuthenticated 用户是否认证成功；

3. **SecurityContextHolder**

SecurityContextHolder 用来获取登录之后用户信息。Spring Security 会将登录用户数据保存在 Session 中。但是，为了使用方便,Spring Security在此基础上还做了一些改进，其中最主要的一个变化就是线程绑定。当用户登录成功后,Spring Security 会将登录成功的用户信息保存到 SecurityContextHolder 中。SecurityContextHolder 中的数据保存默认是通过ThreadLocal 来实现的，使用 ThreadLocal 创建的变量只能被当前线程访问，不能被其他线程访问和修改，也就是用户数据和请求线程绑定在一起。当登录请求处理完毕后，Spring Security 会将 SecurityContextHolder 中的数据拿出来保存到 Session 中，同时将 SecurityContexHolder 中的数据清空。以后每当有请求到来时，Spring Security 就会先从 Session 中取出用户登录数据，保存到 SecurityContextHolder 中，方便在该请求的后续处理过程中使用，同时在请求结束时将 SecurityContextHolder 中的数据拿出来保存到 Session 中，然后将 Security SecurityContextHolder 中的数据清空。这一策略非常方便用户在 Controller、Service 层以及任何代码中获取当前登录用户数据。

### HTTP基本认证

#### 简述

HTTP基本认证是在RFC2616标准中定义的一种认证模式，它以一种很简单的方式与用户进行交互。HTTP基本认证可以分为如下4个步骤：

- 客户端首先**发起一个未携带认证信息**的请求；

- 然后**服务器端返回一个401 Unauthorized的响应信息**，并在**WWW-Authentication头部中说明认证形式**：当进行HTTP基本认证时，WWW-Authentication会被**设置为Basic realm=“被保护的页面”**；

- 接下来**客户端会收到这个401 Unauthorized响应信息**，并弹出一个对话框，询问用户名和密码。当用户输入后，客户端会将**用户名和密码使用冒号进行拼接并用Base64编码**，然后将其放入到请求的Authorization头部并发送给服务器；

- 最后服务器端对客户端发来的信息进行解码得到用户名和密码，并对该信息进行校验判断是否正确，最终给客户端返回响应内容。

HTTP基本认证是一种无状态的认证方式，与表单认证相比，HTTP基本认证是一种基于HTTP层面的认证方式，无法携带Session信息，也就**无法实现Remember-Me**功能。另外，用户名和密码在传递时仅做了一次简单的Base64编码，几乎等同于以明文传输，极易被进行密码窃听和重放攻击。所以在实际开发中，很少会使用这种认证方式来进行安全校验。

#### 代码实现

**创建HttpSecurityConfig配置类**

```java
/**
 * 类功能描述：HTTP基本认证
 */
@EnableWebSecurity
public class HttpSecurityConfig extends WebSecurityConfigurerAdapter {
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        // super.configure(http);
        http.authorizeRequests() // 认证配置
                .anyRequest() // 任何请求
                .authenticated() // 都需要身份验证
                .and()
                .httpBasic(); // http基本认证
    }
}
```

**控制层设置接口**

```java
@RestController
@RequestMapping("/test")
public class Test {

    @GetMapping("/test")
    public String test() {
        System.out.println("收到请求");
        return "success";
    }
}
```

**访问后端**

在浏览器中输入：`http://localhost:8888/test/test`

出现登录页面：

![image-20231223220712578](typora文档图片/image-20231223220712578.png)

用户名默认为：`user`，密码后端自动生成：

![image-20231223220740532](typora文档图片/image-20231223220740532.png)



**Basic认证**

在未登录状态下访问目标资源时，查看响应头，可以看到`WWW-Authenticate`认证信息:WWW-Authenticate：Basic realm="Realm"，其中：

WWW-Authenticate: 表示**服务器告知浏览器进行代理认证工作**。

Basic: 表示认证类型为Basic认证。

realm="Realm": 表示认证域名为Realm域。

![image-20231224131516333](typora文档图片/image-20231224131516333.png)

**认证过程**：

根据401和以上响应头信息，浏览器会弹出一个对话框，要求输入用户名/密码，Basic认证会将其拼接成 “`用户名:密码`” 格式，中间是一个冒号，并利用Base64编码成加密字符串xxx；然后在请求头中附加 Authorization: Basic xxx 信息，发送给后台认证；后台需要利用Base64来进行解码xxx，得到用户名和密码，再校验 `用户名:密码` 信息。如果认证错误，浏览器会保持弹框；如果认证成功，浏览器会缓存有效的Base64编码，在之后的请求中，浏览器都会在请求头中添加该有效编码。

### Form表单认证

在SpringBoot开发环境中，**只要我们添加了Spring Security的依赖包，就会自动实现表单认证。**可以通过WebSecurityConfigurerAdapter提供的configure方法看到默认的认证方式就是表单认证，源码如下：

```java
    protected void configure(HttpSecurity http) throws Exception {
        this.logger.debug("Using default configure(HttpSecurity). If subclassed this will potentially override subclass configure(HttpSecurity).");
        http.authorizeRequests((requests) -> {
            ((ExpressionUrlAuthorizationConfigurer.AuthorizedUrl)requests.anyRequest()).authenticated();
        });
        http.formLogin();
        http.httpBasic();
    }
```

#### 表单认证中的预置url和页面

> 默认的formLogin配置中，自动配置了一些url和页面:
>
> - **/login(get)**: get请求时会跳转到这个页面，只要我们**访问任意一个需要认证的请求时，都会跳转**到这个登录界面。
> - **/login(post)**: post请求时会触发这个接口，在登录页面点击登录时，默认的登录页面表单中的action就是关联这个login接口。
> - **/login?error**: 当用户名或密码错误时，会跳转到该页面。
> - **/:** 登录成功后，默认跳转到该页面，如果配置了index.html页面，则 ”/“ 会重定向到index.html页面，当然这个页面要由我们自己实现。
> - **/logout:** 注销页面。
> - **/login?logout:** 注销成功后跳转到的页面。
>
> 由此可见，SpringSecurity默认有两个login，即登录页面和登录接口的地址都是 /login:
>
> - GET http://localhost:8080/login
> - POST http://localhost:8080/login
>
> 如果是 GET 请求，表示你想访问登录页面；如果是 POST 请求，表示你想提交登录数据。
> 简单了解。

#### 自定义认证

```java
package com.qf.my.ss.demo.config;

import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.builders.WebSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter;

package com.springsecurity.config;

import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.builders.WebSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter;

/**
 * 类功能描述：form表单认证
 */
@EnableWebSecurity
public class FormSecurityConfig extends WebSecurityConfigurerAdapter {
    
    // 静态资源访问不走过滤器链 直接放行
    @Override
    public void configure(WebSecurity web) throws Exception {
        web.ignoring().antMatchers("/js/**", "/cs/**", "/images/**");
    }

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests()
                .anyRequest()
                .authenticated()
                .and()
                .formLogin()
                .loginPage("login.html")
                .permitAll()
                .defaultSuccessUrl("/main.html", true)
                .loginProcessingUrl("/login") // 指登录成功后，是否始终跳转到登录成功url。它默认为false
                .failureUrl("/error.html") // 用户密码错误跳转接口
                .usernameParameter("username") // 要认证的用户参数名，默认username
                .passwordParameter("password") // 要认证的密码参数名，默认password
                .and()
                .logout() // 配置注销
                .logoutUrl("/logout") //注销接口
                .logoutSuccessUrl("/login.html") // 注销成功后跳转到的接口
                .permitAll()
                .deleteCookies("myCookie") // 删除自定义的cookie
                .and()
                .csrf() // 注意:需禁用crsf防护功能,否则登录不成功
                .disable();
    }
}

```

#### WebSecurity和HttpSecurity

Spring Security内部是如何加载我们自定义的登录页面的？

WebSecurity和HttpSecurity。

- `WebSecurity`：

    在这个类里定义了一个securityFilterChainBuilders集合，可以同时管理多个SecurityFilterChain过滤器链，当WebSecurity在执行时，会构建出一个名为 **”springSecurityFilterChain“** 的 **Spring BeanFilterChainProxy代理类**，它的作用是来 **定义哪些请求可以忽略安全控制，哪些请求必须接受安全控制**；以及在合适的时候 **清除SecurityContext** 以避免内存泄漏，同时也可以用来 **定义请求防火墙和请求拒绝处理器**，也可以在这里 **开启Spring Security 的Debug模式**。

- `HttpSecurity`：

    HttpSecurity用来构建包含一系列的过滤器链SecurityFilterChain，平常我们的配置就是围绕着这个SecurityFilterChain进行。

### Http摘要认证

#### **概念**

HTTP摘要认证和HTTP基本认证一样，也是在RFC2616中定义的一种认证方式，它的出现是为了弥补HTTP基本认证存在的安全隐患，但该认证方式也并不是很安全。**HTTP摘要认证会使用对通信双方来说都可知的口令进行校验，且最终以密文的形式来传输数据，所以相对于基本认证来说，稍微安全了一些**。

**HTTP摘要认证与基本认证类似，基于简单的“挑战-回应”模型。**当我们发起一个未经认证的请求时，服务器会返回一个401回应，并给客户端返回与验证相关的参数，期待客户端依据这些参数继续做出回应，从而完成整个验证过程。

#### **摘要认证核心参数**

服务端给客户端返回的验证相关参数如下：

```
username: 用户名。

password: 用户密码。

realm: 认证域，由服务器返回。

opaque: 透传字符串，客户端应原样返回。

method: 请求的方法。

nonce: 由服务器生成的随机字符串，包含过期时间(默认过期时间300s)和密钥。

nc: 即nonce-count,指请求的次数，用于计数，防止重放攻击。qop被指定时，nc也必须被指定。

cnonce: 客户端发给服务器的随机字符串，qop被指定时，cnonce也必须被指定。

qop: 保护级别，客户端根据此参数指定摘要算法。若取值为 auth,则只进行身份验证；若取值为auth-int，则还需要校验内容完整性，默认的qop为auth。

uri: 请求的uri。

response: 客户端根据算法算出的摘要值，这个算法取决于qop。

algorithm: 摘要算法，目前仅支持MD5。

entity-body: 页面实体，非消息实体，仅在auth-int中支持。
```

> 通常服务器端返回的数据包括realm、opaque、nonce、qop等字段，如果客户端需要做出验证回应，就必须按照一定的算法得到一些新的数据并一起返回。**在以上各种参数中，对服务器而言，最重要的字段是nonce；对客户端而言，最重要的字段是response。**

#### 摘要认证实现

```java
package com.qf.my.spring.security.demo.config;

import com.qf.my.spring.security.demo.service.MyUserDetailService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter;
import org.springframework.security.web.authentication.www.DigestAuthenticationEntryPoint;
import org.springframework.security.web.authentication.www.DigestAuthenticationFilter;

/**
 * 摘要认证的配置
 * @author Thor
 * @公众号 Java架构栈
 */
@EnableWebSecurity
public class DigestConfig extends WebSecurityConfigurerAdapter {

    @Autowired
    private DigestAuthenticationEntryPoint digestAuthenticationEntryPoint;

    @Autowired
    private MyUserDetailService userDetailService;

    //配置认证入口端点，主要是设置认证参数信息
    @Bean
    public DigestAuthenticationEntryPoint digestAuthenticationEntryPoint(){
        DigestAuthenticationEntryPoint point = new DigestAuthenticationEntryPoint();
        point.setKey("security demo");
        point.setRealmName("thor");
        point.setNonceValiditySeconds(500);
        return point;
    }

    public DigestAuthenticationFilter digestAuthenticationFilter(){
        DigestAuthenticationFilter filter = new DigestAuthenticationFilter();
        filter.setAuthenticationEntryPoint(digestAuthenticationEntryPoint);
        filter.setUserDetailsService(userDetailService);
        return filter;
    }

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests()
                .antMatchers("/hello").hasAuthority("role")
                .anyRequest().authenticated()
                .and().csrf().disable()
                //当未认证时访问某些资源,则由该认证入口类来处理.
                .exceptionHandling()
                .authenticationEntryPoint(digestAuthenticationEntryPoint)
                .and()
                //添加自定义过滤器到过滤器链中
                .addFilter(digestAuthenticationFilter());

    }
}

```



## 授权

在 Spring Security 的授权体系中，有两个关键接口：

1. AccessDecisionManager

AccessDecisionManager (访问决策管理器)，用来决定此次访问是否被允许。

```java
public interface AccessDecisionManager {
    void decide(Authentication authentication, Object object, Collection<ConfigAttribute> configAttributes) throws AccessDeniedException, InsufficientAuthenticationException;

    boolean supports(ConfigAttribute attribute);

    boolean supports(Class<?> clazz);
}
```

2. AccessDecisionVoter

AccessDecisionVoter (访问决定投票器)，投票器会检查用户是否具备应有的角色，进而投出赞成、反对或者弃权票。

```java
public interface AccessDecisionVoter<S> {
    int ACCESS_GRANTED = 1;
    int ACCESS_ABSTAIN = 0;
    int ACCESS_DENIED = -1;

    boolean supports(ConfigAttribute attribute);

    boolean supports(Class<?> clazz);

    int vote(Authentication authentication, S object, Collection<ConfigAttribute> attributes);
}
```

3. ConfigAttribute

ConfigAttribute，用来保存授权时的角色信息。

```java
public interface ConfigAttribute extends Serializable {
    String getAttribute();
}
```

在 Spring Security 中，用户请求一个资源(通常是一个接口或者一个 Java 方法)需要的角色会被封装成一个 ConfigAttribute 对象，在 ConfigAttribute 中只有一个 getAttribute方法，该方法返回一个 String 字符串，就是角色的名称。一般来说，角色名称都带有一个 ROLE_ 前缀，投票器 AccessDecisionVoter 所做的事情，其实就是比较用户所具各的角色和请求某个资源所需的 ConfigAtuibute 之间的关系。



## 自定义用户名和密码

### 基础

当什么也没有配置的时候，账号和密码是由 Spring Security 定义生成的。而在实际项目中账号和密码都是从数据库中查询出来的。 所以要通过自定义逻辑控制认证逻辑。

需要自定义逻辑时，只需要实现 UserDetailsService 接口即可。

```java
package org.springframework.security.core.userdetails;

public interface UserDetailsService {
    UserDetails loadUserByUsername(String username) throws UsernameNotFoundException;
}
```

**返回值 UserDetails**

```java
public interface UserDetails extends Serializable {
    Collection<? extends GrantedAuthority> getAuthorities(); // 表示获取登录用户所有权限
    String getPassword(); // 表示获取密码
    String getUsername(); // 表示获取用户名
    boolean isAccountNonExpired(); // 表示判断账户是否过期
    boolean isAccountNonLocked(); // 表示判断账户是否被锁定
    boolean isCredentialsNonExpired(); // 表示凭证{密码}是否过期
    boolean isEnabled(); // 表示当前用户是否可用
}
```

该接口有三个实现类

![image-20231224142058662](typora文档图片/image-20231224142058662.png)

**使用的时候使用 User**。

**密码加密**

PasswordEncoder 接口

```java
public interface PasswordEncoder {
    String encode(CharSequence rawPassword); // 表示把参数按照特定的解析规则进行解
    boolean matches(CharSequence rawPassword, String encodedPassword);
    // 表示验证从存储中获取的编码密码与编码后提交的原始密码是否匹配。如果密码匹配，则返回 true；如果不匹配，则返回 false。第一个参数表示需要被解析的密码。第二个参数表示存储的密码。
    default boolean upgradeEncoding(String encodedPassword) {
        return false;
    }
    // 表示如果解析的密码能够再次进行解析且达到更安全的结果则返回 true，否则返回false。默认返回 false。
}
```

**实现类**

![image-20231224143736234](typora文档图片/image-20231224143736234.png)

`BCryptPasswordEncoder` 是 Spring Security 官方推荐的密码解析器，平时多使用这个解析器。 BCryptPasswordEncoder 是对 bcrypt 强散列方法的具体实现。是基于 Hash 算法实现的单向加密。可以通过 strength 控制加密强度，默认 10.

**示例**

```java
@Test
public void test01(){
    // 创建密码解析器
    BCryptPasswordEncoder bCryptPasswordEncoder = new BCryptPasswordEncoder();
    // 对密码进行加密
    String atguigu = bCryptPasswordEncoder.encode("atguigu");
    // 打印加密之后的数据
    System.out.println("加密之后数据：\t"+atguigu);
    //判断原字符加密后和加密之前是否匹配
    boolean result = bCryptPasswordEncoder.matches("atguigu", atguigu);
    // 打印比较结果
    System.out.println("比较结果：\t"+result);
}

```



### SpringSecurity Web 权限方案

#### 设置登录账号以及密码

- 使用application.properties

    ```properties
    # 配置用户名
    spring.security.user.name=root
    # 配置密码
    spring.security.user.password=123456
    ```

- 编写类实现接口

    ```java
    @Configuration
    public class SecurityConfig {
        // 注入 PasswordEncoder 类到 spring 容器中
        // 即向IOC容器里注入一个PasswordEncoder，用于生成密码的base64编码的字符串，和解析base64编码的字符串为实际密码内容。
        @Bean
        public PasswordEncoder passwordEncoder(){
            return new BCryptPasswordEncoder();
        }
    }
    
    
    
    @Service
    public class LoginService implements UserDetailsService {
        @Override
        public UserDetails loadUserByUsername(String username) throws 
            UsernameNotFoundException {
            // 判断用户名是否存在
            if (!"admin".equals(username)){
                throw new UsernameNotFoundException("用户名不存在！");
            }
            // 从数据库中获取的密码 atguigu 的密文
            String pwd = "$2a$10$2R/M6iU3mCZt3ByG7kwYTeeW0w7/UqdeXrb27zkBIizBvAven0/na";
            // 第三个参数表示权限
            return new User(username,pwd,AuthorityUtils.commaSeparatedStringToAuthorityList("admin,"));
        }
    }
    ```

    

2.通过创建配置类实现设置

> 将用户名和密码写在配置类里，虽然配置类中可以自己编写用户名和密码的代码，但因为它是配置类的缘故，不适合将从数据库中获取用户名和密码的业务代码写入到配置类中。

```java
@Configuration
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        //用于密码的密文处理
        BCryptPasswordEncoder passwordEncoder = new BCryptPasswordEncoder();
        //生成密文
        String password = passwordEncoder.encode("123456");
        //设置用户名和密码
        auth.inMemoryAuthentication().withUser("qfAdmin").password(password).roles("admin");
    }
  
    @Bean
    PasswordEncoder passwordEncoder(){
        return new BCryptPasswordEncoder();
    }
}
```

3.编写自定义实现类（常用）

- 设计数据库的用户表

- 引入Mybatis和连接池的依赖

```xml
        <!--        mysql驱动-->
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
        </dependency>

        <!--        druid连接-->
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>druid-spring-boot-starter</artifactId>
            <version>1.1.10</version>
        </dependency>

        <!--        mybatis-->
        <dependency>
            <groupId>org.mybatis.spring.boot</groupId>
            <artifactId>mybatis-spring-boot-starter</artifactId>
            <version>1.3.2</version>
        </dependency>
```

- 编写application.properties配置文件

```properties
# 指明mapper映射文件的位置
mybatis.mapper-locations=classpath:mapper/*.xml
# 配置连接池Druid
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver
spring.datasource.url=jdbc:mysql://localhost:3306/db_security?serverTimezone=Asia/Shanghai
spring.datasource.username=root
spring.datasource.password=123456
spring.datasource.type=com.alibaba.druid.pool.DruidDataSource
```

- 编写UserDetailService实现类

    从数据库中获取用户名和密码的业务

```java
@Service
public class MyUserDetailService implements UserDetailsService {

    @Autowired
    private SysUserMapper userMapper;

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        //设置角色
        List<GrantedAuthority> auths = AuthorityUtils.commaSeparatedStringToAuthorityList("user");
        //可以从数据库获取用户名和密码
        if(StringUtils.isNullOrEmpty(username)){
            return null;
        }
        SysUser sysUser = userMapper.selectByUsername(username);
        User user = null;
        if(Objects.nonNull(sysUser)){
            user = new User(username,sysUser.getPassword(),auths);
        }
        return user;
    }
}

```

- 编写SecurityConfig配置类，指明对UserDetailsService实现类认证

```java
    @Autowired
    private UserDetailsService userDetailsService;

    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.userDetailsService(userDetailsService).passwordEncoder(passwordEncoder());
    }
```



## 角色和权限

### 概念

所谓**权限**，就是用户是否有访问当前页面，或者是执行某个操作的权利。

所谓**角色**，是对权限的汇总，比如“管理员”角色，可以对数据进行增删改查，增删改查是数据的四个权限，拥有“管理员”角色的用户拥有这四个权限。“普通用户”角色，只具备数据的增和查两种权限，那么拥有“普通用户”角色的用户只拥有这两个权限。

Spring Security提供了四个方法用于角色和权限的访问控制。通过这些方法，对用户是否具有某个或某些权限，进行过滤访问。对用户是否具备某个或某些角色，进行过滤访问：

- hasAuthority
- hasAnyAuthority
- hasRole
- hasAnyRole

### **hasAuthority方法**

判断当前主体是否有指定的权限，有返回true，否则返回false。

注意：该方法适用于只拥有一个权限的用户。

在配置类中设置当前主体具有怎样的权限才能访问。

示例

![image-20231224145613020](typora文档图片/image-20231224145613020.png)

```java
@EnableWebSecurity
public class PermissionConfig extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        //配置没有权限的跳转页面
        http.exceptionHandling().accessDeniedPage("/nopermission.html");
        http.formLogin()
                .loginPage("/login.html") //设置自定义登陆页面
                .loginProcessingUrl("/login") //登陆时访问的路径
                .failureUrl("/error.html")//登陆失败的页面
                .defaultSuccessUrl("/index.html").permitAll() //登陆成功后跳转的路径
                .and().authorizeRequests()
                .antMatchers("/","/login").permitAll() //设置可以直接访问的路径，取消拦截
                //1.hasAuthority方法：当前登陆用户，只有具有admin权限才可以访问这个路径
                .antMatchers("/index.html").hasAuthority("26")
                .anyRequest().authenticated()
                .and().csrf().disable(); //关闭csrf防护
    }

    @Autowired
    private UserDetailsService userDetailsService;

    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.userDetailsService(userDetailsService).passwordEncoder(passwordEncoder());
    }


    @Bean
    PasswordEncoder passwordEncoder(){
        return new BCryptPasswordEncoder();
    }
}

```

- 从数据库查询权限的Service

```java

@Service
public class PermissionServiceImpl implements PermissionService {

    @Autowired
    private SysRoleUserMapper roleUserMapper;

    @Autowired
    private SysRolePermissionMapper rolePermissionMapper;

    @Autowired
    private SysUserMapper userMapper;


    @Override
    public List<Integer> getPermissonsByName(String username) {

        if(StringUtils.isNullOrEmpty(username)){
            return null;
        }
        SysUser sysUser = userMapper.selectByUsername(username);
        List<Integer> permissionIds = new ArrayList<>();
        if(Objects.nonNull(sysUser)){
            Integer id = sysUser.getId();
            List<Integer> roleIds = roleUserMapper.selectByUserId(id);
            if(!CollectionUtils.isEmpty(roleIds)){
                //查询全选
                roleIds.forEach(rid -> {
                    List<Integer> pIds = rolePermissionMapper.selectByRoleId(rid);
                    permissionIds.addAll(pIds);
                });
                //去重
                Set<Integer> pSet = new HashSet<>(permissionIds);
                permissionIds.clear();
                permissionIds.addAll(pSet);

            }
        }
        return permissionIds;
    }
}

```

- 在userdetailsService，为返回的User对象设置权限

```java

@Service
public class MyUserDetailService implements UserDetailsService {

    @Autowired
    private SysUserMapper userMapper;

    @Autowired
    private PermissionService permissionService;


    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        if(StringUtils.isNullOrEmpty(username)){
            return null;
        }
        //从数据库获得该用户相关的权限
        List<Integer> permissons = permissionService.getPermissonsByName(username);
        //设置权限
        List<GrantedAuthority> auths = AuthorityUtils.commaSeparatedStringToAuthorityList(
                permissons.stream().map(String::valueOf).collect(Collectors.joining(",")));
        SysUser sysUser = userMapper.selectByUsername(username);
        User user = null;
        if(Objects.nonNull(sysUser)){
            user = new User(username,sysUser.getPassword(),auths);
        }
        return user;
    }
}

```

### **hasAnyAuthority方法**

适用于一个主体有多个权限的情况，多个权限用逗号隔开。

如果当前的主体有任何提供的角色（给定的作为一个逗号分隔的字符串列表）的话，返回 true.

```java


@EnableWebSecurity
public class PermissionConfig extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        //配置没有权限的跳转页面
        http.exceptionHandling().accessDeniedPage("/nopermission.html");
        http.formLogin()
                .loginPage("/login.html") //设置自定义登陆页面
                .loginProcessingUrl("/login") //登陆时访问的路径
                .failureUrl("/error.html")//登陆失败的页面
                .defaultSuccessUrl("/index.html").permitAll() //登陆成功后跳转的路径
                .and().authorizeRequests()
                .antMatchers("/","/login").permitAll() //设置可以直接访问的路径，取消拦截
                //1.hasAuthority方法：当前登陆用户，只有具有admin权限才可以访问这个路径
                .antMatchers("/index.html").hasAnyAuthority("26,9")
                .anyRequest().authenticated()
                .and().csrf().disable(); //关闭csrf防护
    }

    @Autowired
    private UserDetailsService userDetailsService;

    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.userDetailsService(userDetailsService).passwordEncoder(passwordEncoder());
    }


    @Bean
    PasswordEncoder passwordEncoder(){
        return new BCryptPasswordEncoder();
    }
}


```

### hasRole方法

如果用户具备给定角色就允许访问，否则报403错误。

![image-20231224145932486](typora文档图片/image-20231224145932486.png)

- 修改配置类

```java
@Override
    protected void configure(HttpSecurity http) throws Exception {
        //配置没有权限的跳转页面
        http.exceptionHandling().accessDeniedPage("/nopermission.html");
        http.formLogin()
                .loginPage("/login.html") //设置自定义登陆页面
                .loginProcessingUrl("/login") //登陆时访问的路径
                .failureUrl("/error.html")//登陆失败的页面
                .defaultSuccessUrl("/index.html").permitAll() //登陆成功后跳转的路径
                .and().authorizeRequests()
                .antMatchers("/","/login").permitAll() //设置可以直接访问的路径，取消拦截
                .antMatchers("/index.html").hasRole("admin")
                .anyRequest().authenticated()
                .and().csrf().disable(); //关闭csrf防护
    }
```

- 在PermissionServiceImpl添加获得角色的功能

```java
    @Override
    public List<Integer> getRoleByName(SysUser sysUser) {
        return roleUserMapper.selectByUserId(sysUser.getId());
    }
```

- 修改UserDetailsService

```java
//权限设置
@Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        //根据用户输入的用户名去数据库查询具体的用户对象
        if(StringUtils.isNullOrEmpty(username)){
            return null;
        }
        //数据库查询
        SysUser sysUser = userMapper.selectByUsername(username);
        User user = null;
        if(Objects.nonNull(sysUser)){
            //从数据库获得该用户相关的权限
        		List<Integer> permissons = permissionService.getPermissonsByName(username);
        		String perString = permissons.stream().map(String::valueOf).collect(Collectors.joining(","));

          	//从数据库获得该用户的角色
        		SysUser sysUser = userMapper.selectByUsername(username);
        		List<Integer> roles = permissionService.getRoleByName(sysUser);
        		String roleString = roles.stream().map(num -> "ROLE_" + num).collect(Collectors.joining(","));

        		//设置权限
        		List<GrantedAuthority> auths = AuthorityUtils.commaSeparatedStringToAuthorityList(perString+","+roleString);

            user = new User(username,sysUser.getPassword(),auths);
        }
        return user;

    }
```

> 其中角色student需要在设置时加上“ROLE_”前缀，因为通过源码hasRole方法给自定义的角色名前加上了“ROLE_”前缀

```java
private static String hasRole(String role) {
        Assert.notNull(role, "role cannot be null");
        Assert.isTrue(!role.startsWith("ROLE_"), () -> {
            return "role should not start with 'ROLE_' since it is automatically inserted. Got '" + role + "'";
        });
        return "hasRole('ROLE_" + role + "')";
    }
```



### hasAnyRole方法

设置多个角色，多个角色之间使用逗号隔开，只要用户具有某一个角色，就能访问。

![image-20231224145953618](typora文档图片/image-20231224145953618.png)

![image-20231224150019864](typora文档图片/image-20231224150019864.png)

```java
@Override
    protected void configure(HttpSecurity http) throws Exception {
        //配置没有权限的跳转页面
        http.exceptionHandling().accessDeniedPage("/nopermission.html");
        http.formLogin()
                .loginPage("/login.html") //设置自定义登陆页面
                .loginProcessingUrl("/login") //登陆时访问的路径
                .failureUrl("/error.html")//登陆失败的页面
                .defaultSuccessUrl("/index.html").permitAll() //登陆成功后跳转的路径
                .and().authorizeRequests()
                .antMatchers("/","/login").permitAll() //设置可以直接访问的路径，取消拦截
                .antMatchers("/index.html").hasAnyRole("role1","role2")
                .anyRequest().authenticated()
                .and().csrf().disable(); //关闭csrf防护
    }
```



## 自动登陆

1. 准备数据库表

创建persistent_logins表，用于持久化自动登陆的信息。

```sql
create table persistent_logins (username varchar(64) not null, series varchar(64) primary key, token varchar(64) not null, last_used timestamp not null)
```

2.实现自动登陆

- 修改SecurityConfig配置类

```java
package com.qf.my.ss.demo.config;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.security.config.annotation.authentication.builders.AuthenticationManagerBuilder;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.web.authentication.rememberme.JdbcTokenRepositoryImpl;
import org.springframework.security.web.authentication.rememberme.PersistentTokenRepository;

import javax.sql.DataSource;

/**
 * @author Thor
 * @公众号 Java架构栈
 */
@EnableWebSecurity
public class PermissionConfig extends WebSecurityConfigurerAdapter {

    @Autowired
    private DataSource dataSource;

    @Override
    protected void configure(HttpSecurity http) throws Exception {

        //配置数据源
        JdbcTokenRepositoryImpl tokenRepository = new JdbcTokenRepositoryImpl();
        tokenRepository.setDataSource(dataSource);
        

        //配置没有权限的跳转页面
        http.exceptionHandling().accessDeniedPage("/nopermission.html");
        http.formLogin()
                .loginPage("/login.html") //设置自定义登陆页面
                .loginProcessingUrl("/login") //登陆时访问的路径
                .failureUrl("/error.html")//登陆失败的页面
                .defaultSuccessUrl("/index.html").permitAll() //登陆成功后跳转的路径
                .and().authorizeRequests()
                .antMatchers("/","/login").permitAll() 
                .antMatchers("/index.html").hasRole("1")
                .anyRequest().authenticated()
                //开启记住我功能
                .and().rememberMe().userDetailsService(userDetailsService)
                //持久化令牌方案
                .tokenRepository(tokenRepository)
                //设置令牌有效期，为7天有效期
                .tokenValiditySeconds(60*60*24*7)
                .and().csrf().disable(); //关闭csrf防护
    }

    @Autowired
    private UserDetailsService userDetailsService;

    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.userDetailsService(userDetailsService).passwordEncoder(passwordEncoder());
    }


    @Bean
    PasswordEncoder passwordEncoder(){
        return new BCryptPasswordEncoder();
    }
}

```

- 前端页面添加自动登陆表单项

```html
                <div class="checkbox">
                    <label>
                        <input type="checkbox" name="remember-me"> 记住我
                    </label>
                </div>
```

3.自动登陆底层实现逻辑

> - 首先从前端传来的 cookie 中解析出 series 和 token；
>
> - 根据 series 从数据库中查询出一个 PersistentRememberMeToken 实例；
>
> - 如果查出来的 token 和前端传来的 token 不相同，说明账号可能被人盗用(别人用你的令牌登录之后，token 会变)。此时根据用户名移除相关的 token，相当于必须要重新输入用户名密码登录才能获取新的自动登录权限。
>
> - 接下来校验 token 是否过期;
>
> - 构造新的 PersistentRememberMeToken 对象，并且更新数据库中的 token(这就是我们文章开头说的，新的会话都会对应一个新的 token)；
>
> - 将新的令牌重新添加到 cookie 中返回；
>
> - 根据用户名查询用户信息，再走一波登录流程。
>
> 

## 用户注销

![](typora文档图片/用户注销流程.png)

1.在配置类添加注销的配置

```java
 @Override
    protected void configure(HttpSecurity http) throws Exception {
        //注销的配置
        http.logout().logoutUrl("/logout") //注销时访问的路径
                .logoutSuccessUrl("/logoutSuccess").permitAll(); //注销成功后访问的路径

        //配置没有权限的跳转页面
        http.exceptionHandling().accessDeniedPage("/error.html");
        http.formLogin()
                .loginPage("/login.html") //设置自定义登陆页面
                .loginProcessingUrl("/usr/login") //登陆时访问的路径
//                .defaultSuccessUrl("/index").permitAll() //登陆成功后跳转的路径
                .defaultSuccessUrl("/success.html").permitAll() //登陆成功后跳转的路径
                .and().authorizeRequests()
                    .antMatchers("/","/add","/user/login").permitAll() //设置可以直接访问的路径，取消拦截
                    //1.hasAuthority方法：当前登陆用户，只有具有admin权限才可以访问这个路径
                    //.antMatchers("/index").hasAuthority("admin")
                    //2.hasAnyAuthority方法：当前登陆用户，具有admin或manager权限可以访问这个路径
                    //.antMatchers("/index").hasAnyAuthority("admin,manager")
                    //3.hasRole方法：当前主体具有指定角色，则允许访问
                    //.antMatchers("/index").hasRole("student")
                    //4.hasAnyRole方法：当前主体只要具备其中某一个角色就能访问
                    .antMatchers("/index").hasAnyRole("student1,teacher")
                .anyRequest().authenticated()
                .and().csrf().disable(); //关闭csrf防护
    }
```

2.设置注销链接

添加success.html页面作为登陆成功后的跳转页面

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
</head>
<body>
    登陆成功 <a href="/logout">退出</a>
</body>
</html>
```

登陆后访问退出按钮，实现注销功能。

## JWT

即 Json Web Token

1.基于Token的认证方式

> 使用基于Token的身份验证方法，在服务端不需要存储用户的登陆信息。流程如下：
>
> - 客户端使用用户名和密码请求登陆。
> - 服务端收到请求，去验证用户名和密码。
> - 验证成功后，服务端会签发一个Token，再把这个Token发送给客户端。
> - 客户端收到Token以后可以把它存储在Cookie本地。
> - 客户端每次向服务端请求资源时需要携带Cookie中该Token。
> - 服务端收到请求后，验证客户端携带的Token，如果验证成功则返回数据。

<img src="typora文档图片/jwt认证.png" alt="jwt认证" style="zoom:70%;" />

2.什么是JWT

> JSON Web Token （JWT）是一个开放的行业标准（RFC 7519），它定义了一种简洁的、自包含的协议格式，用于在通信双方传递json对象，传递的信息经过数字签名可以被验证和信任。JWT可以使用HMAC算法或使用RSA的公钥/私钥对进行签名，防止被篡改。
>
> JWT官网： https://jwt.io
>
> JWT令牌的优点：
>
> - JWT基于json，非常方便解析。
> - 可以在令牌中自定义丰富的内容，易扩展。
> - 通过非对称加密算法及数字签名技术，JWT防止篡改，安全性高。
> - 资源服务使用JWT可不依赖认证服务即完成授权。
>
> JWT令牌的缺点：
>
> - JWT令牌较长，占存储空间比较大。



3.JWT组成

> 一个JWT实际上就一个字符串，它由三部分组成，头部、负载与签名。



1）头部（Header）

> 头部用于描述关于该JWT的最基本信息，例如其类型（即JWT）以及签名所用的算法（如HMAC SHA256 或 RSA）等。这也可以被表示成一个JSON对象。

```json
{
  "alg":"HS256",
  "typ":"JWT"
}
```

> - alg：签名算法
> - typ：类型
>
> 我们对头部的json字符串进行BASE64编码，编码后的字符串如下：

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9
```

> Base64是一种基于64个可打印字符串来表示二进制数据的表示方式。JDK提供了非常方便的Base64Encoder和Base64Decoder，用它们可以非常方便的完成基于Base64的编码和解码。



2）负载（Payload）

> 负载，是存放有效信息的地方，比如用户的基本信息可以存在该部分中。负载包含三个部分：
>
> - 标准中注册的声明（建议但不强制使用）
>     - iss：jwt签发者
>     - sub：jwt所面向的用户
>     - aud：接收jwt的一方
>     - exp：jwt的过期时间，过期时间必须大于签发时间
>     - nbf：定义在什么时间之前，该jwt都是不可用的
>     - iat：jwt的签发时间
>     - jti：jwt的唯一身份标识，主要用来作为一次性token，从而回避重放攻击。
>
> - 公共的声明
>
> 公共的声明可以添加任何信息，一般添加用户的相关信息或其他业务需要的必要信息，但不建议添加敏感信息，因为该部分在客户端可解密。
>
> - 私有的声明
>
> 私有声明是提供者和消费者所共同定义的声明，一般不建议存放敏感信息，因为base64是对称解密的，意味着该部分信息可以归类为明文信息。
>
> 私有声明也就是自定义claim，用于存放自定义键值对。

```json
{
  "sub": "1234567890",
  "name": "John Doe",
  "iat": 1516239022
}
```

> 其中sub是标准的声明，name是自定义的私有声明，编码后如下：

```
eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ
```



3）签证、签名（Signature）

> jwt的第三部分是一个签证信息，由三部分组成：
>
> - Header（Base64编码后）
> - Payload（Base64编码后）
> - Secret（盐，必须保密）
>
> 这个部分需要Base64加密后的header和base4加密后的payload使用.连接组成的字符串，然后通过header重声明的加密方式进行加盐Secret组合加密，然后就构成了JWT的第三部分——使用“qfjava”作为盐：

```
eZqdTo1mRMB-o7co1oAiTvNvumfCkt-1H-CdfNm78Cw
```

> 从官方工具中可以看到，三个部分组合出的完整字符串：

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.eZqdTo1mRMB-o7co1oAiTvNvumfCkt-1H-CdfNm78Cw
```

![image-20210310163620788](../../../../zeleishi/Documents/工作/精品视频/springsecurity/pic/jwttoken.png)



> 注意：secret是保存在服务器端的，jwt在签发生成也是在服务器端的，secret就是用来进行jwt的签发和验证，所以，它就是服务器端的私钥，在任何场景都不应该泄漏。一旦客户端得知这个secret，那就意味着客户端是可以自我签发jwt了。

4.使用JJWT

JJWT是一个提供端到端的JWT创建和验证的开源Java库。也就是说使用JJWT能快速完成JWT的功能开发。

- 引入依赖

> 创建Springboot工程并引入jjwt依赖，pom.xml如下：

```xml
        <!--jjwt-->
        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt</artifactId>
            <version>RELEASE</version>
        </dependency>
```

- 创建Token

```java
    @Test
    public void testCrtToken(){

        //创建JWT对象
        JwtBuilder builder = Jwts.builder().setId("1001")//设置负载内容
                .setSubject("小明")
                .setIssuedAt(new Date())//设置签发时间
                .signWith(SignatureAlgorithm.HS256, "qfjava");//设置签名秘钥
        //构建token
        String token = builder.compact();
        System.out.println(token);

    }
```



> JWT将用户信息转换成Token字符串，生成结果如下：

```
eyJhbGciOiJIUzI1NiJ9.eyJqdGkiOiIxMDAxIiwic3ViIjoi5bCP5piOIiwiaWF0IjoxNjE1MzY2MDEyfQ.2LNcw1v64TNQ96eCpWKvtAccBUA-cEVMDyJNMef-zu0
```



- 解析Token

> 通过JWT解析Token，获取Token中存放的用户信息，即生成Claims对象。

```java
    @Test
    public void testParseToken(){
        String token = "eyJhbGciOiJIUzI1NiJ9.eyJqdGkiOiIxMDAxIiwic3ViIjoi5bCP5piOIiwiaWF0IjoxNjE1MzY2MDEyfQ.2LNcw1v64TNQ96eCpWKvtAccBUA-cEVMDyJNMef-zu0";
        //解析Token，生成Claims对象，Token中存放的用户信息解析到了claims对象中
        Claims claims = Jwts.parser().setSigningKey("qfjava").parseClaimsJws(token).getBody();
        System.out.println("id:" + claims.getId());
        System.out.println("subject:" + claims.getSubject());
        System.out.println("IssuedAt:" + claims.getIssuedAt());
    }
```

> 解析结果如下：

```
id:1001
subject:小明
IssuedAt:Wed Mar 10 16:46:52 CST 2021
```



- Token过期检验

> 在有效期内Token可以正常读取，超过有效期则Token失效

```java
    @Test
    public void testExpToken(){
        long now = System.currentTimeMillis();  //当前时间
        long exp = now + 1000 * 60; //过期时间为1分钟
        JwtBuilder builder = Jwts.builder().setId("1001")
                .setSubject("小明")
                .setIssuedAt(new Date())
                .signWith(SignatureAlgorithm.HS256, "qfjava")
                .setExpiration(new Date(exp));//设置超时
    }
```



- 自定义claims

> 除了使用官方api设置属性值，也可以添加自定义键值对。

```java
    @Test
    public void testCustomClaims(){
        long now = System.currentTimeMillis();  //当前时间
        long exp = now + 1000 * 60; //过期时间为1分钟
        JwtBuilder builder = Jwts.builder().setId("1001")
                .setSubject("小明")
                .setIssuedAt(new Date())
                .signWith(SignatureAlgorithm.HS256, "qfjava")
                .setExpiration(new Date(exp))
                .claim("role", "admin");//设置自定义键值对
    }
```

> 使用下面语句获取属性值：

```java
claims.get("role")
```

## 微服务项目-使用Security+JWT实现权限管理

1.前后端分离的权限管理

![Spring Security + jwt 前后端分离的权限系统的时序图](typora文档图片/Spring Security + jwt 前后端分离的权限系统的时序图.png)

2.引入依赖

```xml
        <!--redis-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
        </dependency>

        <!--spring security-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-security</artifactId>
        </dependency>

        <!--mysql驱动-->
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
        </dependency>

        <!--druid连接-->
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>druid-spring-boot-starter</artifactId>
            <version>1.1.10</version>
        </dependency>

        <!--mybatis-->
        <dependency>
            <groupId>org.mybatis.spring.boot</groupId>
            <artifactId>mybatis-spring-boot-starter</artifactId>
            <version>1.3.2</version>
        </dependency>

        <!--jjwt-->
        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt</artifactId>
            <version>RELEASE</version>
        </dependency>

        <!--lombok-->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
        </dependency>

```

3.登陆过滤器的实现

```java
package com.qf.my.security.admin.demo.filter;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.qf.my.security.admin.demo.common.ResponseUtil;
import com.qf.my.security.admin.demo.common.ResultModel;
import com.qf.my.security.admin.demo.entity.SecurityUser;
import com.qf.my.security.admin.demo.entity.User;
import com.qf.my.security.admin.demo.security.TokenManager;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.AuthenticationException;
import org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter;
import org.springframework.security.web.util.matcher.AntPathRequestMatcher;

import javax.servlet.FilterChain;
import javax.servlet.ServletException;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;
import java.util.ArrayList;

/**
 * @author Thor
 * @公众号 Java架构栈
 */
public class TokenLoginFilter extends UsernamePasswordAuthenticationFilter {

    private TokenManager tokenManager;
    private RedisTemplate redisTemplate;
    private AuthenticationManager authenticationManager;

    public TokenLoginFilter(AuthenticationManager authenticationManager,TokenManager tokenManager, RedisTemplate redisTemplate) {
        this.tokenManager = tokenManager;
        this.redisTemplate = redisTemplate;
        this.authenticationManager = authenticationManager;
        //不是只允许post请求，经过这个filter
        this.setPostOnly(false);
        //设置登陆的路径和请求方式
        this.setRequiresAuthenticationRequestMatcher(new AntPathRequestMatcher("/user/login","POST"));
    }

    /**
     * 执行认证的方法
     * @param request
     * @param response
     * @return
     * @throws AuthenticationException
     */
    @Override
    public Authentication attemptAuthentication(HttpServletRequest request, HttpServletResponse response) throws AuthenticationException {
        //获取表单提供的数据
        ObjectMapper objectMapper = new ObjectMapper();
        try {
            User user = objectMapper.readValue(request.getInputStream(), User.class);
            //校验==认证的过程
            Authentication authenticate = authenticationManager.authenticate(
                    new UsernamePasswordAuthenticationToken(user.getUsername(), user.getPassword()
                            , new ArrayList<>())
            );
            return authenticate;

        } catch (IOException e) {
            e.printStackTrace();
            throw new RuntimeException("认证失败");
        }

    }

    /**
     * 认证成功以后调用的方法
     * @param request
     * @param response
     * @param chain
     * @param authResult
     * @throws IOException
     * @throws ServletException
     */
    @Override
    protected void successfulAuthentication(HttpServletRequest request, HttpServletResponse response, FilterChain chain, Authentication authResult) throws IOException, ServletException {

        //得到用户名
        SecurityUser securityUser = (SecurityUser) authResult.getPrincipal();
        String username = securityUser.getUsername();
        //生成token
        String token = tokenManager.crtToken(username);
        //存入到redis  username: 权限
        redisTemplate.opsForValue().set(username,securityUser.getPermissionValueList());
        //返回token
        ResponseUtil.out(response, ResultModel.success(token));

    }

    /**
     * 认证失败调用的方法
     * @param request
     * @param response
     * @param failed
     * @throws IOException
     * @throws ServletException
     */
    @Override
    protected void unsuccessfulAuthentication(HttpServletRequest request, HttpServletResponse response, AuthenticationException failed) throws IOException, ServletException {
        ResponseUtil.out(response,ResultModel.error(401,failed.getMessage()));
    }
}

```

4.权限过滤器的实现

```java
package com.qf.my.security.admin.demo.filter;

import com.mysql.cj.util.StringUtils;
import com.qf.my.security.admin.demo.security.TokenManager;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter;
import org.springframework.security.web.authentication.www.BasicAuthenticationFilter;
import org.springframework.util.CollectionUtils;

import javax.servlet.FilterChain;
import javax.servlet.ServletException;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;
import java.util.ArrayList;
import java.util.Collection;
import java.util.Collections;
import java.util.List;

/**
 * @author Thor
 * @公众号 Java架构栈
 */
public class TokenAuthFilter extends BasicAuthenticationFilter {

    private TokenManager tokenManager;
    private RedisTemplate redisTemplate;

    public TokenAuthFilter(AuthenticationManager authenticationManager,
                           TokenManager tokenManager,RedisTemplate redisTemplate) {
        super(authenticationManager);
        this.tokenManager = tokenManager;
        this.redisTemplate = redisTemplate;
    }

    /**
     * 权限相关的操作
     * @param request
     * @param response
     * @param chain
     * @throws IOException
     * @throws ServletException
     */
    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain chain) throws IOException, ServletException {
        //获得token
        String token = request.getHeader("token");
        if(!StringUtils.isNullOrEmpty(token)){
            //使用jwt解析token获得username
            String username = tokenManager.getUsernameFromToken(token);
            //从redis中获得该用户名对应的权限
            List<String> permissionValueList = (List<String>) redisTemplate.opsForValue().get(username);
            //将取出的权限存入到权限上下文中，表示当前token对应的用户具备哪些权限
            Collection<GrantedAuthority> authorityCollection = new ArrayList<>();
            if(!CollectionUtils.isEmpty(permissionValueList)){
                for (String permissionValue : permissionValueList) {
                    SimpleGrantedAuthority authority = new SimpleGrantedAuthority(permissionValue);
                    authorityCollection.add(authority);
                }
            }
            //生成权限信息对象
            UsernamePasswordAuthenticationToken authenticationToken = new UsernamePasswordAuthenticationToken(username,token,authorityCollection);
            //把权限信息对象存入到权限上下文中
            SecurityContextHolder.getContext().setAuthentication(authenticationToken);
        }
        //放行
        chain.doFilter(request,response);
    }
}

```

5.注销处理器的实现

```java
package com.qf.my.security.admin.demo.security;

import com.mysql.cj.util.StringUtils;
import com.qf.my.security.admin.demo.common.ResponseUtil;
import com.qf.my.security.admin.demo.common.ResultModel;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.security.core.Authentication;
import org.springframework.security.web.authentication.logout.LogoutHandler;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

/**
 * @author Thor
 * @公众号 Java架构栈
 */
public class TokenLogoutHandler implements LogoutHandler {

    private TokenManager tokenManager;
    private RedisTemplate redisTemplate;

    public TokenLogoutHandler(TokenManager tokenManager, RedisTemplate redisTemplate) {
        this.tokenManager = tokenManager;
        this.redisTemplate = redisTemplate;
    }

    /**
     * 注销时具体要执行的业务
     * @param request
     * @param response
     * @param authentication
     */
    @Override
    public void logout(HttpServletRequest request, HttpServletResponse response, Authentication authentication) {
        //1.从请求头中获得前端携带的token
        String token = request.getHeader("token");
        if(!StringUtils.isNullOrEmpty(token)){
            //2.使用jwt解析token
            String username = tokenManager.getUsernameFromToken(token);
            //3.删除redis中的数据
            redisTemplate.delete(username);
        }
        ResponseUtil.out(response, ResultModel.success("注销成功"));
    }
}

```

6.用户名密码验证逻辑

```java
package com.qf.my.security.admin.demo.service.impl;

import com.qf.my.security.admin.demo.entity.SecurityUser;
import com.qf.my.security.admin.demo.entity.User;
import com.qf.my.security.admin.demo.service.PermissionService;
import com.qf.my.security.admin.demo.service.UserService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.core.userdetails.UsernameNotFoundException;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.Objects;

@Service("userDetailsService")
public class UserDetailsServiceImpl implements UserDetailsService {

    @Autowired
    private UserService userService;

    @Autowired
    private PermissionService permissionService;

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        //根据用户名从数据库查询到该用户的信息
        User user = userService.selectByUsername(username);
        if(Objects.isNull(user)) {
            throw new UsernameNotFoundException("当前用户不存在");
        }
        //根据用户名从数据库查询到该用户的权限信息
        List<String> permissionValues = permissionService.selectPermissionValueByUserId(user.getId());
        SecurityUser securityUser = new SecurityUser();
        securityUser.setCurrentUserInfo(user);
        securityUser.setPermissionValueList(permissionValues);
        return securityUser;
    }
}

```











在SpringSecurity框架中有三个非常核心的类和接口，分别是：

- SecurityFilterChain接口；
- FilterChainProxy类；
- DelegatingFilterProxy类；

这个三个接口和类的相互之间的步骤关系如下：

1. 生成一个`FilterChainProxy`类型的对象，该类有一个**属性**`filterChains`，是SecurityFilterChain类型的List集合；该对象被spring容器通过bean管理，名称为 `springSecurityFilterChain`， 类型为`FilterChainProxy`。

    底层通过`FilterChainProxy`代理去调用各种Filter(Filter链)，`Filter`通过调用`AuthenticationManager`完成认证 ，通过调用`AccessDecisionManager`完成授权。

    ```java
    // SecurityFilterChain
    public interface SecurityFilterChain {
        boolean matches(HttpServletRequest request);
    
        List<Filter> getFilters();
    }
    
    // FilterChainProxy
    public class FilterChainProxy extends GenericFilterBean {
        private List<SecurityFilterChain> filterChains;
    }
    ```

2. 生成一个DelegatingFilterProxy类型的对象，将名为springSecurityFilterChain的bean（上面的）作为`DelegatingFilterProxy`类对象的**属性targetBeanName的值**，供后面请求时获取bean。这样`FilterChainProxy`类型的对象就被`DelegatingFilterProxy`类型的对象委托管理了。

    注意：DelegatingFilterProxy对象的生成是tomcat启动过程中会调用所有继承了RegistrationBean类的onStartUp方法，最终调用了实现类中的addRegistration方法。

3. 前端发起请求时，调用了DelegatingFilterProxy类型的拦截器执行doFilter方法。doFilter方法获取被委托的对象FilterChainProxy并调用其doFilter方法（即FilterChainProxy的doFilter方法），执行获取到的所有的拦截器然后再获取代理对象执行容器加载时保存的拦截器再执行。

-------------------------------**第一步**-------------------------------

**生成一个FilterChainProxy类型的对象**

![image-20231225134414405](typora文档图片/image-20231225134414405.png)

可以看到请求访问时，`DelegatingFilterProxy`管理`FilterChainProxy`，`FilterChainProxy`里调用`SecurityFilterChain`类型的过滤器。

[Java SPI 机制](https://blog.csdn.net/qq_37967783/article/details/131505676)

Spring的Factories就是Spring版本的Java Spi，我在关于java基础系列文章中有详细介绍Java SPI机制。 Spring Factories的最重要的功能就是：可以通过配置文件指定Spring容器加载一些特定的组件。

Spring Factories是一种类似于Java SPI的机制，它在META-INF/spring.factories文件中配置接口的实现类名称，然后在程序中读取这些配置文件并实例化。

在springboot启动过程中会获取spring.factories配置文件里的配置类并加载到spring容器中，观察spring.factories配置文件里的配置内容，涉及到springsecurity的如下图红框处所示。

![image-20231225170234268](typora文档图片/image-20231225170234268.png)

先看 `SecurityAutoConfiguration` 和 `SecurityFilterAutoConfiguration` 这两个配置类：

![image-20231225170325342](typora文档图片/image-20231225170325342.png)

**配置类是如何获取到 springSecurityFilterChain这个bean的**

分析`SecurityAutoConfiguration`配置类：

```java
@Configuration(
    proxyBeanMethods = false
)
@ConditionalOnClass({DefaultAuthenticationEventPublisher.class})
@EnableConfigurationProperties({SecurityProperties.class})
@Import({SpringBootWebSecurityConfiguration.class, WebSecurityEnablerConfiguration.class, SecurityDataConfiguration.class, ErrorPageSecurityFilterConfiguration.class})
public class SecurityAutoConfiguration {
    public SecurityAutoConfiguration() {
    }

    @Bean
    @ConditionalOnMissingBean({AuthenticationEventPublisher.class})
    public DefaultAuthenticationEventPublisher authenticationEventPublisher(ApplicationEventPublisher publisher) {
        return new DefaultAuthenticationEventPublisher(publisher);
    }
}
```

该配置类里导入的四个配置类：

- SpringBootWebSecurityConfiguration.class：作用是WebSecurityConfigurerAdapter 类存在但是bean对象不存在时注册默认的WebSecurityConfigurerAdapter 类型是DefaultConfigurerAdapter的bean。

    ```java
    @Configuration(
        proxyBeanMethods = false
    )
    @ConditionalOnDefaultWebSecurity
    @ConditionalOnWebApplication(
        type = Type.SERVLET
    )
    class SpringBootWebSecurityConfiguration {
        SpringBootWebSecurityConfiguration() {
        }
    
        @Bean
        @Order(2147483642)
        SecurityFilterChain defaultSecurityFilterChain(HttpSecurity http) throws Exception {
            ((HttpSecurity)((HttpSecurity)((ExpressionUrlAuthorizationConfigurer.AuthorizedUrl)http.authorizeRequests().anyRequest()).authenticated().and()).formLogin().and()).httpBasic();
            return (SecurityFilterChain)http.build();
        }
    }
    ```

    

- WebSecurityEnablerConfiguration.class：

    该类文件如下，可以看到其中声明了@EnableWebSecurity注解：

    ```java
    @Configuration(
        proxyBeanMethods = false
    )
    // 只有当BeanFactory中没有指定的bean的时候才能匹配，主要是用来做自动配置的，当程序没有配置指定的类的时候，就会使用默认配置
    @ConditionalOnMissingBean(
     
       name = {"springSecurityFilterChain"}
    )
    @ConditionalOnClass({EnableWebSecurity.class})
    @ConditionalOnWebApplication(
        type = Type.SERVLET
    )
    @EnableWebSecurity
    class WebSecurityEnablerConfiguration {
        WebSecurityEnablerConfiguration() {
        }
    }
    ```

    

- SecurityDataConfiguration.class：

- ErrorPageSecurityFilterConfiguration.class
