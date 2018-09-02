---
title: Spring Boot（十一）：关系型数据库之jdbcTemplate
date: 2018-09-02 10:14:54
tags:
 - SpringBoot
categories: 
 - SpringBoot
---

### 介绍

pom文件引入：

~~~java
<!-- jdbc -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jdbc</artifactId>
</dependency>
<!-- mysql -->
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
</dependency>
~~~

1. 可以不指定 driver-class-name， spring boot 会自动识别 url 
2. 数据连接池默认使用 tomcat-jdbc 
3. 连接池的配置： spring.datasource.tomcat.*  

<!-- more -->

实体类：

~~~java
public class SbUser {
    private int id;
    private String name;
    private Date createTime;
    // 省略getter/setter
}
~~~

接口：

~~~java
public interface SbUserDao {
    int insert(SbUser sbUser);

    int deleteById(int id);

    int updateById(SbUser sbUser);

    SbUser selectById(int id);

    Page<SbUser> queryForPage(int pageCurrent, int pageSize, String name);
}
~~~

实现类：

~~~java
@Autowired
    private JdbcTemplate jdbcTemplate;

    @Override
    public int insert(SbUser sbUser) {
        String sql = "insert into sb_user (name, create_time) values (?, ?)";
        return jdbcTemplate.update(sql, sbUser.getName(),
                sbUser.getCreateTime());
    }

    @Override
    public int deleteById(int id) {
        String sql = "delete from sb_user where id=?";
        return jdbcTemplate.update(sql, id);
    }

    @Override
    public int updateById(SbUser sbUser) {
        String sql = "update sb_user set name=?, create_time=? where id=?";
        return jdbcTemplate.update(sql, sbUser.getName(),
                sbUser.getCreateTime(), sbUser.getId());
    }

    @Override
    public SbUser selectById(int id) {
        String sql = "select * from sb_user where id=?";
        return jdbcTemplate.queryForObject(sql, new RowMapper<SbUser>() {
            @Override
            public SbUser mapRow(ResultSet rs, int rowNum) throws SQLException {
                SbUser sbUser = new SbUser();
                sbUser.setId(rs.getInt("id"));
                sbUser.setName(rs.getString("name"));
                sbUser.setCreateTime(rs.getDate("create_time"));
                return sbUser;
            }
        }, id);
    }

    @Override
    public Page<SbUser> queryForPage(int pageCurrent, int pageSize, String name) {
        // 若要like查询，如下
        StringBuffer sql = new StringBuffer("select * from sb_user where 1");
        if(name != null){
            // Sql.checkSql 的作用是防止sql注入
            sql.append(" and name like '%").append(Sql.checkSql(name)).append("%' ");
        }
        return queryForPage(sql.toString(), pageCurrent, pageSize, SbUser.class);
    }
~~~

分页：

~~~java
/**
 * 分页， jdbcTemplate 不支持 like 是定义，只能拼装
 */
public <T> Page<T> queryForPage(String sql, int pageCurrent, int pageSize, Class<T>
        clazz, Object... args) {
    Assert.hasText(sql, "sql 语句不能为空");
    Assert.isTrue(pageCurrent >= 1, "pageNo 必须大于等于 1");
    Assert.isTrue(clazz != null, "clazz 不能为空");
    String sqlCount = Sql.countSql(sql);
    int count = jdbcTemplate.queryForObject(sqlCount, Integer.class, args);
    pageCurrent = Sql.checkPageCurrent(count, pageSize, pageCurrent);
    pageSize = Sql.checkPageSize(pageSize);
    int totalPage = Sql.countTotalPage(count, pageSize);
    String sqlList = sql + Sql.limitSql(count, pageCurrent, pageSize);
    List<T> list = jdbcTemplate.query(sqlList, new BeanPropertyRowMapper<T>(clazz),
            args);
    return new Page<T>(count, totalPage, pageCurrent, pageSize, list);
}
~~~

分页工具类：

~~~java
public class Page<T> {
    private static final long serialVersionUID = -5764853545343945831L;

    /**
     * 默认每页记录数(20)
     */
    public static final int DEFAULT_PAGE_SIZE = 20;

    /**
     * 最大每页记录数(1000)
     */
    public static final int MAX_PAGE_SIZE = 1000;

    /**
     * 当前分页的数据集
     */
    private List list;

    /**
     * 总记录数
     */
    private int totalCount;

    /**
     * 总页数
     */
    private int totalPage;

    /**
     * 当前页
     */
    private int pageCurrent;

    /**
     * 每页记录数
     */
    private int pageSize;

    /**
     * 排序字段
     */
    private String orderField;

    /**
     * 排序方式：asc or desc
     */
    private String orderDirection;
    
    /**
     * 构造函数
     *
     * @param totalCount
     *            总记录数
     * @param totalPage
     *            总页数
     * @param pageCurrent
     * @param pageSize
     * @param list
     */
    public Page(int totalCount, int totalPage, int pageCurrent, int pageSize, List<T> list) {
        this.totalCount = totalCount;
        this.totalPage = totalPage;
        this.pageCurrent = pageCurrent;
        this.pageSize = pageSize;
        this.list = list;
    }
    // 省略getter/setter
}
~~~

~~~java
public class Sql {
    private Sql() {
    }

    /**
     * 检测sql，防止sql注入
     * @param sql
     * @return 正常返回sql；异常返回""
     */
    public static String checkSql(String sql) {
        String inj_str = "'|and|exec|insert|select|delete|update|count|*|%|chr|mid|master|truncate|char|declare|;|or|-|+|,";
        String inj_stra[] = inj_str.split("\\|");
        for (int i = 0; i < inj_stra.length; i++) {
            if (sql.indexOf(inj_stra[i]) >= 0) {
                return "";
            }
        }
        return sql;
    }

    /**
     * 计算总页数
     * @param totalCount 总记录数.
     * @param pageSize 每页记录数.
     * @return totalPage 总页数.
     */
    public static int countTotalPage(final int totalCount, final int pageSize) {
        if (totalCount % pageSize == 0) {
            return totalCount / pageSize; // 刚好整除
        } else {
            return totalCount / pageSize + 1; // 不能整除则总页数为：商 + 1
        }
    }

    /**
     * 校验当前页数pageCurrent<br/>
     * 1、先根据总记录数totalCount和每页记录数pageSize，计算出总页数totalPage<br/>
     * 2、判断页面提交过来的当前页数pageCurrent是否大于总页数totalPage，大于则返回totalPage<br/>
     * 3、判断pageCurrent是否小于1，小于则返回1<br/>
     * 4、其它则直接返回pageCurrent
     *
     * @param totalCount 要分页的总记录数
     * @param pageSize 每页记录数大小
     * @param pageCurrent 输入的当前页数
     * @return pageCurrent
     */
    public static int checkPageCurrent(int totalCount, int pageSize, int pageCurrent) {
        int totalPage = countTotalPage(totalCount, pageSize); // 最大页数
        if (pageCurrent > totalPage) {
            // 如果页面提交过来的页数大于总页数，则将当前页设为总页数
            // 此时要求totalPage要大于获等于1
            if (totalPage < 1) {
                return 1;
            }
            return totalPage;
        } else if (pageCurrent < 1) {
            return 1; // 当前页不能小于1（避免页面输入不正确值）
        } else {
            return pageCurrent;
        }
    }

    /**
     * 校验页面输入的每页记录数pageSize是否合法<br/>
     * 1、当页面输入的每页记录数pageSize大于允许的最大每页记录数MAX_PAGE_SIZE时，返回MAX_PAGE_SIZE
     * 2、如果pageSize小于1，则返回默认的每页记录数DEFAULT_PAGE_SIZE
     * @param pageSize 页面输入的每页记录数
     * @return checkPageSize
     */
    public static int checkPageSize(int pageSize) {
        if (pageSize > Page.MAX_PAGE_SIZE) {
            return Page.MAX_PAGE_SIZE;
        } else if (pageSize < 1) {
            return Page.DEFAULT_PAGE_SIZE;
        } else {
            return pageSize;
        }
    }

    /**
     * 计算当前分页的开始记录的索引
     * @param pageCurrent 当前第几页
     * @param pageSize 每页记录数
     * @return 当前页开始记录号
     */
    public static int countOffset(final int pageCurrent, final int pageSize) {
        return (pageCurrent - 1) * pageSize;
    }

    /**
     * 根据总记录数，对页面传来的分页参数进行校验，并返分页的SQL语句
     * @param pageCurrent 当前页
     * @param pageSize 每页记录数
     * @param totalCount DWZ分页查询参数
     * @return limitSql
     */
    public static String limitSql(int totalCount, int pageCurrent, int pageSize) {
        // 校验当前页数
        pageCurrent = checkPageCurrent(totalCount, pageSize, pageCurrent);
        pageSize = checkPageSize(pageSize); // 校验每页记录数
        return " limit " + countOffset(pageCurrent, pageSize) + "," + pageSize;
    }

    /**
     * 根据分页查询的SQL语句，获取统计总记录数的语句
     * @param sql 分页查询的SQL
     * @return countSql
     */
    public static String countSql(String sql) {
        // 去除第一个from前的内容
        String countSql = sql.substring(sql.toLowerCase().indexOf("from")); 
        return "select count(*) " + removeOrderBy(countSql);
    }

    /**
     * 移除SQL语句中的的order by子句（用于分页前获取总记录数，不需要排序）
     * @param sql 原始SQL
     * @return 去除order by子句后的内容
     */
    private static String removeOrderBy(String sql) {
        Pattern pat = Pattern.compile("order\\s*by[\\w|\\W|\\s|\\S]*", Pattern.CASE_INSENSITIVE);
        Matcher mc = pat.matcher(sql);
        StringBuffer strBuf = new StringBuffer();
        while (mc.find()) {
            mc.appendReplacement(strBuf, "");
        }
        mc.appendTail(strBuf);
        return strBuf.toString();
    }
}
~~~

测试类：

~~~java
@Autowired
private SbUserDao sbUserDao;
@Test
public void insert() {
    SbUser sbUser = new SbUser();
    sbUser.setName("测试");
    sbUser.setCreateTime(new Date());
    int result = sbUserDao.insert(sbUser);
    System.out.println(result);
}
@Test
public void delete() {
    int result = sbUserDao.deleteById(1);
    System.out.println(result);
}
@Test
public void update() {
    SbUser sbUser = new SbUser();
    sbUser.setId(2);
    sbUser.setName("测试 2");
    sbUser.setCreateTime(new Date());
    int result = sbUserDao.updateById(sbUser);
    System.out.println(result);
}
@Test
public void select() {
    SbUser result = sbUserDao.selectById(2);
    System.out.println(result);
}

// 分页测试
@Test
public void queryForPage(){
    Page<SbUser> result = sbUserDao.queryForPage(1, 20, "测试");
    System.out.println(result.getList());
}
~~~





### 