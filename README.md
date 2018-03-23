# Spider

标签（空格分隔）： spiser http javascript nodejs python scrapy

---

### 什么是爬虫
说到爬虫可能有比很多人会觉得，它很神秘，技术很难掌握。如果说是比较复杂的爬虫网络系统或者是聚焦爬虫这种比较高级的，大量依赖算法进行网页分析筛选过滤，然后根据一定的搜索策略从队列中选择URL进行爬取的，可能是会比较难的。但是对于比较传统爬虫，定向的URL抓取网页，就比较简单了，其实就是**发起一个或者若干个网络请**求，然后拿到请求结果（响应体，可以是网页、数据包、文件等等），根据我们的需求**处理请求结果**，拿到我们想要的数据，然后把**数据持久化**，存储到数据库或者文件里面等等，有必要的话还可以对数据做可视化等，这个就涉及到另一块领域了——数据分析。

所以总结一下，爬虫最主要的就是三个部分：

 - 数据采集（发起网络请求，得到请求结果）
 - 处理请求结果，得到想要的数据
 - 数据持久化（写到数据库或者文件里面）


### 数据采集
数据采集其实就是发起一个请求，所以我们有必要了解一下http请求：

 1. URL
 2. 请求方法（GET、POST）
 3. 请求头部Headers
 4. 请求内容（POST：body；GET：query）
 5. 响应内容
 
发起一个请求方式有很多种，在nodejs里面可以用API中的http.requset，也可以用其它第三方模块比如request、superagent等，python里面可以用requests库

    var request = require('request');
    var iconv = require('iconv-lite');
    var options = { 
        method: 'GET',
        url: "http://search.51job.com/list/080200,000000,0000,00,9,99,%2520,2,1.html",
        gzip: true,
        encoding: null,
        timeout: 6000,
        headers: {
            'Accept':'text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8',
            'Accept-Encoding': 'gzip, deflate',
            'Accept-Language': 'zh-CN,zh;q=0.8,en;q=0.6',
            'Referer': "http://www.51job.com/",
            'User-Agent': "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_12_6) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/61.0.3163.100 Safari/537.36",
            'Connection': 'keep-alive',
            'Upgrade-Insecure-Requests': '1'
        },
    };
    
    request(options, function(error, response, body) {
      if (!response || response.statusCode != 200) {
        console.log('request failed');
        return false;
      }
      if (error) {
        console.log(error);
        return false;
      }
    
      var html = iconv.decode(body, 'GB2312');
    });
    
有时候请求了却没有返回内容，或者是请求被限制了......因为有爬虫，肯定也会有反爬虫。
#### 1.Headers
现在很多网站都会有反爬虫机制，主要用来区别这个请求到底是正常的设备浏览器访问呢，还是程序代码的直接的恶意的请求。所以我们要**尽量把自己的请求伪装成是正常浏览器发出的**，这个里面比较重要或者是比较基础的一部分就是头部信息（Headers）。
我们可以用浏览器打开一个我们需要爬取的网站，然后f12打开开发者模式，在Network里面可以看到请求和响应信息。

![Request Headers][1]


  [1]: https://raw.githubusercontent.com/BigFly2333/what-is-spider/master/public/images/network.jpg
  
其实大部分头部信息可以直接复制过来，有一些信息是会被一些服务器进行验证的：

 - Referer：上一级URL，就是判断这个请求是从哪里过来的，如果是空或者一些奇怪的地址，有些服务端就会进行限制。
 - User-Agent：保存了请求设备及浏览器的一些信息，有的服务端会根据这个字段来判断，这个请求是否真的设备或浏览器端发出的。有的时候会出现同一浏览器或设备访问频率过高等问题，这个时候服务端也有可会用该字段来判断（更有效的办法可能是用ip判断吧）进行限制。所以我们可以**在爬虫系统中配置多个User-Agent，每次请求的时候随机获取**使用，这样可以防止同一User-Agent请求频率过高的问题。在python的scrapy爬虫框架中，可以在settings里面配置一个`USER_AGENTS`集合，系统就会随机获取。
给大家推荐一个UserAgent大全网站：https://developers.whatismybrowser.com/useragents/explore/  各种设备，系统，各种浏览器都有
 - Cookie：一般在用户登录或者某些操作后，服务端会在返回包中包含Cookie信息要求浏览器设置Cookie，没有Cookie会很容易被辨别出来是伪造请求；也有本地通过JS，根据服务端返回的某个信息进行处理生成的加密信息，设置在Cookie里面
 - 其他字段：有些服务端也会根据其他字段进行判断限制，有时候还会设置自定义的字段，所以要弄清楚里面的原理，还是要耐心的分析


#### 2.代理ip
还有一块比较重要的反爬虫机制是ip限制，每次请求其实都会携带一个本地ip地址，服务端可以获取到这个ip来进行限制。
所以我们需要用代理ip访问，把自己的ip地址隐藏起来。代理ip一般有三种：

 1. **高匿名代理：**  不会改变客户机的请求，在服务端看来就像是一个真正的客户浏览器再访问，而客户机的真实ip是被隐藏的，服务端也不会认为是使用了代理。
 2. **普通匿名代理：**  能够隐藏客户机的真实ip，但是会改变请求信息，在请求体里面会有代理转发的标记，服务端则可能会识别出来认为是使用代理的，虽然服务端可能不会知道你的ip地址，但是有些网站会把使用代理的请求都限制掉。当然一些能够侦查ip的网站还是能够找到你的真实ip的。
 3. **透明代理：**  不仅会改变请求信息，还会发送真实ip 
 
如何判别一个代理ip是否是匿名或者透明，因为我们用的代理是最好是匿名的，这样才能保证我们的真实能够被隐藏，不被限制：
 1. 我个人用的比较简单的办法是用这个代理ip去请求 http://ip.chinaz.com/getip.aspx （检查当前ip的网站），如果显示的当前ip与真实ip相同，则判断它是透明的，否则判定它为匿名的。在实际应用中个人感觉可以简单的判断出匿名和透明，但是没有具体测试过（如有不对望指出）
 2. 网上也搜到了一些办法，比较好的是：自建一个简单的服务器，然后用代理ip去访问这个服务器，在服务端对这个请求进行判断，主要的几个字段有：  
 `REMOTE_ADDR`: 访问客服端的ip地址  
 `HTTP_VIA`: 如果有该条信息，说明使用了代理服务器，代理服务器的地址就是后面的值  
 `HTTP_X_FORWARDED_FOR`:如果有该条信息，也说明使用了代理服务器，代理服务器的地址就是后面的值  
    **没有使用代理服务器的情况：**  
    REMOTE_ADDR = 您的 IP；HTTP_VIA = 没数值或不显示；HTTP_X_FORWARDED_FOR = 没数值或不显示  
    **使用透明代理服务器的情况：Transparent Proxies**  
    REMOTE_ADDR = 代理服务器 IP；HTTP_VIA = 代理服务器 IP；HTTP_X_FORWARDED_FOR = 您的真实 IP  
    **使用普通匿名代理服务器的情况：Anonymous Proxies**  
    REMOTE_ADDR = 代理服务器 IP；HTTP_VIA = 代理服务器 IP；HTTP_X_FORWARDED_FOR = 代理服务器 IP  
    **使用欺骗性代理服务器的情况：Distorting Proxies**  
    REMOTE_ADDR = 代理服务器 IP；HTTP_VIA = 代理服务器 IP；HTTP_X_FORWARDED_FOR = 随机的 IP  
    **使用高匿名代理服务器的情况：High Anonymity Proxies (Elite proxies)**  
    REMOTE_ADDR = 代理服务器 IP；HTTP_VIA = 没数值或不显示；HTTP_X_FORWARDED_FOR = 没数值或不显示

为了保持爬虫的效率，我们需要一个或者一系列代理ip，ip的稳定性也是比较重要的。我们当然可以花钱去买一个，这样是最省事的。有一些网站上也会有免费的代理放出来（百度搜一下代理ip，会有很多），不过这些大部分都是不稳定的，所以我们必须自建一个系统去不断的筛选过滤它们，可以搞一个自己的**ip代理池**：

 - 找一些有免费代理ip的网站（多找一些）
 - 爬取网站上的ip，进行可用性过滤（用代理请求一个网站，如果请求成功返回码是200，并且能正常返回返回体），再进行是否匿名过滤
 - 然后将我们筛选出来的ip存储到数据库中
 - 定期再进行过滤、爬取（重复上面的步奏），因为网上的这些免费的代理ip存活时间参差不齐，很多只能存活很短的时间，所以要制定一个策略，来尽可能保持数据库里面的代理ip持续可用
 
#### 3.动态网站  
 **静态：** 就是服务端进行数据渲染，然后将数据组装渲染好的html返回给客户端，这样我们再response body里面就能直接拿到html，里面就有我们想要的数据。  
 **动态：** 有的网站就很皮了，它网站上的一部分数据可能是动态渲染的，就是通过js ajax请求数据，动态渲染到页面上。  
   - 我们可以通过找到他的具体ajax请求，去模拟请求拿到数据，但是这种方式，相对会比较麻烦，因为一来一般这种直接请求数据的接口都会有一定的安全机制，不太容易去破解，然后这种数据一般不会是明文的，不好解析。  
   - 结合PhantomJS，对网页进行动态渲染，然后截取渲染完成后的网页。PhantomJS是一个基于webkit的JavaScript API。可以简单的把它理解为一个无界面的浏览器，它能动态执行js，操作dom元素，网页截图等，功能强大。
 
 ### 数据处理
 
 #### 1.非结构化数据
 ##### 1.1 html  
 我们一般写爬虫多是会去爬一些网站，这个时候我们得到的就会是一大段html代码，相对与普通的文本来说html已经算是比较结构化的文档了  
  - 对于网站比较固定的爬虫，就比如说只爬一个网站的，这样爬下来的html结构还是比较固定的，我们就可以用css选择器（ https://www.w3.org/TR/selectors/ ）、XPATH（ https://www.w3.org/TR/xpath/ ），来对html进行解析，nodejs中我们可以用cheerio模块（和jquery的用法相似，非常方便），python中scrapy框架有内置的选择器模块，也可以使用第三方库BeautifulSoup等。再配合正则表达式，就可以获取我们想要的数据  
  - 但是对于网站相对不固定的，比如爬百度搜索，一页下来十几个网站，结构都不一样的，就无法用一套选择器来解析了。第一种办法，可以用选择器把整个body部分的文本内容取过来，然后就相当于解析文本一样，用正则或者其他办法来解析；还有就是通过一定的算法，对网页内容进行过滤，获取到正文部分（一般我们需要的东西会是在正文部分），然后再进行正则等，这样相对来说无效内容就会少很多，这方面python就会有优势了，也会有一些这方面的现成库可以使用，比如python-readability、newspaper等  
  
 ##### 1.2 纯文本  
   - 正则匹配  
   - 分词：根据抓取的网站类型，使用不同词库，进行基本的分词，然后变成词频统计，类似于向量的表示，词为方向，词频为长度。  
   - NLP：自然语言处理，进行语义分析，用结果表示，例如正负面等。推荐一个免费的工具包：thulac，有python、java、c++和so等多个办法，是清华大学自然语言处理与社会人文计算实验室研制推出的一套中文词法分析工具包（ http://thulac.thunlp.org/ ）  
   
 #### 2.结构化数据  
 比如json数据之类的，这种结构化的数据就比较好解析，获取某一个你想要的字段即可
