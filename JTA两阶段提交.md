# <center>分布式事物之二段提交</center>
## 说明
> 两阶段提交协议（The two-phase commit protocol，2PC）是XA用于在全局事务中协调多个资源的机制。两阶段协议遵循OSI（Open System Interconnection，开放系统互联）/DTP标准，虽然它比标准本身早若干年出现。两阶段提交协议包含了两个阶段：第一阶段（也称准备阶段）和第二阶段（也称提交阶段）。一个描述两阶段提交很好的类比是典型的结婚仪式，每个参与者（结婚典礼中的新郎和新娘）都必须服从安排，在正式步入婚姻生活之前说“我愿意”。考虑有的杯具情形，“参与者”之一在做出承诺前的最后一刻反悔。两阶段提交之于此的结果也成立，虽然不具备那么大的破坏性。<br/><br/>
当commit()请求从客户端向事务管理器发出，事务管理器开始两阶段提交过程。在第一阶段，所有的资源被轮询到，问它们是否准备好了提交作业。每个参与者可能回答“准备好（READY）”，“只读（READ_ONLY）”，或“未准备好（NOT_READY）”。如果有任意一个参与者在第一阶段响应“未准备好（NOT_READY）”，则整个事务回滚。如果所有参与者都回答“准备好（READY）”，那这些资源就在第二阶段提交。回答“只读（READ_ONLY）”的资源，则在协议的第二阶段处理中被排除掉。

## springboot使用XA/JTA两阶段提交
在pom文件中添加如下配置
```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jta-atomikos</artifactId>
</dependency>
```
测试代码

```
package com.example.jtademo;

import com.atomikos.icatch.jta.UserTransactionImp;
import org.springframework.boot.jta.atomikos.AtomikosDataSourceBean;

import javax.transaction.SystemException;
import javax.transaction.UserTransaction;
import java.sql.Connection;
import java.sql.PreparedStatement;
import java.util.Properties;

/**
 * @Auther: xuepengbo
 * @Date: 2018/8/4 16:23
 * @Description:
 */
public class demo {

    private static AtomikosDataSourceBean createAtomikosDataSourceBean(String dbName) {
        // 连接池基本属性
        Properties p = new Properties();
        p.setProperty("url", "jdbc:mysql://localhost:3306/" + dbName);
        p.setProperty("user", "root");
        p.setProperty("password", "123456");

        // 使用AtomikosDataSourceBean封装com.mysql.jdbc.jdbc2.optional.MysqlXADataSource
        AtomikosDataSourceBean ds = new AtomikosDataSourceBean();
        //atomikos要求为每个AtomikosDataSourceBean名称，为了方便记忆，这里设置为和dbName相同
        ds.setUniqueResourceName(dbName);
        ds.setXaDataSourceClassName("com.mysql.jdbc.jdbc2.optional.MysqlXADataSource");
        ds.setXaProperties(p);
        return ds;
    }

    public static void main(String[] args) {

        AtomikosDataSourceBean ds1 = createAtomikosDataSourceBean("test");
        AtomikosDataSourceBean ds2 = createAtomikosDataSourceBean("test1");

        Connection conn1 = null;
        Connection conn2 = null;
        PreparedStatement ps1 = null;
        PreparedStatement ps2 = null;

        UserTransaction userTransaction = new UserTransactionImp();
        try {
            // 开启事务
            userTransaction.begin();

            // 执行db1上的sql
            conn1 = ds1.getConnection();
            ps1 = conn1.prepareStatement("INSERT into user(name) VALUES (?)");
            ps1.setString(1, "tianshouzhi");
            ps1.executeUpdate();

            // 模拟异常 ，直接进入catch代码块，2个都不会提交
            // int i = 1/0;

            // 执行db2上的sql
            conn2 = ds2.getConnection();
            ps2 = conn2.prepareStatement("INSERT into user(name) VALUES (?)");
            ps2.setString(1, "xuepengbo");
            ps2.executeUpdate();

            // 两阶段提交
            userTransaction.commit();
        } catch (Exception e) {
            e.printStackTrace();
            try {
                userTransaction.rollback();
            } catch (SystemException e1) {
                e1.printStackTrace();
            }
        } finally {
            try {
                ps1.close();
                ps2.close();
                conn1.close();
                conn2.close();
                ds1.close();
                ds2.close();
            } catch (Exception ignore) {
            }
        }
    }
}

```
