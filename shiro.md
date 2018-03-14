---

title: shiro
date: 2016-10-31 20:05:45
tags: Security
---

# Apache Shiro 

Apache Shiro是一个功能强大且易于使用的Java安全框架，可执行身份验证，授权，加密和会话管理。 使用Shiro易于理解的API，您可以快速轻松地保护任何应用程序 - 从最小的移动应用程序到最大的Web和企业应用程序。

Shiro提供应用程序安全API来执行以下几个方面:

* Authentication

* Authorization

* Cryptography

* Session Management


## 核心概念

### Subject

当前“用户”，这个用户不一定是一个具体的人，与当前应用交互的任何东西都是Subject，如网络爬虫，机器人等；即一个抽象概念；所有Subject都绑定到SecurityManager，与Subject的所有交互都会委托给SecurityManager；可以把Subject认为是一个门面；SecurityManager才是实际的执行者。

获取当前用户

```java
import org.apache.shiro.subject.Subject;
import org.apache.shiro.SecurityUtils;
...
Subject currentUser = SecurityUtils.getSubject();
```

### SecurityManager

安全管理器；即所有与安全有关的操作都会与SecurityManager交互；且它管理着所有Subject。

### Realms

~~一个Realm充当Shiro和应用程序安全数据之间的“桥梁”或“连接器”。可以把Realm看成DataSource，即安全数据源。~~

```java
protected Filter initDelegate(WebApplicationContext wac) throws ServletException {
		Filter delegate = wac.getBean(getTargetBeanName(), Filter.class);
		if (isTargetFilterLifecycle()) {
			delegate.init(getFilterConfig());
		}
		return delegate;
	}
```


AdviceFilter
```java
public void doFilterInternal(ServletRequest request, ServletResponse response, FilterChain chain)
           throws ServletException, IOException {

       Exception exception = null;

       try {

           boolean continueChain = preHandle(request, response);
           if (log.isTraceEnabled()) {
               log.trace("Invoked preHandle method.  Continuing chain?: [" + continueChain + "]");
           }

           if (continueChain) {
               executeChain(request, response, chain);
           }

           postHandle(request, response);
           if (log.isTraceEnabled()) {
               log.trace("Successfully invoked postHandle method");
           }

       } catch (Exception e) {
           exception = e;
       } finally {
           cleanup(request, response, exception);
       }
   }
```

## DefaultFilter
```java
public enum DefaultFilter {

    anon(AnonymousFilter.class),
    authc(FormAuthenticationFilter.class),
    authcBasic(BasicHttpAuthenticationFilter.class),
    logout(LogoutFilter.class),
    noSessionCreation(NoSessionCreationFilter.class),
    perms(PermissionsAuthorizationFilter.class),
    port(PortFilter.class),
    rest(HttpMethodPermissionFilter.class),
    roles(RolesAuthorizationFilter.class),
    ssl(SslFilter.class),
    user(UserFilter.class);

    private final Class<? extends Filter> filterClass;

    private DefaultFilter(Class<? extends Filter> filterClass) {
        this.filterClass = filterClass;
    }

    public Filter newInstance() {
        return (Filter) ClassUtils.newInstance(this.filterClass);
    }

    public Class<? extends Filter> getFilterClass() {
        return this.filterClass;
    }

    public static Map<String, Filter> createInstanceMap(FilterConfig config) {
        Map<String, Filter> filters = new LinkedHashMap<String, Filter>(values().length);
        for (DefaultFilter defaultFilter : values()) {
            Filter filter = defaultFilter.newInstance();
            if (config != null) {
                try {
                    filter.init(config);
                } catch (ServletException e) {
                    String msg = "Unable to correctly init default filter instance of type " +
                            filter.getClass().getName();
                    throw new IllegalStateException(msg, e);
                }
            }
            filters.put(defaultFilter.name(), filter);
        }
        return filters;
    }
}

```

# SecurityManager
```java
public interface SecurityManager extends Authenticator, Authorizer, SessionManager {

    Subject login(Subject subject, AuthenticationToken authenticationToken) throws AuthenticationException;

    void logout(Subject subject);

    Subject createSubject(SubjectContext context);

}
```

# SessionManager体系结构
![SessionManager体系结构](shiro/SessionManager.png)

## 会话监听
- SessionListener
- SessionListenerAdapter

## 会话存储/持久化
### 1、SessionDao
```java
public interface SessionDAO {

    Serializable create(Session session);

    Session readSession(Serializable sessionId) throws UnknownSessionException;

    void update(Session session) throws UnknownSessionException;

    void delete(Session session);

    Collection<Session> getActiveSessions();
}
```
### 1.1 SessionDao体系结构
![SessionDao体系结构](shiro/SessionDAO.png)  

AbstractSessionDAO 提供了 SessionDAO 的基础实现，如生成会话 ID 等；CachingSessionDAO 提供了对开发者透明的会话缓存的功能，只需要设置相应的 CacheManager 即可；MemorySessionDAO 直接在内存中进行会话维护；而 EnterpriseCacheSessionDAO 提供了缓存功能的会话维护，默认情况下使用 MapCache 实现，内部使用 ConcurrentHashMap 保存缓存的会话。


Shiro 提供了使用 Ehcache 进行会话存储，Ehcache 可以配合 TerraCotta 实现容器无关的分布式集群。
```
sessionDAO=org.apache.shiro.session.mgt.eis.EnterpriseCacheSessionDAO
sessionDAO. activeSessionsCacheName=shiro-activeSessionCache
sessionManager.sessionDAO=$sessionDAO
cacheManager = org.apache.shiro.cache.ehcache.EhCacheManager
cacheManager.cacheManagerConfigFile=classpath:ehcache.xml
securityManager.cacheManager = $cacheManager&nbsp;
```

### 会话ID生成器
### 自定义SessionDAO
自定义实现 SessionDAO，继承 CachingSessionDAO即可，如下：
```java
public class MySessionDAO extends CachingSessionDAO {
    private JdbcTemplate jdbcTemplate = JdbcTemplateUtils.jdbcTemplate();

    protected Serializable doCreate(Session session) {
        Serializable sessionId = generateSessionId(session);
        assignSessionId(session, sessionId);
        String sql = "insert into sessions(id, session) values(?,?)";
        jdbcTemplate.update(sql, sessionId, SerializableUtils.serialize(session));
        return session.getId();
    }

    protected void doUpdate(Session session) {
    if(session instanceof ValidatingSession && !((ValidatingSession)session).isValid()) {
        return; //如果会话过期/停止 没必要再更新了
    }
        String sql = "update sessions set session=? where id=?";
        jdbcTemplate.update(sql, SerializableUtils.serialize(session), session.getId());
    }

    protected void doDelete(Session session) {
        String sql = "delete from sessions where id=?";
        jdbcTemplate.update(sql, session.getId());
    }

    protected Session doReadSession(Serializable sessionId) {
        String sql = "select session from sessions where id=?";
        List<String> sessionStrList = jdbcTemplate.queryForList(sql, String.class, sessionId);
        if(sessionStrList.size() == 0) return null;
        return SerializableUtils.deserialize(sessionStrList.get(0));
    }
}
```
### 会话验证
Shiro 提供了会话验证调度器，用于定期的验证会话是否已过期，如果过期将停止会话；出于性能考虑，一般情况下都是获取会话时来验证会话是否过期并停止会话的；但是如在 web 环境中，如果用户不主动退出是不知道会话是否过期的，因此需要定期的检测会话是否过期，Shiro 提供了会话验证调度器 SessionValidationScheduler 来做这件事情。
```java
public interface SessionValidationScheduler {

    boolean isEnabled();

    void enableSessionValidation();

    void disableSessionValidation();
}

```

Shiro也提供了使用Quartz会话验证调度器(QuartzSessionValidationScheduler),使用时需要导入shiro-quartz依赖：
```xml
<dependency>
     <groupId>org.apache.shiro</groupId>
     <artifactId>shiro-quartz</artifactId>
     <version>1.2.2</version>
</dependency>
```

### sessionFactory
sessionFactory 是创建会话的工厂，根据相应的 Subject 上下文信息来创建会话；默认提供了 SimpleSessionFactory 用来创建 SimpleSession 会话。


shiro如何判定是否登陆？