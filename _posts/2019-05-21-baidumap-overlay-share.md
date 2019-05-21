---
layout: post
title: 百度地区几何区域定位，距离计算
categories: BaiduMap
description: 如何在百度地图绘制几何区域，并检测用户是否在该集合区域内，计算两点之间的距离
keywords: 百度地图, 几何, 距离计算, 定位
---

# 百度地图多边形区域及定位分享

### 1、前言

公司小程序项目需求：

> 1）在地图中标注城市中的服务区块，且为几何区域，并需要检测用户是否在该几何区域内，若在区域内才可以进行添加地址并下单；
>
> 2）工人进行服务时，需要首先拍照上传，但上传位置不得超出管理员在后台所设置的距离。

因为开发习惯，所以考虑直接使用百度地图的API。

### 2、几何区域标注

#### 第一次构建

标注几何区域必定会在地图中生成一个遮盖层，在百度地图中遮盖物包括但不仅限于`圆形`、`椭圆`、`折线`、`弧线`、`几何`这几种，但是要保留遮盖物就必须要存储遮盖物的位置，而通过查阅[示例demo](http://lbsyun.baidu.com/jsdemo.htm#c2_9)发现几何区域由其每个角的经纬度度坐标构成

![几何区域]({{ site.url }}/assets/images/posts/201905/21-001-c2_p.png)

因此只需要在地图上标注多个散列点即可

![多点标注]({{ site.url }}/assets/images/posts/201905/21-001-c1_3.png)

但是对于用户来说非常的不直观，而且也不知道标注顺序，那就来个实时预览好了，不过实时预览需要3个及以上的点才可以构成一个块，最后所得如下：

![实时预览]({{ site.url }}/assets/images/posts/201905/21-001-1003.gif)

终于是完成了，附上代码：

```javascript
// 初始化地图
function init_map()
{
  map = new BMap.Map("map");
  // 选择城市后 设置地图中心点
  map.centerAndZoom(city_name, 12);
  // 鼠标滑轮缩放
  map.enableScrollWheelZoom(true);
  // 添加缩放控件
  map.addControl(new BMap.NavigationControl());
  // 单击获取点击的经纬度
  map.addEventListener("click", function (e) {
    let lat = e.point.lat;
    let lng = e.point.lng;
    // 添加标注
    let point = new BMap.Point(lng, lat);
    let marker = new BMap.Marker(point);
    map.addOverlay(marker);
    // 储存经纬度
    point_arr.push({lng:lng, lat:lat});
    createDraw();
  });
  createDraw();
}

// 绘制几何区域
function createDraw() {
  let ply;
  // 标注大于3个时
  if (point_arr.length >= 3) {
    // 清楚地图上所有的遮盖物
    map.clearOverlays();
    let pts = [];
    for (var i in point_arr) {
      // 给第一个点添加标注 起点位置
      if (i == '0') {
        // 给开始的点添加标注
        let point = new BMap.Point(point_arr[i]['lng'], point_arr[i]['lat']);
        let marker = new BMap.Marker(point);
        map.addOverlay(marker);
      }
      pts.push(new BMap.Point(point_arr[i]['lng'], point_arr[i]['lat']));
    }
    // 展示几何区域
    map.addOverlay(new BMap.Polygon(pts));
  }
}
```

不过不过开心的有点早了，最后一个点标注的时候有点飘了，当我清除所有遮盖物时重新标记或者重新编辑时，飘的更厉害了，鼠标点击位置与地图上所标注位置相差巨大，但是如果关闭绘制集合区域，单点标注是没有问题的，具体原由，暂未找到。

#### 第二次构建

若是按照上面的构建方法，肯定是不行的，继续找出路。经过各种搜索，其实在百度`javascript api`的示例demo下面有个[开源库](http://lbsyun.baidu.com/index.php?title=jspopular/openlibrary)，提供了很多lib库，其中包括`鼠标绘制工具条库`

![鼠标绘制工具条库]({{ site.url }}/assets/images/posts/201905/21-001-1004.png)

可以根据鼠标绘制遮层直接实施显示遮层，而不需要向第一次构建那样傻傻的刷新，好在过程还是一样一样的，只需要替换预览部分即可

![实时预览]({{ site.url }}/assets/images/posts/201905/21-001-1005.gif)

Perfect~~ 改进后的代码

```javascript
function init_map()
{
  map = new BMap.Map("map");
  // 地图中心点
  map.centerAndZoom(city_name, 12);
  // 缩放
  map.enableScrollWheelZoom(true);
  // 添加缩放空间
  map.addControl(new BMap.NavigationControl());
  // 初始化遮盖区域
  createDraw();
  // 初始化绘制工具
  initDrawTool();
}

// 初始化区域绘制
function createDraw() {
  if (point_arr.length >= 3) {
    map.clearOverlays();
    let pts = [];
    for (var i in point_arr) {
      pts.push(new BMap.Point(point_arr[i]['lng'], point_arr[i]['lat']));
    }
    map.addOverlay(new BMap.Polygon(pts));
  }
}

// 初始化绘制工具
function initDrawTool() {
  //实例化鼠标绘制工具
  let drawingManager = new BMapLib.DrawingManager(map, {
    enableDrawingTool: true, //是否显示工具栏
    drawingToolOptions: {
      anchor: BMAP_ANCHOR_BOTTOM_RIGHT, //位置
      offset: new BMap.Size(5, 5), //偏离值
      scale: 0.7, //工具栏缩放比例
      drawingModes: [
        BMAP_DRAWING_POLYGON
      ]// 绘制模型 几何区域
    },
    /*polygonOptions: {
      strokeColor: "red", //边线颜色。
      fillColor: "red", //填充颜色。当参数为空时，圆形将没有填充效果。
      strokeWeight: 2, //边线的宽度，以像素为单位。
      strokeOpacity: 0.8, //边线透明度，取值范围0 - 1。
      fillOpacity: 0.6, //填充的透明度，取值范围0 - 1。
      strokeStyle: 'solid' //边线的样式，solid或dashed。
    }, //多边形的样式*/
  });
  
	// 绘制完成的监听事件
  drawingManager.addEventListener('overlaycomplete', function(e) {
    points = [];
    var path = e.overlay.getPath();
    path.forEach(function(value, index) {
      // 获取依次点击的经纬坐标
      points.push({lng:value.lng, lat:value.lat});
    });
  });
}
```

### 3、检测是否在区域内

#### 第一次构建

由于在前面发现了[开源库](http://lbsyun.baidu.com/index.php?title=jspopular/openlibrary)，翻看了下各个库的内容，百度地图提供了一个`几何运算`(小开心 >_<)

> GeoUtils类提供若干几何算法，用来帮助用户判断点与矩形、 圆形、多边形线、多边形面的关系,并提供计算折线长度和多边形的面积的公式。 主入口类是GeoUtils。

翻看了[demo演示](http://api.map.baidu.com/library/GeoUtils/1.2/examples/simple.html)，直接查看[源文件](http://api.map.baidu.com/library/GeoUtils/1.2/src/GeoUtils.js)，其部分代码如下：

```javascript
/**
 * 判断点是否多边形内
 * @param {Point} point 点对象
 * @param {Polyline} polygon 多边形对象
 * @returns {Boolean} 点在多边形内返回true,否则返回false
 */
GeoUtils.isPointInPolygon = function(point, polygon){
  //检查类型
  if(!(point instanceof BMap.Point) ||
     !(polygon instanceof BMap.Polygon)){
    return false;
  }

  //首先判断点是否在多边形的外包矩形内，如果在，则进一步判断，否则返回false
  var polygonBounds = polygon.getBounds();
  if(!this.isPointInRect(point, polygonBounds)){
    return false;
  }

  var pts = polygon.getPath();//获取多边形点

  //下述代码来源：http://paulbourke.net/geometry/insidepoly/，进行了部分修改
  //基本思想是利用射线法，计算射线与多边形各边的交点，如果是偶数，则点在多边形外，否则
  //在多边形内。还会考虑一些特殊情况，如点在多边形顶点上，点在多边形边上等特殊情况。

  var N = pts.length;
  var boundOrVertex = true; //如果点位于多边形的顶点或边上，也算做点在多边形内，直接返回true
  var intersectCount = 0;//cross points count of x 
  var precision = 2e-10; //浮点类型计算时候与0比较时候的容差
  var p1, p2;//neighbour bound vertices
  var p = point; //测试点

  p1 = pts[0];//left vertex        
  for(var i = 1; i <= N; ++i){//check all rays            
    if(p.equals(p1)){
      return boundOrVertex;//p is an vertex
    }

    p2 = pts[i % N];//right vertex            
    if(p.lat < Math.min(p1.lat, p2.lat) || p.lat > Math.max(p1.lat, p2.lat)){//ray is outside of our interests                
      p1 = p2; 
      continue;//next ray left point
    }

    if(p.lat > Math.min(p1.lat, p2.lat) && p.lat < Math.max(p1.lat, p2.lat)){//ray is crossing over by the algorithm (common part of)
      if(p.lng <= Math.max(p1.lng, p2.lng)){//x is before of ray                    
        if(p1.lat == p2.lat && p.lng >= Math.min(p1.lng, p2.lng)){//overlies on a horizontal ray
          return boundOrVertex;
        }

        if(p1.lng == p2.lng){//ray is vertical                        
          if(p1.lng == p.lng){//overlies on a vertical ray
            return boundOrVertex;
          }else{//before ray
            ++intersectCount;
          } 
        }else{//cross point on the left side                        
          var xinters = (p.lat - p1.lat) * (p2.lng - p1.lng) / (p2.lat - p1.lat) + p1.lng;//cross point of lng                        
          if(Math.abs(p.lng - xinters) < precision){//overlies on a ray
            return boundOrVertex;
          }

          if(p.lng < xinters){//before ray
            ++intersectCount;
          } 
        }
      }
    }else{//special case when ray is crossing through the vertex                
      if(p.lat == p2.lat && p.lng <= p2.lng){//p crossing over p2                    
        var p3 = pts[(i+1) % N]; //next vertex                    
        if(p.lat >= Math.min(p1.lat, p3.lat) && p.lat <= Math.max(p1.lat, p3.lat)){//p.lat lies between p1.lat & p3.lat
          ++intersectCount;
        }else{
          intersectCount += 2;
        }
      }
    }            
    p1 = p2;//next ray left point
  }

  if(intersectCount % 2 == 0){//偶数在多边形外
    return false;
  } else { //奇数在多边形内
    return true;
  }            
}
```

一串串的英文，看不懂？没关系，嗯？谁说没关系的，你再仔细瞅瞅，注释中有详细中文说明

> //下述代码来源：http://paulbourke.net/geometry/insidepoly/，进行了部分修改
> //基本思想是利用射线法，计算射线与多边形各边的交点，如果是偶数，则点在多边形外，否则
> //在多边形内。还会考虑一些特殊情况，如点在多边形顶点上，点在多边形边上等特殊情况。

但是本项目是小程序，不能直使用js，还是调用外部的js就更加不可能了，咋办，只能放服务端执行了，但是有个缺点，在服务端执行js，并且js中需要再次发送一个结果的异步请求，因为js得到结果后无法存储在php的变量中，然后客户端还需要再次请求一次拿取结果

```php
/**
 * 检测用户当前是否在区域内
 * @param string $area 区域数据
 * @param string $lat
 * @param string $lng
 */
public function checkInArea($area = '', $lat = '112.911101', $lng = '28.18926')
{
  //        $s = '[{"long":"112.92966","lat":"28.199241"},{"long":"112.920605","lat":"28.202679"},{"long":"112.900339","lat":"28.204016"},{"long":"112.893296","lat":"28.192428"},{"long":"112.898399","lat":"28.179374"},{"long":"112.919096","lat":"28.174343"},{"long":"112.930307","lat":"28.182112"}]';
  // 请求地址
  $url = url('setInPolygon');
  // 获取ys_token 用于接口访问
  $time = time();
  $ys_token = create_ys_token($this->uid, $time);
  // 执行js判断用户当前位置是否在区域内
  echo '<script type="text/javascript" src="http://api.map.baidu.com/api?v=1.2"></script>';
  echo '<script type="text/javascript" src="http://api.map.baidu.com/library/GeoUtils/1.2/src/GeoUtils_min.js"></script>';
  echo '<script src="http://libs.baidu.com/jquery/2.1.4/jquery.min.js"></script>';
  echo "<script>
            var area = $area;
            var pts = [];
            // 生成几何区域
            for (var i in area) {
                pts.push(new BMap.Point(area[i]['long'], area[i]['lat']))
            }
            var ply = new BMap.Polygon(pts);
            // 我的位置
            var pt =new BMap.Point($lat, $lng);
            // 计算是否在区域内
            var result = BMapLib.GeoUtils.isPointInPolygon(pt, ply);
            // 异步推送结果
            $.ajax({
                url: '$url',
                data:{in_city:result,uid:$this->uid,timestamp:$time,ys_token:'$ys_token'},
                success:function(){}
            });
        </script>";
}

/**
 * 设置是否在区域内
 * @return array|\think\response\Json|\think\response\Jsonp|\think\response\Xml
 */
public function setInPolygon()
{
  $in_city = request()->param('in_city');
  cache('uid-' . $this->uid, $in_city);
  return apiOut('success', 0);
}

/**
 * 获取是否在区域内
 * @return array|\think\response\Json|\think\response\Jsonp|\think\response\Xml
 */
public function getInPolygon()
{
  return apiOut('success', 0, ['in_city' => cache('uid-' . $this->uid)]);
}
```

#### 第二次构建

虽然流程是可以了，但是这个骚操作也太灾难了，既然知道是射线法，那为什么不自己在后台写个呢，还有js源可以参考，再次翻看了几遍[源文件](http://api.map.baidu.com/library/GeoUtils/1.2/src/GeoUtils.js)，好像有点不对，开头是判断百度地图需要用到的类型，而且这些类型只是得到坐标以及是否符合其类型的要求，后面的计算方式跟百度地图可是完全没关系的，被开头给迷惑了，怎么这么笨呢。。。去掉不需要的，直接转php就完事了,好气啊😤

```php
if (!function_exists('isPointInPolygon')) {
    /**
     * 判断点是否在多边形内
     * 参考：http://api.map.baidu.com/library/GeoUtils/1.2/src/GeoUtils.js
     * @param array $point 当前位置经纬度 ['lng' => '', 'lat' => '']
     * @param array $pts   多边形坐标集
     * @return bool 为真则在区域内
     */
    function isPointInPolygon($point, $pts)
    {
        if (empty($point) || empty($pts)) {
            return false;
        }
        $N = count($pts);
        $boundOrVertex = true; //如果点位于多边形的顶点或边上，也算做点在多边形内，直接返回true
        $intersectCount = 0;//cross points count of x
        $precision = 2e-10; //浮点类型计算时候与0比较时候的容差
        $p1 = $p2 = [];// neighbour bound vertices
        $p = $point; //测试点

        $p1 = $pts[0];//left vertex
        for ($i = 1; $i <= $N; ++$i) {//check all rays
            if ($p['lng'] == $p1['lng'] && $p['lat'] == $p1['lat']) {
                return $boundOrVertex;//p is an vertex
            }
            $p2 = $pts[$i % $N];//right vertex
            if ($p['lat'] < min($p1['lat'], $p2['lat']) || $p['lat'] > max($p1['lat'], $p2['lat'])) {//ray is outside of our interests
                $p1 = $p2;
                continue;//next ray left point
            }
            if ($p['lat'] > min($p1['lat'], $p2['lat']) && $p['lat'] < max($p1['lat'], $p2['lat'])) {//ray is crossing over by the algorithm (common part of)
                if ($p['lng'] <= max($p1['lng'], $p2['lng'])) {//x is before of ray
                    if ($p1['lat'] == $p2['lat'] && $p['lng'] >= min($p1['lng'], $p2['lng'])) {//overlies on a horizontal ray
                        return $boundOrVertex;
                    }
                    if ($p1['lng'] == $p2['lng']) {//ray is vertical
                        if ($p1['lng'] == $p['lng']) {//overlies on a vertical ray
                            return $boundOrVertex;
                        } else {//before ray
                            ++$intersectCount;
                        }
                    } else { //cross point on the left side
                        $xinters = ($p['lat'] - $p1['lat']) * ($p2['lng'] - $p1['lng']) / ($p2['lat'] - $p1['lat']) + $p1['lng'];//cross point of lng
                        if (abs($p['lng'] - $xinters) < $precision) {//overlies on a ray
                            return $boundOrVertex;
                        }
                        if ($p['lng'] < $xinters) {//before ray
                            ++$intersectCount;
                        }
                    }
                }
            } else {// special case when ray is crossing through the vertex
                if ($p['lat'] == $p2['lat'] && $p['lng'] <= $p2['lng']) {//p crossing over p2
                    $p3 = $pts[($i + 1) % $N]; //next vertex
                    if ($p['lat'] >= min($p1['lat'], $p3['lat']) && $p['lat'] <= max($p1['lat'], $p3['lat'])) {// $p['lat'] lies between $p1['lat'] & $p3['lat']
                        ++$intersectCount;
                    } else {
                        $intersectCount += 2;
                    }
                }
            }
            $p1 = $p2;//next ray left point
        }

        if ($intersectCount % 2 == 0) {//偶数在多边形外
            return false;
        } else { //奇数在多边形内
            return true;
        }
    }
}
```

23333~ 很嗨皮了有么有，经过测试，结果一致，棒~👍

### 4、位置距离计算

翻越很多山，又遇一个坑 = = ☠️ 小程序中使用的定位是腾讯地图，而腾讯地图经纬度与百度地图的经纬度不一致，无法直接使用，因此需要在小程序端将腾讯坐标转换为国测局左边，再在服务通过百度接口转换为百度坐标

> [坐标系小知识点我查看](https://blog.csdn.net/m0_37738114/article/details/80452485)

距离计算在[几何运算源文件](http://api.map.baidu.com/library/GeoUtils/1.2/src/GeoUtils.js)也有示例，有了前车之鉴，直接拿来转换

```javascript
if (!function_exists('getDistance')) {
    /**
     * 获取两点间的距离 - 百度地图坐标
     * 参考：http://api.map.baidu.com/library/GeoUtils/1.2/src/GeoUtils.js
     * @param array $point1 坐标一 ['lng' => '', 'lat' => '']
     * @param array $point2 坐标二 ['lng' => '', 'lat' => '']
     * @return float|int 返回距离 单位：米
     */
    function getDistance($point1 = [], $point2 = [])
    {
        // 地球半径
        $earth_radius = 6370996.81;
        $point1['lng'] = getLoop($point1['lng'], -180, 180);
        $point1['lat'] = getRange($point1['lat'], -74, 74);
        $point2['lng'] = getLoop($point2['lng'], -180, 180);
        $point2['lat'] = getRange($point2['lat'], -74, 74);

        // 转换弧度
        $x1 = $x2 = $y1 = $y2 = '';
        $x1 = pi() * $point1['lng'] / 180;
        $y1 = pi() * $point1['lat'] / 180;
        $x2 = pi() * $point2['lng'] / 180;
        $y2 = pi() * $point2['lat'] / 180;

        $distance = $earth_radius * acos(sin($y1) * sin($y2) + cos($y1) * cos($y2) * cos($x2 - $x1));

        return $distance;
    }
}
if (!function_exists('getLoop')) {
    /**
     * 将v值限定在a,b之间，经度使用
     * 参考：http://api.map.baidu.com/library/GeoUtils/1.2/src/GeoUtils.js
     */
    function getLoop($v, $a, $b)
    {
        while( $v > $b){
            $v -= $b - $a;
        }
        while($v < $a){
            $v += $b - $a;
        }
        return $v;
    }
}

if (!function_exists('getRange')) {
    /**
     * 将v值限定在a,b之间，纬度使用
     * 参考：http://api.map.baidu.com/library/GeoUtils/1.2/src/GeoUtils.js
     */
    function getRange($v, $a, $b)
    {
        if($a != null){
            $v = max($v, $a);
        }
        if($b != null){
            $v = min($v, $b);
        }
        return $v;
    }
}
```

### 5、初始化中心位置错误

初始化加载百度地图时，所设置的中心不在正中央，而是在在左上角原点(0, 0)位置

![地图中心位置不正确]({{ site.url }}/assets/images/posts/201905/21-001-1006.png)

参考网文，是因为页面加载后，地图为手动加载，而手动加载地图时，容器的宽度及高度为0，所以中央位置在左上角原点(0, 0)位置

解决方法：先加载容器，百度地图初始化延迟加载即可。

```javascript
$('#myModal').modal();
setTimeout(function () {
	init_map();
}, 100)
```

