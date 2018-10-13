---
layout: post
category: "java"
title:  "PageHelper分页插件原理剖析"
tags: [java,mybatis]
---

&#8194;PageHelper是一款开源免费的Mybatis第三方物理分页插件。它的使用方法非常简单，例如在Spring boot项目中，引入下面的maven依赖：

```
        <dependency>
            <groupId>com.github.pagehelper</groupId>
            <artifactId>pagehelper-spring-boot-starter</artifactId>
            <version>1.2.3</version>
        </dependency>

```
然后在需要分页的查询前面调用PageHelper的静态方法：

```
        Page<User> page = PageHelper.startPage(startPage, pageSize);
        List<User> policies = userMapper.selectUsers();

```
那么它是怎么帮我实现分页的呢？进入PageHelper源码：

```
@Intercepts(@Signature(type = Executor.class, method = "query", args = {MappedStatement.class, Object.class, RowBounds.class, ResultHandler.class}))
public class PageHelper implements Interceptor {
    //sql工具类
    private SqlUtil sqlUtil;
    //属性参数信息
    private Properties properties;
    //配置对象方式
    private SqlUtilConfig sqlUtilConfig;
    //自动获取dialect,如果没有setProperties或setSqlUtilConfig，也可以正常进行
    private boolean autoDialect = true;
    //运行时自动获取dialect
    private boolean autoRuntimeDialect;
    //多数据源时，获取jdbcurl后是否关闭数据源
    private boolean closeConn = true;
    //缓存
    private Map<String, SqlUtil> urlSqlUtilMap = new ConcurrentHashMap<String, SqlUtil>();
    private ReentrantLock lock = new ReentrantLock();
// ...
}

```

&#8194;SqlUtil：数据库类型专用sql工具类，一个数据库url对应一个SqlUtil实例，SqlUtil内有一个Parser对象，如果是mysql，它是MysqlParser，如果是oracle，它是OracleParser，这个Parser对象是SqlUtil不同实例的主要存在价值。执行count查询、设置Parser对象、执行分页查询、保存Page分页对象等功能，均由SqlUtil来完成。

&#8194;autoRuntimeDialect：多个数据源切换时，比如mysql和oracle数据源同时存在，就不能简单指定dialect，这个时候就需要运行时自动检测当前的dialect。

&#8194;Map<String, SqlUtil> urlSqlUtilMap：它就用来缓存autoRuntimeDialect自动检测结果的，key是数据库的url，value是SqlUtil。由于这种自动检测只需要执行1次，所以做了缓存。

&#8194;ReentrantLock lock：这个lock对象是比较有意思的现象，urlSqlUtilMap明明是一个同步ConcurrentHashMap，又搞了一个lock出来同步ConcurrentHashMap做什么呢？是否是画蛇添足？在《Java并发编程实战》一书中有详细论述，简单的说，ConcurrentHashMap可以保证put或者remove方法一定是线程安全的，但它不能保证put、get、remove的组合操作是线程安全的，为了保证组合操作也是线程安全的，所以使用了lock。

```
public class PageStaticSqlSource extends PageSqlSource {
    private String sql;
    private List<ParameterMapping> parameterMappings;
    private Configuration configuration;
    private SqlSource original;
    
    @Override
    protected BoundSql getDefaultBoundSql(Object parameterObject) {
        String tempSql = sql;
        String orderBy = PageHelper.getOrderBy();
        if (orderBy != null) {
            tempSql = OrderByParser.converToOrderBySql(sql, orderBy);
        }
        return new BoundSql(configuration, tempSql, parameterMappings, parameterObject);
    }

    @Override
    protected BoundSql getCountBoundSql(Object parameterObject) {
        // localParser指的就是MysqlParser或者OracleParser
        // localParser.get().getCountSql(sql)，可以根据原始的sql，生成一个count查询的sql
        return new BoundSql(configuration, localParser.get().getCountSql(sql), parameterMappings, parameterObject);
    }

    @Override
    protected BoundSql getPageBoundSql(Object parameterObject) {
        String tempSql = sql;
        String orderBy = PageHelper.getOrderBy();
        if (orderBy != null) {
            tempSql = OrderByParser.converToOrderBySql(sql, orderBy);
        }
        // getPageSql可以根据原始的sql，生成一个带有分页参数信息的sql，比如 limit ?, ?
        tempSql = localParser.get().getPageSql(tempSql);
        // 由于sql增加了分页参数的？号占位符，getPageParameterMapping()就是在原有List<ParameterMapping>基础上，增加两个分页参数对应的ParameterMapping对象，为分页参数赋值使用
        return new BoundSql(configuration, tempSql, localParser.get().getPageParameterMapping(configuration, original.getBoundSql(parameterObject)), parameterObject);
    }
}

```
getDefaultBoundSql：获取原始的未经改造的BoundSql。  
getCountBoundSql：不需要写count查询，插件根据分页查询sql，智能的为你生成的count查询BoundSql。  
getPageBoundSql：获取分页查询的BoundSql。

```
public class MysqlParser extends AbstractParser {
    @Override
    public String getPageSql(String sql) {
        StringBuilder sqlBuilder = new StringBuilder(sql.length() + 14);
        sqlBuilder.append(sql);
        sqlBuilder.append(" limit ?,?");
        return sqlBuilder.toString();
    }

    @Override
    public Map<String, Object> setPageParameter(MappedStatement ms, Object parameterObject, BoundSql boundSql, Page<?> page) {
        Map<String, Object> paramMap = super.setPageParameter(ms, parameterObject, boundSql, page);
        paramMap.put(PAGEPARAMETER_FIRST, page.getStartRow());
        paramMap.put(PAGEPARAMETER_SECOND, page.getPageSize());
        return paramMap;
    }
}

```
这里在查询语句后面拼接limit用于分页。  

```
// PageSqlSource装饰原SqlSource   
public void processMappedStatement(MappedStatement ms) throws Throwable {
        SqlSource sqlSource = ms.getSqlSource();
        MetaObject msObject = SystemMetaObject.forObject(ms);
        SqlSource pageSqlSource;
        if (sqlSource instanceof StaticSqlSource) {
            pageSqlSource = new PageStaticSqlSource((StaticSqlSource) sqlSource);
        } else if (sqlSource instanceof RawSqlSource) {
            pageSqlSource = new PageRawSqlSource((RawSqlSource) sqlSource);
        } else if (sqlSource instanceof ProviderSqlSource) {
            pageSqlSource = new PageProviderSqlSource((ProviderSqlSource) sqlSource);
        } else if (sqlSource instanceof DynamicSqlSource) {
            pageSqlSource = new PageDynamicSqlSource((DynamicSqlSource) sqlSource);
        } else {
            throw new RuntimeException("无法处理该类型[" + sqlSource.getClass() + "]的SqlSource");
        }
        msObject.setValue("sqlSource", pageSqlSource);
        //由于count查询需要修改返回值，因此这里要创建一个Count查询的MS
        msCountMap.put(ms.getId(), MSUtils.newCountMappedStatement(ms));
    }

// 执行分页查询
private Page doProcessPage(Invocation invocation, Page page, Object[] args) throws Throwable {
        //保存RowBounds状态
        RowBounds rowBounds = (RowBounds) args[2];
        //获取原始的ms
        MappedStatement ms = (MappedStatement) args[0];
        //判断并处理为PageSqlSource
        if (!isPageSqlSource(ms)) {
            processMappedStatement(ms);
        }
        //设置当前的parser，后面每次使用前都会set，ThreadLocal的值不会产生不良影响
        ((PageSqlSource)ms.getSqlSource()).setParser(parser);
        try {
            //忽略RowBounds-否则会进行Mybatis自带的内存分页
            args[2] = RowBounds.DEFAULT;
            //如果只进行排序 或 pageSizeZero的判断
            if (isQueryOnly(page)) {
                return doQueryOnly(page, invocation);
            }

            //简单的通过total的值来判断是否进行count查询
            if (page.isCount()) {
                page.setCountSignal(Boolean.TRUE);
                //替换MS
                args[0] = msCountMap.get(ms.getId());
                //查询总数
                Object result = invocation.proceed();
                //还原ms
                args[0] = ms;
                //设置总数
                page.setTotal((Integer) ((List) result).get(0));
                if (page.getTotal() == 0) {
                    return page;
                }
            } else {
                page.setTotal(-1l);
            }
            //pageSize>0的时候执行分页查询，pageSize<=0的时候不执行相当于可能只返回了一个count
            if (page.getPageSize() > 0 &&
                    ((rowBounds == RowBounds.DEFAULT && page.getPageNum() > 0)
                            || rowBounds != RowBounds.DEFAULT)) {
                //将参数中的MappedStatement替换为新的qs
                page.setCountSignal(null);
                BoundSql boundSql = ms.getBoundSql(args[1]);
                args[1] = parser.setPageParameter(ms, args[1], boundSql, page);
                page.setCountSignal(Boolean.FALSE);
                //执行分页查询
                Object result = invocation.proceed();
                //得到处理结果
                page.addAll((List) result);
            }
        } finally {
            ((PageSqlSource)ms.getSqlSource()).removeParser();
        }

        //返回结果
        return page;
    }

```
msCountMap.put(ms.getId(), MSUtils.newCountMappedStatement(ms))，创建count查询的MappedStatement对象，并缓存于msCountMap。如果count=true，则执行count查询，结果total值保存于page对象中，继续执行分页查询。执行分页查询，将查询结果保存于page对象中，page是一个ArrayList对象。args[2] = RowBounds.DEFAULT，改变Mybatis原有分页行为；args[1] = parser.setPageParameter(ms, args[1], boundSql, page)，改变原有参数列表（增加分页参数）。

&#8194;总结：PageHelper会使用ThreadLocal获取到同一线程中的变量信息，各个线程之间的Threadlocal不会相互干扰，也就是Thread1中的ThreadLocal1之后获取到Tread1中的变量的信息，不会获取到Thread2中的信息
所以在多线程环境下，各个Threadlocal之间相互隔离，可以实现，不同thread使用不同的数据源或不同的Thread中执行不同的SQL语句。所以，PageHelper利用这一点通过拦截器获取到同一线程中的预编译好的SQL语句之后将SQL语句包装成具有分页功能的SQL语句，并将其再次赋值给下一步操作，所以实际执行的SQL语句就是有了分页功能的SQL语句。
