# 自动化场地预约脚本的实现（demo版本）

## 问题背景

华科的场馆预约网站极其简陋，有很多人曾经利用脚本来抢场地，既然该问题具有可实现性，就可以开始准备了。写此次脚本的过程中涉及到网络抓包，html，http和python中的request库的知识，在掌握基本的知识之后就可以开始行动了。

## 问题简化

为了简化整个过程，提炼出以下几步：

1. 完成统一验证账户的登录
2. 进入到场馆选择界面，先点击提示界面中的我同意按钮，选择西体
3. 将日期切换到两天后，选择好晚上8-10的时间，选择好同伴和所需场地，点击预约
4. 跳转到step2选择统一支付，滑动验证图形码，点击预约
5. 跳转到付款界面，选择微信付款扫描二维码，完成预约场地

此处进一步简化了问题的模型，因为有学长完成了华科统一验证系统的的登录过程，该过程采用RSA加密，破解掉该过程，再利用机器学习模型完成激活码的识别，就可以完成任何华科登录网站的自动登录，此处我直接调用了学长写的包，省去了这一步骤

## 网络抓包

梳理了整个预约场地动作的流程之后就可以开始抓包

### 第一步

![image-20240821104616129](C:\Users\zy202\AppData\Roaming\Typora\typora-user-images\image-20240821104616129.png)

这是进入到选择场地的界面过程中抓到的所有流量包，可以先看到这么多流量包中包括了document文件，网页中的js和css文件以及该页面具有的一些jpg和png图片，对于我们的抓包而言我们只需要获取到html的内容，所以只需要关注document文件即可。

![image-20240821104857774](C:\Users\zy202\AppData\Roaming\Typora\typora-user-images\image-20240821104857774.png)

这是第一个文档文件的预览

仔细阅读html代码可以发现网页中有一个token值和cg_csrf_token值，但是还不知道有何用处，其余代码并无特殊之处，我们分析完进行下一步抓包

![image-20240821105149972](C:\Users\zy202\AppData\Roaming\Typora\typora-user-images\image-20240821105149972.png)

![image-20240821105141892](C:\Users\zy202\AppData\Roaming\Typora\typora-user-images\image-20240821105141892.png)

### 第二步

![image-20240821105321322](C:\Users\zy202\AppData\Roaming\Typora\typora-user-images\image-20240821105321322.png)

这是第二步点击西体场所后的抓包信息，可以看到页面中只有一个document文件，但是最后有一个类型为xhr文件，查阅学习之后发现这是一个ajax请求的文件，我们后续处理。

先点击第一个syqk?cdbh=69文件，这个是进入西体界面发起的请求，cdbh就是场地编号，69是西体的号码

![image-20240821105545447](C:\Users\zy202\AppData\Roaming\Typora\typora-user-images\image-20240821105545447.png)

![image-20240821105615077](C:\Users\zy202\AppData\Roaming\Typora\typora-user-images\image-20240821105615077.png)

观察表头发现这是一个get文件，继续观察负载中有一个cdbh:69的参数，分析页面代码可以看到此页面仍有token和cg_csrf_token，但是token的数值发生了变化，csrftoken没有变化，查阅资料发现csrf_token属于安全令牌，确保网页的安全性，在一定时间内访问该网站留下的cookie值中会保持不变，直到会话关闭或者cookie过期才会变化。

![image-20240821105931134](C:\Users\zy202\AppData\Roaming\Typora\typora-user-images\image-20240821105931134.png)

此页面中发现一个函数，里面写着场地是否可以预约的判断方式。其余信息用处不大。

![image-20240821110033378](C:\Users\zy202\AppData\Roaming\Typora\typora-user-images\image-20240821110033378.png)

![image-20240821110106066](C:\Users\zy202\AppData\Roaming\Typora\typora-user-images\image-20240821110106066.png)



最后分析这个ajax请求，发起模式为post，在headers中唯一特殊的一处就是最后的X-Requested-With一行，这个专门用于ajax请求中。ajax代表着异步请求，说明页面信息可以不按照流程一步一步请求，而是异步请求加快了获取页面信息的速度，下面观察这个请求的更多信息。

![image-20240821110249814](C:\Users\zy202\AppData\Roaming\Typora\typora-user-images\image-20240821110249814.png)

![image-20240821110301944](C:\Users\zy202\AppData\Roaming\Typora\typora-user-images\image-20240821110301944.png)

发现这个请求的表单数据中有changdibh（场地编号）,data,date,time,token，返回值中返回了时间段，场地的片编号和zt值，而zt值代表场地是否可以预约，所以我们可以分析出这个异步请求是用来请求我们可能访问的时间段内所有的场地预约信息。那么data里面的数字584代表的是片编号，@后面接的是时间，date是我要预约的时间，time是现在发起请求的时间，因为每时每刻场地信息都可能发生变化，所以需要当前时间的信息，这也是为什么需要发起异步请求的理由，我们需要足够快地加载出场地的信息，否则我们点开西体场地的界面的时候无法看到哪些场地是可以预约的。最后发现该请求的返回信息中仍有一个token数值，这个和前面的token数值不一样，存疑继续分析。

### 第三步

选择好日期，同伴和场地片编号之后就可以进行下一步，点击我要预约，抓包到响应信息

![image-20240821111734946](C:\Users\zy202\AppData\Roaming\Typora\typora-user-images\image-20240821111734946.png)

![image-20240821111741329](C:\Users\zy202\AppData\Roaming\Typora\typora-user-images\image-20240821111741329.png)

这个post请求的表单数据中有很多信息，其中`starttime,endtime`是我选择预约的时间，下面几个parterner的信息是同伴的信息，type是1是固定的，然后是姓名，学号和密码，choosetime是选择的片编号，changdibh是西体的编号，date是我要预约的日期，cg_csrf_token是安全令牌，还有token。这个地方就迎来了脚本的第一个大难题，token和csrf的数值从哪里获取，在反复观察已有页面发现csrf的值基本是固定的，但是过了很长一段时间后发生变化，可以推测这个数据是存储在cookie里面的，我们脚本的运行时间短，不需要考虑，而token的数值出现了很多遍，网站编写得很烂，变量命名不是很完善，所以这个地方不好找。分析前面第一步和第二步中所有的网页，发现跳转一次token数值就会变化一次。但是ajax请求是一个异步请求，这里的token值发现和发起这个异步请求时候界面的token值一致，可以推断出前面界面中token值没有用。但是我们发现ajax请求中也有一个token值，这个数值和最后step2的数值一致，我们就可以理解这个token数值的意义和作用了。网站为了追求时效性，涉及了异步请求获得场地的信息，然后用户根据这里的信息及时判断选择哪一片场地，带着异步请求的token数值进入到下一步，即锁定场地进行付款，这个时间间隔是很短的，在抢场地这种耗时很短的活动中可用性很强。



### 第四步

下一步就是选择付款方式发起付款，因为电子账户充值复杂，我们使用统一账户二维码付款。

![image-20240821112912027](C:\Users\zy202\AppData\Roaming\Typora\typora-user-images\image-20240821112912027.png)

![image-20240821112917551](C:\Users\zy202\AppData\Roaming\Typora\typora-user-images\image-20240821112917551.png)

![image-20240821112936711](C:\Users\zy202\AppData\Roaming\Typora\typora-user-images\image-20240821112936711.png)

我们看到点击付款之后发起了post请求，但是状态码是302重定位状态码，说明服务器给了重定向的指令，根据响应链可以看到重定位的界面应该就是付款界面，我们只需要顺利完成请求就可以得到这个url。那么阅读表单数据，后面两个token和前面step2的token值一样已经获取了。只剩前面三个，多次请求发现-2代表二维码付款，-1代表电子账户支付，那么只需要找到前面两个，这个步骤从step2来，大概率在step2里面

![image-20240821114017390](C:\Users\zy202\AppData\Roaming\Typora\typora-user-images\image-20240821114017390.png)

阅读html代码找到了对于的信息，那么所有的抓包工作就完成了，在程序中输入最后的付款url点击就可以付款了。

## 代码实现

完成了抓包之后代码其实难度不大，我写出代码的一些难处和细节

1. headers的user-agent选择edge，因为不开代理访问速度更快，而不开代理

2. 每次请求的页面是固定的，不需要去网页中找href

3. ajax异步请求的headers加上那个`'x-requested-with':'XMLHttpRequest'`

4. headers中的host和refer好像有的时候有用

5. 设置时间限制，没到8点先完成登录但是不开始下一步，到了时间开始，每隔0.2秒一次，上限200次

   