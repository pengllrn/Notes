## Spring(三)：Aop

### 数据库事务分析

#### 转账

``` java
public void transfer(String sourceName, String targetName, Float money) {
    //1.获得账户
    Account sorceAccount = iaccountDao.findAccountByName(sourceName);
    Account targetAccount = iaccountDao.findAccountByName(targetName);
    //2.设置账户
    sorceAccount.setMoney(sorceAccount.getMoney() - money);
    targetAccount.setMoney(targetAccount.getMoney() + money);
    //3.更新账户
    iaccountDao.updateAccount(sorceAccount);
    iaccountDao.updateAccount(targetAccount);
}
```

在步骤3更新账户过程中如果出现了错误，例如sourceAccount更新成功，而targetAccount没有更新成功，则会导致一个账户金额减少，而另一个账户金额不变。这不符合事务的一致性原则。

``` java
//3.更新账户
    iaccountDao.updateAccount(sorceAccount);
	int i = 1 /0;//抛出异常
    iaccountDao.updateAccount(targetAccount);
```

之所以出现这个问题的是因为每次获得账户以及更新账户都会提交一次事务，相当于启用了多个事务，互不干扰。

因此考虑使用事务控制，让更新完账户之后进行一次事务的提交，若在事务提交之前出现了异常，则启动事务启动回滚，放弃之前的所有更新操作。最后的事务提交类似于保存操作。

#### 方案

使用ThreadLocal<T>对象把Connection和当前线程绑定，从而使一个线程中只能获得一个控制事务的对象。

ConnectionUtils.java

``` java
/**
 * 连接工具类，它从数据源中获取到一个连接，并且和当前线程绑定
 */
@Component
public class ConnectionUtils {
    ThreadLocal<Connection> threadLocal = new ThreadLocal<Connection>();

    @Autowired
    private DataSource dataSource;

    /**
     * 获取当前线程上的连接
     * @return
     */
    public Connection getConnection(){
        try {
            //从ThreadLocal上获取
            Connection conn = threadLocal.get();
            if(conn == null){
                //如果没有从数据源获取
                conn = dataSource.getConnection();
                //存放在ThreadLocal中
                threadLocal.set(conn);
            }
            return conn;
        }catch (Exception e){
            throw new RuntimeException(e);
        }
    }

    /**
     * 把连接和线程解绑
     */
    public void removeConnection(){
        threadLocal.remove();
    }
}
```

TansactionManager.java

``` java
/**
 * 和事务相关的管理器，作用为开启事务，提交事务，回滚事务，是否连接
 */
@Component
public class TransactionManager {
    @Autowired
    private ConnectionUtils connectionUtils;

    /**
     * 开启事务
     */
    public void beginTransaction(){
        try {
            connectionUtils.getConnection().setAutoCommit(false);//不要自动提交
        }catch (Exception e){
            e.printStackTrace();
        }
    }

    /**
     * 提交事务
     */
    public void commit(){
        try {
            connectionUtils.getConnection().commit();
        }catch (Exception e){
            e.printStackTrace();
        }
    }

    /**
     * 回滚事务
     */
    public void rollback(){
        try {
            connectionUtils.getConnection().rollback();
        }catch (Exception e){
            e.printStackTrace();
        }
    }

    /**
     * 释放连接
     */
    public void release(){
        try {
            connectionUtils.getConnection().close();//连接还回连接池
            connectionUtils.removeConnection();
        }catch (Exception e){
            e.printStackTrace();
        }
    }
}
```

Dao实现类使用ConnectionUtils，QueryRunner的第一个参数添加一个Connection。

``` java
@Repository("accountDao2")
public class AccountDaoImpl2 implements IAccountDao {
    @Autowired
    private QueryRunner queryRunner;

    @Autowired
    private ConnectionUtils connectionUtils;

    public List<Account> findAllAccount() {
        try {
            return queryRunner.query(connectionUtils.getConnection(),
                    "select * from account",
                    new BeanListHandler<Account>(Account.class));
        } catch (SQLException e) {
            throw new RuntimeException(e);
        }
    }
 //.....   
}
```

然后数据库的操作变成：

``` java
/**
 * 账户的业务层实现类
 */
@Service("accountService3")
public class AccountServiceImpl3 implements IAccountService {

    @Autowired
    @Qualifier(value = "accountDao2")
    private IAccountDao iaccountDao;

    @Autowired
    private TransactionManager transactionManager;

    public List<Account> findAllAccount() {
        try {
            //1.开启事务
            transactionManager.beginTransaction();
            //2.执行操作
            List<Account> accounts = iaccountDao.findAllAccount();
            //3.提交事务
            transactionManager.commit();
            //4.返回结果
            return accounts;
        }catch (Exception e){
            //5.异常 事务回滚
            transactionManager.rollback();
            throw new RuntimeException(e);
        }finally {
            //6.释放连接
            transactionManager.release();
        }
    }

    public Account findAccountById(Integer id) {
        try {
            //1.开启事务
            transactionManager.beginTransaction();
            //2.执行操作
            Account account = iaccountDao.findAccountById(id);
            //3.提交事务
            transactionManager.commit();
            //4.返回结果
            return account;
        }catch (Exception e){
            //5.异常 事务回滚
            transactionManager.rollback();
            throw new RuntimeException(e);
        }finally {
            //6.释放连接
            transactionManager.release();
        }
    }

    public void addAccount(Account account) {
        try {
            //1.开启事务
            transactionManager.beginTransaction();
            //2.执行操作
            iaccountDao.addAccount(account);
            //3.提交事务
            transactionManager.commit();
            //4.返回结果
        }catch (Exception e){
            //5.异常 事务回滚
            transactionManager.rollback();
            throw new RuntimeException(e);
        }finally {
            //6.释放连接
            transactionManager.release();
        }
    }

    public void updateAccount(Account account) {
        try {
            //1.开启事务
            transactionManager.beginTransaction();
            //2.执行操作
            iaccountDao.updateAccount(account);
            //3.提交事务
            transactionManager.commit();
            //4.返回结果
        }catch (Exception e){
            //5.异常 事务回滚
            transactionManager.rollback();
            throw new RuntimeException(e);
        }finally {
            //6.释放连接
            transactionManager.release();
        }

    }

    public void deleteAccount(Integer id) {
        try {
            //1.开启事务
            transactionManager.beginTransaction();
            //2.执行操作
            iaccountDao.deleteAccount(id);
            //3.提交事务
            transactionManager.commit();
            //4.返回结果
        }catch (Exception e){
            //5.异常 事务回滚
            transactionManager.rollback();
            throw new RuntimeException(e);
        }finally {
            //6.释放连接
            transactionManager.release();
        }

    }

    public void transfer(String sourceName, String targetName, Float money) {
        try {
            //1.开启事务
            transactionManager.beginTransaction();
            //2.执行操作
            //2.1.获得账户
            Account sorceAccount = iaccountDao.findAccountByName(sourceName);
            Account targetAccount = iaccountDao.findAccountByName(targetName);
            //2.2.设置账户
            sorceAccount.setMoney(sorceAccount.getMoney() - money);
            targetAccount.setMoney(targetAccount.getMoney() + money);
            //2.3.更新账户
            iaccountDao.updateAccount(sorceAccount);
            int i = 1/0;
            iaccountDao.updateAccount(targetAccount);
            //3.提交事务
            transactionManager.commit();
            //4.返回结果
        }catch (Exception e){
            //5.异常 事务回滚
            transactionManager.rollback();
            throw new RuntimeException(e);
        }finally {
            //6.释放连接
            transactionManager.release();
        }
    }
}
```

这样就是完整的事务控制了。









