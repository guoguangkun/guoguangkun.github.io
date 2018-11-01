---
title : canvas简介与三个实例快速上手
layout: post
date:   2017-12-11 
categories: 技术文章
---

##   简介

[<canvas>](https://developer.mozilla.org/zh-CN/docs/Web/HTML/Element/canvas) 是 [HTML5](https://developer.mozilla.org/zh-CN/docs/HTML/HTML5 "HTML") 新增的元素，可用于通过使用[JavaScript](https://developer.mozilla.org/zh-CN/docs/JavaScript "JavaScript")中的脚本来绘制图形。例如，它可以用于绘制图形，制作照片，创建动画，甚至可以进行实时视频处理或渲染。

## Canvas 与 SVG 的比较

下表列出了 canvas 与 SVG 之间的一些不同之处。

### Canvas

*   依赖分辨率

*   不支持事件处理器

*   弱的文本渲染能力

*   能够以 .png 或 .jpg 格式保存结果图像

*   最适合图像密集型的游戏，其中的许多对象会被频繁重绘

### SVG

*   不依赖分辨率

*   支持事件处理器

*   最适合带有大型渲染区域的应用程序（比如谷歌地图）

*   复杂度高会减慢渲染速度（任何过度使用 DOM 的应用都不快）

*   不适合游戏应用

[<canvas>](https://developer.mozilla.org/zh-CN/docs/Web/HTML/Element/canvas) 看起来和 [<img>](https://developer.mozilla.org/zh-CN/docs/Web/HTML/Element/img) 元素很相像，唯一的不同就是它并没有 src 和 alt 属性。实际上，<canvas> 标签只有两个属性—— [width](https://developer.mozilla.org/zh-CN/docs/Web/HTML/Element/canvas#attr-width)和[height](https://developer.mozilla.org/zh-CN/docs/Web/HTML/Element/canvas#attr-height)。这些都是可选的，并且同样利用 [DOM](https://developer.mozilla.org/en-US/docs/Glossary/DOM "DOM: The DOM (Document Object Model) is an API that represents and interacts with any HTML or XML document. The DOM is a document model loaded in the browser and representing the document as a node tree, where each node represents part of the document (e.g. an element, text string, or comment).")[properties](https://developer.mozilla.org/zh-CN/docs/Web/API/HTMLCanvasElement) 来设置。当没有设置宽度和高度的时候，canvas会初始化宽度为300像素和高度为150像素。

该元素可以使用[CSS](https://developer.mozilla.org/en-US/docs/Glossary/CSS "CSS: CSS (Cascading Style Sheets) is a declarative language that controls how webpages look in the browser.")来定义大小，但在绘制时图像会伸缩以适应它的框架尺寸：如果CSS的尺寸与初始画布的比例不一致，它会出现扭曲。

> 注意: 如果你绘制出来的图像是扭曲的, 尝试用width和height属性为<canvas>明确规定宽高，而不是使用CSS，画面扭曲或者模糊一般会出现在移动端，pc端一般没有影响，如果移动端出现锯齿，下面是一个解决移动端锯齿的方法。


```
// 去除锯齿

let canvas = document.querySelector('#gauge')

const width = canvas.width,height=canvas.height;

const cxt = canvas.getContext('2d')

if (window.devicePixelRatio) {

  canvas.style.width = width + "px";

  canvas.style.height = height + "px";

  canvas.height = height * window.devicePixelRatio;

  canvas.width = width * window.devicePixelRatio;

  cxt.scale(window.devicePixelRatio, window.devicePixelRatio);

};
```


### 用法示例（三个实例上手canvas）：

* Demo1用canvas画一个电子时钟：

效果展示：

 ![image.png](https://upload-images.jianshu.io/upload_images/6958980-d7d4ed2cc9598524.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240) 

```
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Document</title>
</head>
<body>
    <canvas class='clock'></canvas>
</body>
<script>
    function drawClock(selector,option) {
        this.box = document.querySelector(selector); //获取目标元素canvas
        this.bgcolor=option.bgcolor||'#e8e7e3';  // 默认配置背景色
        this.pointercolor=option.pointercolor||"#b1b71e"; // 默认配置指针颜色
        this.pinColor = option.pinColor || '#e5a786'// 默认及刻度颜色
        this.isShowNumber=option.isShowNumber||false // 是否显示数字时钟
        this.size=option.size||300 // 画布大小
        this.ctx = this.box.getContext('2d') // 获取2d画笔
        this.box.width = this.size //设置宽高
        this.box.height = this.size
        this.font = option.font || '48px serif'//设置默认字体
        this.linStyle = option.lineStyle || 'round'//设置线的样式
        this.minutePinWidth = option.minutePinWidth || 3 //设置指针宽度
        this.hourPinWidth = option.hourPinWidth || 6
        this.init() // 初始化
    }
    drawClock.prototype = {
        init(){
            let that = this
            this.tickTock(that) //让指针动起来
        },
        safeDraw(callback){  //每次都重新重制画笔不影响下一次画其他东西
            this.ctx.save() // 保存上次的画笔信息
            this.ctx.beginPath(); // 从新开始
            callback.call(this) // this改变this指向
            this.ctx.closePath(); // 关闭路径
            this.ctx.restore() // 重制画笔
        },
        drawBg(){ // 绘制背景图
            this.safeDraw(
                function(){
                    this.ctx.fillStyle = this.bgcolor  //设置填充颜色
                    this.ctx.arc(this.size/2, this.size/2, this.size/2, 0, Math.PI*2, true) //绘制圆形 。@params（圆心x,圆心y，起始弧度，结束弧度，是否是顺时针）
                    this.ctx.fill()//填充颜色
                }
            )
        },
        drawTick() {  // 绘制刻度   
            let length = 10 
            for(let i = 0; i < 60 ;i ++) {
                if (i%5==0) { //每五个一个长的刻度
                    length = 20
                } else {
                    length = 10
                }
                this.safeDraw(
                    function() {
                        this.ctx.strokeStyle = this.pointercolor // 设置描边的颜色
                        this.ctx.translate(this.size/2,this.size/2) // 将坐标轴的起点移动到圆心   @important  移动和旋转坐标轴原点需要提前
                        this.ctx.rotate(Math.PI/30*i)//然后依照弧度进行旋转
  
                         
                        this.ctx.moveTo(this.size/2-length,0)//画线先将起点移动到距边缘length的距离
                        this.ctx.lineTo(this.size/2,0)//再将length画过去
                        this.ctx.stroke()//进行描边
                    }
                )
            }
        },
        drawSecondPin() { // 绘制秒针
            let second =new Date().getSeconds()
            this.safeDraw(
                function() {
                    this.ctx.lineCap = this.linStyle
                    this.ctx.translate(this.size/2,this.size/2) // 
                    this.ctx.rotate(second*Math.PI/30-Math.PI/2)
                    this.ctx.moveTo(0,0)
                    this.ctx.lineTo(this.size/2-30,0)
                    this.ctx.strokeStyle=this.pointercolor
                    this.ctx.stroke()
                }
            )
        },
        tickTock(that) { // 让时间动起来
            setInterval(
                function() {
                    that.ctx.clearRect(0, 0, that.size, that.size);
                    that.drawBg()
                    that.drawTick()
                    that.drawSecondPin()
                    that.drawMinute()
                    that.drawHour()
                    that.drawNumTime()
                    // setTimeout(arguments.callee,1000)
                }, 1000
            )
        },
        drawMinute() {  // 绘制分针
            let minutes =new Date().getMinutes()
            this.safeDraw(
                function() {
                    this.ctx.lineCap = this.linStyle
                    this.ctx.lineWidth = this.minutePinWidth
                    this.ctx.translate(this.size/2,this.size/2)
                    this.ctx.rotate(minutes*Math.PI/30-Math.PI/2)
                    this.ctx.moveTo(0,0)
                    this.ctx.lineTo(this.size/2-50,0)
                    this.ctx.strokeStyle=this.pointercolor
                    this.ctx.stroke()
                }
            )
        },
        drawHour() { // 绘制时针
            let minutes = new Date().getMinutes()
            let hours = new Date().getHours()
            hoursNum = hours*60 + minutes
            this.safeDraw(
                function() {
                    this.ctx.lineCap = this.linStyle
                    this.ctx.lineWidth = this.hourPinWidth
                    this.ctx.translate(this.size/2,this.size/2) // 
                    this.ctx.rotate(hoursNum*Math.PI/360-Math.PI/2)
                    this.ctx.moveTo(0,0)
                    this.ctx.lineTo(this.size/2-100,0)
                    this.ctx.strokeStyle=this.pointercolor
                    this.ctx.stroke()
                }
            )
        },
        drawNumTime() { // 绘制时间 
            let time = new Date().toTimeString().split(' ')[0]
            this.ctx.font =  this.font;
            this.ctx.fillStyle = this.pointercolor
            let textWidth = this.ctx.measureText(time).width //获取字体宽度
            this.ctx.fillText(time, this.size/2-textWidth/2,this.size/2-40); // @params（文本，文本的位置x，文本的位置y）
        }
    }
    new drawClock('.clock',{})
</script>
</html>
```
 |

* Demo2用canvas实现一个仪表盘：


```
import './Gauge.less'


export default {

  name: 'Gauge',

  props: ['data'],

  data() {

    return {

      board: null,

      title: '',

    }

  },

  mounted() {

    if (this.data) { // 检测父元素有没有传过来数据

      let num = 0

      const item = this.data

      this.title = item.name

      let val = item.data.default[0].value

      num = parseFloat(val)  // 处理数据

      this.draw(num)

    }

  },

  methods: {

    draw(percent, sR) {  // 画仪表盘的逻辑

      if (sR < Math.PI / 2 || sR >= 3 / 2 * Math.PI) {

        return;

      };

      // 去除锯齿

      let canvas = document.querySelector('#gauge')

      const width = canvas.width,height=canvas.height;

      const cxt = canvas.getContext('2d')

      if (window.devicePixelRatio) {

        canvas.style.width = width + "px";

        canvas.style.height = height + "px";

        canvas.height = height * window.devicePixelRatio;

        canvas.width = width * window.devicePixelRatio;

        cxt.scale(window.devicePixelRatio, window.devicePixelRatio);

      };

      // 初始化配置项

      let cWidth = width;

      let cHeight = height;

      let baseColor = '#f2f3f6';

      let coverColor = '#2b73f9';

      let fontColor = '#78849A';

      let fontFamily = '36px DINCondensed-Bold';

      let ajustHeight = 20;

      let radius = 65;

      let lineWidth = 12;

      let PI = Math.PI;

      sR = sR || 7 / 8 * PI; // 默认圆弧的起始点弧度为7/8

      const finalRadian = sR + ((PI + (PI - sR) * 2) * percent / 100); // 小圆圈的终点弧度

      const step = (PI + (PI - sR) * 2) / 100; // 一个1%对应的弧度大小

      let text = 0; // 显示的数字

      let num = 0; // 仪表盘进度

      // 添加动画效果

      window.requestAnimationFrame(paint);

      function paint() { // 绘制图样

        cxt.clearRect(0, 0, cWidth, cHeight); // 每次绘制都先清空画布

        let endRadian = sR + num * step;

        // 画灰色圆弧

        drawCanvas(cWidth / 2, cHeight / 2 + ajustHeight, radius, sR, sR + (PI + (PI - sR) * 2), baseColor, lineWidth);

        // 画红色圆弧

        drawCanvas(cWidth / 2, cHeight / 2 + ajustHeight, radius, sR, endRadian, coverColor, lineWidth);

        // 小圆头

        // 小圆头其实就是一个圆，关键的是找到其圆心

        let angle = 2 * PI - endRadian; // 转换成逆时针方向的弧度（三角函数中的）

        let xPos = Math.cos(angle) * radius + cWidth / 2; // 小圆 圆心的x坐标

        let yPos = -Math.sin(angle) * radius + cHeight / 2 + ajustHeight; // 小圆 圆心的y坐标

        fillCanvas(xPos, yPos, 3, 0, 2 * PI, baseColor, 1);

        // 填充数字

        cxt.fillStyle = fontColor; // 初始化填充样式

        cxt.font = fontFamily;

        let textWidth = cxt.measureText(text + '%').width;

        cxt.fillText(text + '%', cWidth / 2 - textWidth / 2, cHeight / 2 + ajustHeight + lineWidth);

        num++;

        text++;

        if (endRadian.toFixed(2) <= finalRadian.toFixed(2)) {

          window.requestAnimationFrame(paint);

        }

        if (num >= percent) { //  小数和超出范围处理

          text = percent

          return

        }

        if (num >= 100) { //  小数和超出范围处理

          num = 100

          text = percent

          return

        }

        if (percent <= 0) { //  小数和超出范围处理

          num = 0

          text = percent

          return

        }

      }

      function drawCanvas(x, y, r, sRadian, eRadian, color, lineWidth) { // 描边轮廓绘图

        cxt.beginPath();

        cxt.lineCap = "round";

        cxt.strokeStyle = color;

        cxt.lineWidth = lineWidth;

        cxt.arc(x, y, r, sRadian, eRadian, false);

        cxt.stroke();

      }

      function fillCanvas(x, y, r, sRadian, eRadian, color, lineWidth) { // 填充内容绘图

        cxt.beginPath();

        cxt.lineCap = "round";

        cxt.fillStyle = color;

        cxt.lineWidth = lineWidth;

        cxt.arc(x, y, r, sRadian, eRadian, false);

        cxt.fill();

      }

    }

  },

  render() {

    return <div class="NewGuageBox">

      <div class="gauge_title">

        <span class="title" onClick={() => {

          this.HelpInfo(this.data['help'].content.default)

        }}>{this.title}<i class="iconfont icon-info"></i></span>

      </div>

      <div class="gauge_box">

        <canvas id="gauge" width="300" height="150"></canvas>

      </div>

    </div>;

  },

};

```
 |

demo3 用canvas实现这样的效果：

![image.png](https://upload-images.jianshu.io/upload_images/6958980-6337b071b5f1e72a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240) 



 ```
import Charts from './charts1'

render() {  // 父级元素的数据

    let config = [{

        class:'canvas',

        color:'#ea6e5a',

        value:'20',

        title:'20',

        label:'业务占比'

    },{

        class:'canvas1',

        color:'',

        value:'80',

        title:'80',

        label:'业务占比'

    },{

        class:'canvas2',

        color:'#c89efe',

        value:'90',

        title:'90',

        label:'上架量'

    }]

    return <div className="wrap">{

        config.map((item,index)=>{

            return  <Charts width='600' height='100' data={item} key={index}></Charts>

        })

    }

    </div>

}

``` 
图表组件：

```
/**

 * Created by guoguangkun on 2017/9/19.

 */

import React, { Component } from 'react'

import { bindActionCreators } from 'redux'

import { connect } from 'react-redux'

import { hashHistory, Link } from 'react-router'

import { Spin, message, Form, Icon, Input, Button, Row, Col } from 'antd'

import { fetchLogin, userInfo } from 'actions/common'

const FormItem = Form.Item

@connect((state, props) => ({

    config: state.config,

    loginResponse: state.tabListResult,

}))

@Form.create({

    onFieldsChange(props, items) {

        // console.log(items)

        // props.cacheSearch(items);

    },

})

export default class Charts extends Component {

    // 初始化页面常量 绑定事件方法

    constructor(props, context) {

        super(props)

        this.state = {

            loading: false,

        }

        this.draw = this.draw.bind(this)

    }

    componentDidMount() {

        console.log(this.props.data)

        this.draw(this.props.data.value,300, '.' + this.props.data.class,this.props.data.color,this.props.data.value,this.props.data.label) // 此处第三个参数，选测器，一定要加前边的 点，因为这个点导致没选择上元素

    }

    draw = (percent, boxSize, target, targetColor, title, label ) => { // 画图的逻辑

        // 去除锯齿

        let canvas =document.querySelector(target)

        const width = canvas.width,height=canvas.height;

        const cxt = canvas.getContext('2d')

        if (window.devicePixelRatio) {

            canvas.style.width = width + "px";

            canvas.style.height = height + "px";

            canvas.height = height * window.devicePixelRatio;

            canvas.width = width * window.devicePixelRatio;

            cxt.scale(window.devicePixelRatio, window.devicePixelRatio);

        };

        // 初始化配置项

        let start = 160;

        let cWidth = width;

        let cHeight = height;

        let maxLength = 400;

        let baseColor =  '#e2ecfa';

        let coverColor = targetColor || '#2b73f9';

        let fontColor = '#78849A';

        let fontFamily = '24px DINCondensed-Bold';

        let ajustHeight = 20;

        let radius = 65;

        let lineWidth = 12;

        const finalRadian = maxLength * percent / 100; // 小圆圈的终点弧度

        const step = maxLength / 100; // 一个1%对应的弧度大小

        let text = 0; // 显示的数字

        let num = 0; // 仪表盘进度

        let PI = Math.PI

        let titleText = title || '60%'

        let labelText = label || '人均带看'

        // 添加动画效果

        window.requestAnimationFrame(paint);

        function paint() { // 绘制图样

            cxt.clearRect(0, 0, cWidth, cHeight); // 每次绘制都先清空画布

            let endRadian = start + num * step;

            // 底色

            drawCanvas(maxLength + start, 30, baseColor, lineWidth,start);

            // 进度

            drawCanvas(endRadian, 30, coverColor, lineWidth,start);

            // 小圆头

            // 小圆头其实就是一个圆，关键的是找到其圆心

            fillCanvas(endRadian, 30, 3, 0, 2 * PI, baseColor, 1);

            // 填充数字

            drawFonts('0',start,60,'#3d4049')

            drawFonts('20%',start+20*step,60,'#3d4049')

            drawFonts('80%',start+80*step,60,'#3d4049')

            drawFonts('100%',start+100*step,60,'#3d4049')

            //绘制标题和数值

            drawFonts(titleText,start-60,40,'#3d4049',fontFamily)

            drawFonts(labelText,start-60,60,'#8e9db2')

            num++;

            text++;

            if (num <= percent) {

                window.requestAnimationFrame(paint);

            }

            if (num >= percent) { //  小数和超出范围处理

                text = percent

                return

            }

            if (num >= 100) { //  小数和超出范围处理

                num = 100

                text = percent

                return

            }

            if (percent <= 0) { //  小数和超出范围处理

                num = 0

                text = percent

                return

            }

        }

        function drawCanvas(x, y, color, lineWidth,start) { // 描边轮廓绘图

            cxt.beginPath();

            cxt.lineCap = "round";

            cxt.strokeStyle = color;

            cxt.lineWidth = lineWidth;

            cxt.moveTo(start, 30);

            cxt.lineTo(x,y)

            cxt.stroke();

        }

        function fillCanvas(x, y, r, sRadian, eRadian, color, lineWidth) { // 填充内容绘图

            cxt.beginPath();

            cxt.lineCap = "round";

            cxt.fillStyle = color;

            cxt.lineWidth = lineWidth;

            cxt.arc(x, y, r, sRadian, eRadian, false);

            cxt.fill();

        }

        function drawFonts(text,x,y,fontColor,fontFamily) { //绘制文字

            cxt.beginPath();

            cxt.fillStyle = fontColor || '#e6eaed'; // 初始化填充样式

            cxt.font = fontFamily || '16px sans-serif';

            cxt.textAlign = 'right' // 设置文字对齐方向

            let textWidth = cxt.measureText(text).width;

            cxt.fillText(text, x, y);

        }

    }

    handleClick = () => {

        console.log(1)

    }

    noop = () => false

    render() {

        return <div className="wrap" onClick={this.handleClick}>

              <canvas className={this.props.data.class}width='600' height='100'></canvas>

        </div>

    }

}
/**

 * Created by guoguangkun on 2017/9/19.   all rights reserved 

 */

```
