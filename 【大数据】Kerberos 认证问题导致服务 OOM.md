三年前来到公司大数据团队，算是入了大数据的坑。一开始对大数据的组件不是很了解，一路走来在不停地学习探索。上周遇到了一个问题，我们数据地图的服务在预发环境触发 POD 级别的 OOM ，新启了个 pod；由于上了个大需求到预发环境所以比平常更关注了一下，
简单排查发现：
- POD 内存飚到了 93% 
- 线程数飙到了近 700

`大家可以思考下为啥我没去看 Java 堆内存？结尾给答案。`
正常情况下线程数最多 250 （web 线程以及一些杂七杂八的线程） 左右，飙到近 700 肯定是有问题的，由于预发环境已经重启，当时立马想的线上会不会也出现同样的问题，抱着侥幸的心态去看了下巧了也飙起来直逼 800 。直接登录到 POD 容器上用 `jstack` 命令把线程栈导出到文件里，元凶立马浮出水面：
![部分异常线程数](https://upload-images.jianshu.io/upload_images/11859806-58b77b778ff7b7bf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
熟悉大数据的同学可能已经看出来了这就是 Kerberos 续期 TGT(票据授予票据) 的线程，可是为什么会这么多呢？我打上马赛克的位置是用户名而且都是同一个，查了资料发现是在调用 `UserGroupInformation.loginUserFromKeytab` 时，在开启自动续期 TGT 的情况就会创建一个 daemon 线程去续期，代码如下：
```
@InterfaceAudience.Public
  @InterfaceStability.Evolving
  public
  static void loginUserFromKeytab(String user,
                                  String path
                                  ) throws IOException {
    if (!isSecurityEnabled())
      return;

    UserGroupInformation u = loginUserFromKeytabAndReturnUGI(user, path);
    if (isKerberosKeyTabLoginRenewalEnabled()) {
      u.spawnAutoRenewalThreadForKeytab();
    }

    setLoginUser(u);

    LOG.info(
        "Login successful for user {} using keytab file {}. Keytab auto"
            + " renewal enabled : {}",
        user, new File(path).getName(), isKerberosKeyTabLoginRenewalEnabled());
  }

private void spawnAutoRenewalThreadForKeytab() {
    if (!shouldRelogin() || isFromTicket()) {
      return;
    }

    // spawn thread only if we have kerb credentials
    KerberosTicket tgt = getTGT();
    if (tgt == null) {
      return;
    }
    long nextRefresh = getRefreshTime(tgt);
    executeAutoRenewalTask(getUserName(),
            new KeytabRenewalRunnable(tgt, nextRefresh));
  }

  /**
   * Spawn a thread to do periodic renewals of kerberos credentials from a
   * keytab file. NEVER directly call this method.
   *
   * @param userName Name of the user for which login needs to be renewed.
   * @param task  The reference of the login renewal task.
   */
  private void executeAutoRenewalTask(final String userName,
                                      AutoRenewalForUserCredsRunnable task) {
    kerberosLoginRenewalExecutor = Optional.of(
            Executors.newSingleThreadExecutor(
                  new ThreadFactory() {
                    @Override
                    public Thread newThread(Runnable r) {
                      Thread t = new Thread(r);
                      t.setDaemon(true);
                      t.setName("TGT Renewer for " + userName);
                      return t;
                    }
                  }
            ));
    kerberosLoginRenewalExecutor.get().submit(task);
  }
```
因为上面出现了很多个对同一个账号续期的线程，也就是程序里可能有个地方会多次对同一个账号进行登录，查询代码还真发现了一处：
```
private <R> R doActionWithRetry(Action<R> action) throws TException {
        try {
            synchronized (client) {
                return action.call();
            }
        } catch (TException e) {
            try {
                synchronized (client) {
                    kerberosClient.login();
                    client.reconnect();
                }
            } catch (MetaException | IOException unHandleException) {
                throw e;
            }
            synchronized (client) {
                return action.call();
            }
        }
    }
```
这是内部封装的一个支持重试的方法，在 catch 住 `TException` 时会调用 `kerberosClient.login()` ，而 `kerberosClient#login` 的方法里就是调用  `UserGroupInformation.loginUserFromKeytab` , 这里暂时不讨论为啥会抛出 `TException` ，因为情况是多种多样的，直接解决可以用下面这个方法代替：
```
private void reLoginExpiringKeytabUser() throws MetaException {
        if (!UserGroupInformation.isSecurityEnabled()) {
            return;
        }
        try {
         
            //获取当前UGI 实例（例如：账号:data.map）
            UserGroupInformation ugi = UserGroupInformation.getLoginUser();
            if (ugi.isFromKeytab()) {
                //这方法会判断当前的 TGT 是否过期，如果过期会重新登录，不会启动一个新的线程
                ugi.checkTGTAndReloginFromKeytab();
            }
        } catch (IOException e) {
            String msg = "Error doing relogin using keytab " + e.getMessage();
            throw new MetaException(msg);
        }
    }
``` 
现在还有一个问题就是已经配置了自动续期为啥还会抛这个错呢？下期再说

## 结语
我们来回答下开篇的问题：为啥没去看 Java 堆内存? 因为这是 pod 级别的 OOM ，而我们的数据地图服务在启动时就给了最大内存，不会变的，只能是其他部分占用内存不正常才导致 POD OOM 的，这算是一个小小的思维点，祝大家生活愉快。
