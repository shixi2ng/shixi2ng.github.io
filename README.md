
<!DOCTYPE html>
<html>

  <head>
  <meta charset="utf-8">
  <meta http-equiv="X-UA-Compatible" content="IE=edge">
  <meta name="viewport" content="width=device-width, initial-scale=1.0,minimum-scale=1.0, maximum-scale=1.0, user-scalable=no">

  <!-- 添加搜索关键字 -->
  <meta name="keywords" content="梦回少年,梦回,saber,lemonjing,少年">

  <title>【Spark】Spark Streaming中复杂的多流Join方案的一个实现 | 梦回少年</title>
  <meta name="description" content="问题：多个不同流根据一定规则join的问题（例如：网约车中订单发单流与接单流join问题）">

  <link rel="stylesheet" href="/assets/css/bootstrap.css">
  <link rel="stylesheet" href="/assets/css/font-awesome.css">
  <link rel="stylesheet" href="/assets/css/main.css" >
  <link rel="stylesheet" href="/assets/gitment/default.css" >
  <link rel="stylesheet" href="/assets/js/prettify/prettify.css">
  <link rel="shortcut icon" href="/assets/img/favicon.ico" />
  <link rel="canonical" href="http://rann.cc/2018/05/23/spark-streaming-stream-join.html">
  <link rel="alternate" type="application/rss+xml" title="梦回少年" href="http://rann.cc/feed.xml" />
  <script src="/assets/gitment/gitment.browser.js"></script>
</head>


  <body>

    <header class="site-header">
    <div class="container">
        <div class="row">
            <nav class="navbar navbar-default" role="navigation">
                <div class="navbar-header col-xs-12 col-sm-12 col-md-3 col-lg-3 center">
                    <button type="button" class="navbar-toggle" data-toggle="collapse" data-target="#example-navbar-collapse">
                        <span class="icon-bar"></span>
                        <span class="icon-bar"></span>
                        <span class="icon-bar"></span>
                    </button>
                    <div class="sitehome">
                        <a href="/" title="首页"><i class="fa fa-home fa-2x homeicon"></i><a><a class="site-title" href="/">梦回少年</a>
                    </div>
                </div>
                <div class="col-md-6 col-lg-6 hidden-xs hidden-sm center">
                    <nav class="site-nav">
                        <ul class="nav nav-pills">
                            <li class="select"><a class="page-link pjaxlink" href="/pages/read.html">读书</a></li>
                            <li class="select"><a class="page-link pjaxlink" href="/pages/class.html">分类</a></li>
                            <!-- <li class="select"><a class="page-link pjaxlink" href="/pages/tags.html">标签</a></li> -->
                            <li class="select"><a class="page-link pjaxlink" href="/pages/archive.html">归档</a></li>
                            <li class="select"><a class="page-link pjaxlink" href="/pages/about.html">关于</a></li>
                        </ul>
                    </nav>
                </div>
                <!-- swifttype 搜索 弃用-->
               <!--  <div class="col-md-3 col-lg-3 hidden-xs hidden-sm">
                    <div class="search">
                        <input type="text" class="search-query st-default-search-input" placeholder="Search">
                    </div>
                </div> -->
                 <!-- /swifttype 搜索 -->
                <div class="col-md-3 col-lg-3 hidden-xs hidden-sm">
                    <div style="position: fixed; top:10px;">
                        <img src="/search/img/cb-search.png" id="cb-search-btn" title="双击ctrl试一下" />
                    </div>
                </div>
                   
                    <div class="collapse navbar-collapse" id="example-navbar-collapse">
                        <ul class="nav navbar-nav phone-nav center">
                            <li class="phoneselect"><a class="page-link pjaxlink" href="/"><i class="fa fa-home"></i>&nbsp;首页</a></li>
                            <li class="phoneselect"><a class="page-link pjaxlink" href="/pages/read.html"><i class="fa fa-book"></i>&nbsp;读书</a></li>
                            <li class="phoneselect"><a class="page-link pjaxlink" href="/pages/class.html"><i class="fa fa-tasks"></i>&nbsp;分类</a></li>
                            <li class="phoneselect"><a class="page-link pjaxlink" href="/pages/tags.html"><i class="fa fa-tags"></i>&nbsp;标签</a></li>
                            <li class="phoneselect"><a class="page-link pjaxlink" href="/pages/archive.html"><i class="fa fa-archive"></i>&nbsp;归档</a></li>
                            <li class="phoneselect"><a class="page-link pjaxlink" href="/pages/about.html"><i class="fa fa-user"></i>&nbsp;关于</a></li>
                        </ul>
                    </div>
            </nav>
            </div>
        </div>
</header>


    <div class="content">
   	<div class="container">	
   		<div class="row">
			<div class="col-md-3 col-lg-3 hidden-xs hidden-sm aside1 fadein-left">
				<div class="profile box-shadow center">
					<div class="overlay"></div>
					<div class="center gavatar">
						<a href="/" class="profile_gavatar"><img class="circle" src="/assets/img/saber.jpg"></a>
					</div>
					<div class="address">
						<h5><span class="fa fa-map-marker"></span> Hangzhou, China</h5>
					</div>
					<div class="center profile_desc">
						技术<br>生活<br>和爱情<br>记录点滴 珍藏回忆<br>
					</div>
				</div>
				
				<div class="tag-cloud-text">
					<a href="http://rann.cc/pages/class.html" title="分类" class="pjaxlink"><p class="center">分类</p></a>
				</div>
				<div class="tag-cloud ">	
					<hr>
					<div class="page-tag">
							
								<a href="http://lemonjing.github.io/pages/class.html#算法" name="算法" class="pjaxlink">算法(19)</a>
							
								<a href="http://lemonjing.github.io/pages/class.html#Java" name="Java" class="pjaxlink">Java(41)</a>
							
								<a href="http://lemonjing.github.io/pages/class.html#随笔" name="随笔" class="pjaxlink">随笔(19)</a>
							
								<a href="http://lemonjing.github.io/pages/class.html#技术" name="技术" class="pjaxlink">技术(17)</a>
							
								<a href="http://lemonjing.github.io/pages/class.html#求职" name="求职" class="pjaxlink">求职(3)</a>
							
								<a href="http://lemonjing.github.io/pages/class.html#读书" name="读书" class="pjaxlink">读书(1)</a>
							
								<a href="http://lemonjing.github.io/pages/class.html#大数据" name="大数据" class="pjaxlink">大数据(13)</a>
							
					</div>					
				</div>
				<div class="clear"></div>

				<div class="tag-cloud-text">
					<a href="http://rann.cc/pages/tags.html" title="标签" class="pjaxlink"><p class="center">标签【点我】</p></a>
				</div>
				<div class="tag-cloud ">	
					<hr>
					<div class="page-tag">
							
								
								<a href="http://lemonjing.github.io/pages/tags.html#原创" name="原创" class="pjaxlink"><i class="fa fa-tags"></i>原创(57)</a>
								
							
								
								<a href="http://lemonjing.github.io/pages/tags.html#Algorithm" name="Algorithm" class="pjaxlink"><i class="fa fa-tags"></i>Algorithm(19)</a>
								
							
								
							
								
							
								
							
								
								<a href="http://lemonjing.github.io/pages/tags.html#背包" name="背包" class="pjaxlink"><i class="fa fa-tags"></i>背包(3)</a>
								
							
								
							
								
							
								
							
								
							
								
							
								
								<a href="http://lemonjing.github.io/pages/tags.html#JavaSE" name="JavaSE" class="pjaxlink"><i class="fa fa-tags"></i>JavaSE(11)</a>
								
							
								
								<a href="http://lemonjing.github.io/pages/tags.html#随笔" name="随笔" class="pjaxlink"><i class="fa fa-tags"></i>随笔(14)</a>
								
							
								
							
								
							
								
							
								
							
								
							
								
							
								
							
								
							
								
								<a href="http://lemonjing.github.io/pages/tags.html#JavaWeb" name="JavaWeb" class="pjaxlink"><i class="fa fa-tags"></i>JavaWeb(3)</a>
								
							
								
							
								
							
								
								<a href="http://lemonjing.github.io/pages/tags.html#校招" name="校招" class="pjaxlink"><i class="fa fa-tags"></i>校招(3)</a>
								
							
								
							
								
							
								
								<a href="http://lemonjing.github.io/pages/tags.html#Spring" name="Spring" class="pjaxlink"><i class="fa fa-tags"></i>Spring(10)</a>
								
							
								
							
								
							
								
							
								
							
								
							
								
							
								
							
								
							
								
							
								
							
								
								<a href="http://lemonjing.github.io/pages/tags.html#NIO" name="NIO" class="pjaxlink"><i class="fa fa-tags"></i>NIO(13)</a>
								
							
								
							
								
							
								
							
								
							
								
							
								
							
								
							
								
							
								
								<a href="http://lemonjing.github.io/pages/tags.html#Spark" name="Spark" class="pjaxlink"><i class="fa fa-tags"></i>Spark(9)</a>
								
							
								
								<a href="http://lemonjing.github.io/pages/tags.html#Scala" name="Scala" class="pjaxlink"><i class="fa fa-tags"></i>Scala(3)</a>
								
							
								
							
								
							
					</div>					
				</div>
				<div class="clear"></div>

	 			<div class="recentuse">
	 					<p>GitHub项目</p>
	 					<hr>
	 					<ul>
	 						<li><a href = "https://github.com/it-interview/easy-job" target="_blank">互联网求职面试知识复习</a></li>
	 						<li><a href = "https://github.com/Lemonjing/LightweightApp" target="_blank">集成LBS查询、音乐、阅读的微信公众账号</a></li>
	 						<li><a href = "https://github.com/Lemonjing/hadoop-dedup" target="_blank">基于hadoop和hbase的大规模海量数据去重</a></li>
	 						<li><a href = "https://github.com/Lemonjing/TinyMooc" target="_blank">轻量级Java平台在线幕课学习网站</a></li>
		 					<li><a href = "https://github.com/Lemonjing/Keccak" target="_blank">SHA-3 algorithm Keccak Implementation</a></li>
		 					<li><a href = "https://github.com/Lemonjing/your-offer" target="_blank">剑指offer的Scala实现</a></li>
	 					</ul>
	 			</div>
	 			
	 			<div class="recentuse">
	 					<p>经常出没</p>
	 					<hr>
	 					<ul>
	 						<li><a href = "http://music.163.com/#/user/home?id=63589002" target="_blank">网易云音乐</a></li>
	 						<li><a href = "https://github.com/Lemonjing" target="_blank">GitHub</a></li>
	 						<li><a href = "http://weibo.com/u/1662536394" target="_blank">微博</a></li>
	 						<li><a href = "http://pianke.me/pages/user/user.html?uid=1924980" target="_blank">片刻</a></li>
		 					<li><a href = "https://www.zhihu.com/people/xmusaber" target="_blank">知乎</a></li>
		 					<li><a href = "http://www.nowcoder.com/profile/213475" target="_blank">牛客</a></li>
	 					</ul>
	 			</div>
				
				<div class="friendlink">
	 					<p>友情链接</p>
	 					<hr>					
		 				<a href = "https://imjad.cn/" target="_blank">AD's Blog</a></br>
						<a href = "http://zyl.me" target="_blank">ZYL的博客</a></br>
						<a href = "http://mxwu.cn" target="_blank">梦忻屋</a></br>
						<a href = "http://www.weiyanweiyu.cn" target="_blank">微言微语</a></br>
						<a href = "http://jloong.com" target="_blank">楚书业</a></br>
						<a href = "https://emiria.io/" target="_blank">蔓舞寻樱</a></br>
						<a href = "http://www.xiaokang.info" target="_blank">小康博客</a><br>
						<a href = "http://blog.iov.me" target="_blank">随心说</a></br>
						<a href = "http://czduban.com" target="_blank">以歌。先生</a><br>
						<a href = "http://5mx.net/" target="_blank">冷夜博客</a><br>
						<a href = "http://www.8hao.com.cn/" target="_blank">艳莜日记</a><br>
						<a href = "http://blog.sunxyz.cn/" target="_blank">诸葛扬阳</a><br>
						<a href = "https://lostyou.love/" target="_blank">邻居</a><br>
						<a href = "http://www.kurodown.com/" target="_blank">酷绒站</a>
	 			</div>
				
			</div>
			
			<div class="col-xs-12 col-sm-12 col-md-9 col-lg-9 box-shadow fadein-right aside2">					 		
					<div class="page-content" id="pjax"><div class="post">
  <header class="post-header">
    <h1 class="post-title">【Spark】Spark Streaming中复杂的多流Join方案的一个实现</h1>
    <div class="info">
	    <p class="post-meta"><i class="fa fa-calendar"></i>&nbsp;2018-05-23</p>
	    
		<i class="fa fa-tags"></i>
		<span class="index-post-tag">
		
			<a class="pjaxlink" href="/pages/tags.html#原创">原创</a>
		
			<a class="pjaxlink" href="/pages/tags.html#Spark">Spark</a>
		
		</span>
	    
	    <i class="fa fa-eye"></i>访问量<span id="busuanzi_value_page_pv">-1</span>	
    </div>
  </header>

  <article class="post-content">
    <p>问题：多个不同流根据一定规则join的问题（例如：网约车中订单发单流与接单流join问题）</p>

<h2 id="问题">问题</h2>

<p>描述：多个不同流根据一定规则join的问题（例如：网约车中订单发单流与接单流join问题）</p>

<p>特点：</p>
<ul>
  <li>不同流需要join的数据时间跨度较长（例如：发单与接单时间跨度最长一周之久）</li>
  <li>数据源格式不定 （例如：binlog数据和业务日志的join）</li>
  <li>join规则多样化</li>
  <li>系统要求吞吐量大（订单表流量是5M/s） 、延迟低（秒级）</li>
</ul>

<h2 id="分析">分析</h2>

<p>显然根据窗口实现是不可取的，首先多流之间跨度较大，窗口无法支持时间跨度这么大的延迟。为此，我们需要一个高效的，具有持久化功能的Cache服务，来缓存先到的流。</p>

<p>并且针对特殊业务，我们需要支持流的保序性。流的保序性是我定义的一个说法（或名词），它指的是如果数据流中存在多张表的数据，而这些表依照一个次序由业务发过来。（如业务数据落到MySQL Binlog，然后可以按照订单id partition到Kafka Topic）我们在下游处理过程和Join的过程中，需要对流中的分表保序。保序要注意的几点是可以按照主键id（订单id）取哈希作为partition key，确保同样主键的数据落到下游同partition的topic，值得注意的一点是，如果Executor端使用了Producer池的话，要确保采用同一个Producer发送。可采取主键id的哈希值对池大小取模的方式来做。</p>

<p>这里保序主要为了确保多流Join时如果有非对等流，即某一个流到达后需要输出它的相关字段，即使没有Join上。（如成单的数据，业务确保了成单状态一定出现在创建订单之前）。</p>

<h2 id="方案">方案</h2>

<p>为了解决上述的多流Join问题，进行了如下的方案实现。</p>

<p><img src="https://raw.githubusercontent.com/Lemonjing/DevUtil/master/github/join.png" alt="" /></p>

<p>1.通过在Spark Streaming引擎中封装一套Cache服务（可读写外部KV存储，如Fusion、HBase），对先到达的数据流Cache住。2.将各种Join的规则配置化引入引擎，根据Join的场景按需选择规则进行应用。在Join过程中，缓存流在Join上之前一直保持，Join上后进行释放。（这里可能会涉及到KV存储remove操作的性能问题，可进行put的替代或假删）</p>

<p>注：通过引入外部KV存储后，对于作业的延迟或异常问题，也需要关注KV存储（如Fusion、HBase）的集群运行情况。</p>

<p>本人系作者原创，欢迎Spark、Flink等大数据技术方面的探讨。</p>

<p>ps：公众号已正式接入图灵机器人，快去和我聊聊吧。</p>

<center>-END-</center>

<div align="center">
<img src="http://rann.cc/assets/img/qrcode-logo.png" width="340" height="400" />
</div>

<blockquote>
  <p>本文系本人个人公众号「梦回少年」原创发布，扫一扫加关注。</p>
</blockquote>
	
  </article>
  <div id="container"></div>
<!-- <script src="https://imsun.github.io/gitment/dist/gitment.browser.js"></script> -->
<script>
var gitment = new Gitment({
  id: window.location.pathname, // 可选。默认为 location.href
  owner: 'lemonjing',
  repo: 'lemonjing.github.io',
  oauth: {
    client_id: 'd434df4970edfd9beba6',
    client_secret: '72e0e4bc8d7dd8f6abb250576aa1f1b6e6f67c1b',
  },
})
gitment.render('container')
</script>
</div>

  <div class="prevandnext">
    	  
	    <div style="margin:0.5em;">
	    <span>上一篇 ：</span><a class="pjaxlink" href="http://rann.cc/2018/08/23/sql-optimized-principles.html"  title="【SQL】SQL优化器原理——查询优化器综述">【SQL】SQL优化器原理——查询优化器综述</a>
	    </div>
	  
  	  
	    <div style="margin:0.5em;">
	    <span>下一篇 ：</span><a class="pjaxlink" href="http://rann.cc/2017/12/23/spark-hostname-issues.html"  title="【Spark】Spark针对https和IPv6的主机名处理的两个issue分析">【Spark】Spark针对https和IPv6的主机名处理的两个issue分析</a>
	    </div>
	  
	  	<div style="margin:0.5em;">
			<span> 版权所有，转载时必须以链接形式注明原始出处</span>
	    </div>
	 
  </div></div>		
			</div>
 		</div>	
	</div>
    </div> 
</div>
	<div class="profile_social">
		<a class="rss" href="/feed.xml" target="_blank"></a>
		<a class="github" href="https://github.com/lemonjing"  target="_blank"></a>
		<a class="weibo" href="http://weibo.com/u/1662536394"  target="_blank"></a>
	</div>
    <div id="backtotop">
    		<a href="#"><i class="fa fa-arrow-circle-up"></i></a>
    </div>
    <div class="pjax_loading"></div>
    
    <footer class="site-footer">
  <div class="wrapper">

    <h2 class="footer-heading">saber's blog</h2>

    <div class="footer-col-wrapper">
      <div class="footer-col  footer-col-1">
        <ul class="contact-list">
          <li>梦回少年</li>
          <li><a href="mailto:xmusaber@163.com">xmusaber@163.com</a></li>
          <li>闽ICP备15018990号</li>
        </ul>
      </div>

      <div class="footer-col  footer-col-2">
        <ul class="social-media-list">
          
          <li>
            <a class="github" href="https://github.com/lemonjing"  target="_blank"></a>
          </li>
          

          
          <li>
            <a class="weibo" href="http://weibo.com/u/1662536394"  target="_blank"></a>
          </li>
          
        </ul>
      </div>
      <div class="footer-col  footer-col-3">
        <p class="text">If,<br/>for example,<br/>you come at four o'clock in the afternoon,<br/>then at three o'clock I shall begin to be happy.</p>
      </div>
    </div>
</div>
    <div class="center sitedesc">
    	Powered by <a href ="http://jekyllrb.com/">Jekyll</a>  |  © 2018 梦回少年  |  Hosted on <a href="https://github.com/lemonjing/lemonjing.github.io"> Github</a></div>
    <div class="center sitedesc"><span id=span_dt_dt></span>&nbsp;&nbsp;|&nbsp;&nbsp;<span id="busuanzi_container_site_pv" style='display:none'>本站总访问量<span id="busuanzi_value_site_pv"></span>次<br/>
    <script type="text/javascript">var cnzz_protocol = (("https:" == document.location.protocol) ? "https://" : "http://");document.write(unescape("%3Cspan id='cnzz_stat_icon_1276104568'%3E%3C/span%3E%3Cscript src='" + cnzz_protocol + "s5.cnzz.com/z_stat.php%3Fid%3D1276104568%26online%3D1%26show%3Dline' type='text/javascript'%3E%3C/script%3E"));</script>
    </div>

<!-- 访问统计 -->
<script async src="https://busuanzi.ibruce.info/busuanzi/2.3/busuanzi.pure.mini.js"></script>

<!-- 萌萌哒运行 -->
  <SCRIPT language=javascript>          
function show_date_time(){
window.setTimeout("show_date_time()", 1000);
BirthDay=new Date("8/15/2015 11:30:00");//这个日期是可以修改的
today=new Date();
timeold=(today.getTime()-BirthDay.getTime());
sectimeold=timeold/1000
secondsold=Math.floor(sectimeold);
msPerDay=24*60*60*1000
e_daysold=timeold/msPerDay
daysold=Math.floor(e_daysold);
e_hrsold=(e_daysold-daysold)*24;
hrsold=Math.floor(e_hrsold);
e_minsold=(e_hrsold-hrsold)*60;
minsold=Math.floor((e_hrsold-hrsold)*60);
seconds=Math.floor((e_minsold-minsold)*60);
span_dt_dt.innerHTML="本站已萌萌哒运行"+daysold+"天"+hrsold+"小时"+minsold+"分"+seconds+"秒";
}show_date_time();</SCRIPT>

  <!-- jQuery -->
  <script type="text/javascript" src="/assets/js/jquery.min.js"></script>
  <script type="text/javascript" src="/assets/js/jquery.pjax.js"></script>
  <!-- Bootstrap Core JavaScript -->
  <script type="text/javascript" src="/assets/js/bootstrap.min.js"></script>
  <script type="text/javascript" src="/assets/js/prettify/prettify.js"></script>

  <!-- jekyII search-->
<div class="cb-search-tool" style="position: fixed; top: 0px ; bottom: 0px; left: 0px; right:  0px;
      opacity: 0.95; background-color: #111111; z-index: 9999; display: none;">
    <input type="text" class="form-control cb-search-content" id="cb-search-content" style="position: fixed; top: 60px" placeholder="文章标题 日期 标签" >

    <div style="position: fixed; top: 16px; right: 16px;">
        <img src="/search/img/cb-close.png"  id="cb-close-btn"/>
    </div>
</div>

<!-- <div style="position: fixed; right: 16px; bottom: 20px;">
    <img src="/search/img/cb-search.png"  id="cb-search-btn"  title="双击ctrl试一下"/>
</div> -->

<link rel="stylesheet" href="/search/css/cb-search.css">

<script src="/search/js/bootstrap3-typeahead.min.js"></script>
<script src="/search/js/cb-search.js"></script>
<!-- jekyII search end -->

  
<script type="text/javascript" src="http://echarts.baidu.com/build/dist/echarts.js"></script>
<script type="text/javascript" src="/assets/js/main.js"></script>

</footer>

  </body>

</html>
