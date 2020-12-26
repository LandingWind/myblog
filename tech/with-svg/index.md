# 与svg动画纠缠不清的两天


这次的文章主题是svg，预备知识基本是没有的

有几个应用过程中遇到的bug，po出来供大家参考避免走同样的弯路。

最近写了个社团的秋纳报名表，刚好UI设计人员给的设计稿有svg切图，以前从来没有接触过svg，这次有时间就尝试了一下，感觉还不错，虽然对低版本浏览器的兼容性是真的差

svg最大的优点就是矢量，资源体积很小，但是显示效果非常棒，对各种大小屏幕的适配性也是极其ok，耗费的主要是即时的计算和渲染

百度或者google一下，很容易找到关于svg的文档，也很容易找到关于svg嵌入html的几种方式的介绍

目前兼容性最好的是embed，不过调整大小我觉得麻烦，最后我还是直接用的svg标签形式嵌入网页。

```html
<svg 
    xmlns="http://www.w3.org/2000/svg"
    xmlns:xlink="http://www.w3.org/1999/xlink" 
    version="1.1"
    width="100%" height="100%"
    viewBox="0 0 2000 2300"
    >
    <!-- background -->
    <image  x="0" y="0" width="100%" xlink:href="./static/Rectangle.svg"></image>
    <!-- moon -->
    <image  id="moon2" x="95%" y="24%" width="11%" xlink:href="./static/Oval Copy 3.svg"></image>
    <image  id="moon1" x="-10%" y="10%" width="16%" xlink:href="./static/Group.svg"></image>
    <image  id="moon3" x="80%" y="63%" width="14%" xlink:href="./static/Oval Copy 3.svg"></image>
    <!-- cloud -->
    <image  id="cloud1" x="-8%" y="23%" width="16%" xlink:href="./static/cloud1.svg"></image>
    <image  id="cloud2" x="45%" y="-13%" width="45%" xlink:href="./static/cloud2.svg"></image>
    <image  id="cloud3" x="60%" y="-13%" width="35%" xlink:href="./static/cloud3.svg"></image>
    <image  id="cloud4" x="95%" y="35%" width="10%" xlink:href="./static/cloud4.svg"></image>
    <image  id="cloud5" x="30%" y="61%" width="18%" xlink:href="./static/cloud5.svg"></image>
    <!-- line -->
    <image  id="line" x="0%" y="2%" width="100%"
        xlink:href="./static/line.svg"></image>
    <!-- title -->
    <image  id="title" x="30%" y="37%" width="40%"
        xlink:href="./static/UIslices/PAPIC.png"></image>
    <!-- word -->
    <text id="svgtext" opacity="0" x="32.5%" y="48%" fill="white">浙大勤创秋季纳新</text>
    <!-- word-underline -->
    <line id="textline" opacity="0" x1="25%" y1="41.5%" x2="75%" y2="41.5%" stroke="white" stroke-width="2"/>
</svg>
```

php主体页面插入svg以及其子元素，链接css文件获取相应元素的动画代码

这样写在当代大多数浏览器上运行流畅，没有任何问题(不包括IE系列和edge) 但是到了各种机型的兼容性测试的时候，发现个别ios版本的微信和qq的内置浏览器无法运行，显示为一片空白

**强烈吐槽腾讯的x5内核的浏览器！**

开始以为是浏览器版本低，不支持svg。不支持就显示静态图片咯！

所以我添加了js代码在网页运行开始判断是否支持svg功能。如下：

```html
<script type="text/javascript">
$(document).ready(function(){
    if (typeof SVGRect !== "undefined") 
    {  /* If the browser does support SVG. */ 
    console.log("support svg!");
    //$("embed").hide();
    }
    else 
    {  /* If the browser does not support SVG. */ 
    alert("do not support svg!");
    }
});
</script>
```

但是没用啊，所有的机子都告诉我他是支持svg的，真是打肿脸充胖子。气死！ 

这里经过了漫长的搜索，baidu、bing、yahoo、google一个个搜过去，中间发现了一个提供各种浏览器版本对于各种功能的兼容性的信息网，收藏！[https://caniuse.com/](https://caniuse.com/) 

经过不断的尝试，发现最大的问题在于svg爸爸下面的image子标签，显示有问题；而其他建立的line、text标签毫无问题，只是因为颜色是白色，在背景不显示的情况下看不出来。 怎么办？ 继续google，不断改换关键字，各种尝试，无果，放弃，睡觉！ 

最终解决这个问题的是js去动态渲染，这个方法成功解决了浏览器的兼容的问题，当然，我们不管IE9版本以下的破烂了。 

这里推荐两个svg的js框架。分别是 **svg.js** 和 **snap.svg**。 我最后用的是svg.js，因为根据别人的博客写到，功能虽少些，但是兼容性更加好。自然选他了。 

文档写的很明白，这里就不作更多关于这个框架的介绍了。确实方便，css3代码不用写了，网页dom树只要写好一个根节点，所有的渲染都通过js生成，恩很react。 下面展示一下我的简单动画所需的js代码

```html
<script type="text/javascript">
    $(document).ready(function() {
        var draw = SVG('mysvgbox').size('100%','100%');
        var image = draw.image('static/Rectangle.png');
        image.attr({x:0,y:0,id:'svgboxbgpic'});
        image.size('100%');
        $("#svgboxbgpic").css('height','auto')

        var moon1 = draw.image('static/Group.svg');
        moon1.attr({width:'16%',height:'16%',x:'-10%',y:'10%'});
        moon1.animate({duration:1500}).move('20%','10%')
            .animate({duration:300}).move('22%','10%')
            .animate({duration:300}).move('20%','10%')
            .animate({duration:300}).move('22%','10%');

        var moon2 = draw.image('static/Oval Copy 3.svg');
        moon2.attr({width:'11%',height:'11%',x:'95%',y:'24%'});
        moon2.animate({duration:3000}).move('65%','24%');   

        var moon3 = draw.image('static/Oval Copy 3.svg');
        moon3.attr({width:'14%',height:'14%',x:'80%',y:'63%'});

        var cloud1 = draw.image('static/cloud1.svg');
        cloud1.attr({width:'18%',height:'30%',x:'-10%',y:'23%'});
        cloud1.animate({duration:1500}).move('-2%','23%')
            .animate(300).move('-3%','23%')
            .animate(500).move('-1%','23%');  

        var cloud2 = draw.image('static/cloud2.svg');
        cloud2.attr({width:'45%',height:'27%',x:'45%',y:'-13%'});
        cloud2.animate(1500).move('45%','-2%')
            .animate(300).move('45%','-4%')
            .animate(500).move('45%','-2%');

        var cloud3 = draw.image('static/cloud3.svg');
        cloud3.attr({width:'35%',height:'23%',x:'60%',y:'-13%'});
        cloud3.animate(1500).move('60%','-2%')
            .animate(250).move('60%','-6%')
            .animate(500).move('60%','-3%');     

        var cloud4 = draw.image('static/cloud4.svg');
        cloud4.attr({width:'9%',height:'15%',x:'97%',y:'35%'});
        cloud4.animate(1500).move('91%','35%')
            .animate(300).move('92%','35%')
            .animate(500).move('91%','35%');

        var cloud5 = draw.image('static/cloud5.svg');
        cloud5.attr({width:'18%',height:'15%',x:'30%',y:'61%'}); 
        cloud5.animate(2000).move('30%','55%');

        var line = draw.image('static/line.svg');
        line.attr({width:'100%',height:'100%',x:'0',y:'-2%'});
        line.animate(2000).move('0','-12%')
            .animate(2000).move('0','-6%');  

        var title = draw.image('static/UIslices/PAPIC.png');
        title.attr({width:'40%',height:'15%',x:'30%',y:'37%'});
        title.animate(1500).move('30%','25%')
            .animate(1500).move('30%','30%');

        var text = draw.text("浙大勤创秋季纳新");
        text.attr({x:'32.5%',y:'48%',fill:'white',opacity:0})
        text.animate({duration:1000,delay:3000}).attr({opacity:1});

        var textline = draw.line().stroke({width:2});
        textline.attr({x1:"25%",y1:"45%",x2:"25%",y2:"45%",stroke:"white",opacity:0});
        textline.animate({duration:300,delay:3500}).attr({opacity:0.8,x2:'75%'});
    })
</script>
```

这是我的根节点

```html
<div id="mysvgbox">
    <!--will insert here-->
</div>
```

当然这个svg.js还有更有意思的东西，只是我的动画太简单了用不上。以后有时间一定好好玩玩

