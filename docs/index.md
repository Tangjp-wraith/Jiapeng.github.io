---
hide:
  #- navigation # 显示右
  #- toc #显示左
  - footer
  - feedback
comments: false
---
<!--
██╗    ██╗ ██████╗ ██████╗ ██╗    ██╗██╗███╗   ██╗
██║    ██║██╔════╝██╔═══██╗██║    ██║██║████╗  ██║
██║ █╗ ██║██║     ██║   ██║██║ █╗ ██║██║██╔██╗ ██║
██║███╗██║██║     ██║   ██║██║███╗██║██║██║╚██╗██║
╚███╔███╔╝╚██████╗╚██████╔╝╚███╔███╔╝██║██║ ╚████║
 ╚══╝╚══╝  ╚═════╝ ╚═════╝  ╚══╝╚══╝ ╚═╝╚═╝  ╚═══╝
                                                  


____    __    ____  ______   ______   ____    __    ____  __  .__   __. 
\   \  /  \  /   / /      | /  __  \  \   \  /  \  /   / |  | |  \ |  | 
 \   \/    \/   / |  ,----'|  |  |  |  \   \/    \/   /  |  | |   \|  | 
  \            /  |  |     |  |  |  |   \            /   |  | |  . `  | 
   \    /\    /   |  `----.|  `--'  |    \    /\    /    |  | |  |\   | 
    \__/  \__/     \______| \______/      \__/  \__/     |__| |__| \__| 

 __        __                _       
 \ \      / /__ _____      _(_)_ __  
  \ \ /\ / / __/ _ \ \ /\ / / | '_ \ 
   \ V  V / (_| (_) \ V  V /| | | | |
    \_/\_/ \___\___/ \_/\_/ |_|_| |_|
                                     
 ___       ___     ____     ____     ___       ___    _____      __      _  
(  (       )  )   / ___)   / __ \   (  (       )  )  (_   _)    /  \    / ) 
 \  \  _  /  /   / /      / /  \ \   \  \  _  /  /     | |     / /\ \  / /  
  \  \/ \/  /   ( (      ( ()  () )   \  \/ \/  /      | |     ) ) ) ) ) )  
   )   _   (    ( (      ( ()  () )    )   _   (       | |    ( ( ( ( ( (   
   \  ( )  /     \ \___   \ \__/ /     \  ( )  /      _| |__  / /  \ \/ /   
    \_/ \_/       \____)   \____/       \_/ \_/      /_____( (_/    \__/    
                                                                            


-->
# 👋欢迎                                   

<center><font  color= #518FC1 size=6 class="ml3">0X10CC的代码空间</font></center>
<script src="https://cdnjs.cloudflare.com/ajax/libs/animejs/2.0.2/anime.min.js"></script>



<center>
<font  color= #608DBD size=3>
<span id="jinrishici-sentence">太阳总是能温暖向日葵</span>
<!-- <script src="https://sdk.jinrishici.com/v2/browser/jinrishici.js" charset="utf-8"></script> -->
</font>
</center>



## 📒关于笔记

**这个笔记本更像一个助记簿**，记录一些自己学习过程中觉得十分重要的知识，方便自己查阅。

???+ example "📚内容分类"
    - [Langs](langs/index.md)
          - C++零碎杂记
          - Effective Modern C++ 阅读笔记（有空一定读完）
    - [计算机基础](core/index.md)
          - CMU15-213：CSAPP
    - [Tools](tools/index.md)








   <body>
        <font color="#B9B9B9">
        <p style="text-align: center; ">
                <span>本站已经运行</span>
                <span id='box1'></span>
    </p>
      <div id="box1"></div>
      <script>
        function timingTime(){
          let start = '2024-2-8 00:00:00'
          let startTime = new Date(start).getTime()
          let currentTime = new Date().getTime()
          let difference = currentTime - startTime
          let m =  Math.floor(difference / (1000))
          let mm = m % 60  // 秒
          let f = Math.floor(m / 60)
          let ff = f % 60 // 分钟
          let s = Math.floor(f/ 60) // 小时
          let ss = s % 24
          let day = Math.floor(s  / 24 ) // 天数
          return day + "天" + ss + "时" + ff + "分" + mm +'秒'
        }
        setInterval(()=>{
          document.getElementById('box1').innerHTML = timingTime()
        },1000)
      </script>
      </font>
    </body>



<!-- <script src="//code.tidio.co/6jmawe9m5wy4ahvlhub2riyrnujz7xxi.js" async></script> -->


