# Bmap
最近使用百度地图api遇到的一些问题和总结
最近在项目中使用到百度地图，说实话百度地图个人不建议再使用了，高德地图可能更好一些，但是大致的使用方法和相关API都是差不多的，只是百度地图会有一些bug，这篇笔记仅供参考

#百度地图使用方法
在项目中怎样引入百度地图我就不详写了，首先你要生成一个自己的key，在引入百度地图的时候以参数的形式拼接上去，这样才能在项目文件中new出map事例，并在事例上进行操作

#我涉及到的地方
首先先简单介绍一下项目内容和我涉及到的需求：我涉及的地图工具主要是查看在不同地点公司广告屏的投放情况，在一级地图上（比如在地点定位到北京，地图上显示的是北京的海淀区，朝阳区等地点广告瓶的数量，此时地图处于以及地图）显示某个城市不同区的广告瓶投放情况，点击相应的区（如朝阳区）将会进入二级地图，此时显示详细地点广告屏的投放情况；再次点击二级地图的点会弹出详情页。由于这个h5的手机端页面也要嵌入到小程序的webview当中，为了和小程序列表页同步，所以需求是：1.在地图上方添加城市筛选tab以及地点筛选搜索框，其中筛选搜索框要实现关键字联想，并在点击下拉框中指定地点之后显示对应地点广告屏的投放情况 ；2.在地图的左下角添加定位按钮，点击之后进行定位并显示相应位置n公里范围内的广告屏投放情况。废话不多说，下面我开始讲解我的做法和遇到的坑以及解决办法

#开始编写
1.地点筛选搜索框及联想搜索
dom上的结构是一个搜索框input来输入关键字，一个div来盛放关键字联想出来的地点json数据,需要注意的是，这个div只负责盛放数据，并不负责展示数据，dom代码如下
                <div class="inputBox">
                    <i class="el el-icon-search"></i>
                    <input class="search-input" id="suggestId" v-model="searchInput" placeholder-style="color:#999" placeholder="输入地点名称查询"/>
                    <div id="searchResultPanel" style="border:1px solid #C0C0C0;width:150px;height:auto; display:none;z-index:9999999"></div>
                </div>
搜索功能相关js代码如下所示，需要注意的是，地图搜索是基于你当前map对象的，所以要在地图init的里面进行调用，在mounted或者created的时候调用地图的init方法即可

                let G = (id) => {
                    return document.getElementById(id);
                }
                let ac = new BMap.Autocomplete(    //建立一个自动完成的对象
                    {"input" : "suggestId"
                    ,"location" : this.map
                });
                ac.addEventListener("onhighlight", (e) => {  //鼠标放在下拉列表上的事件
                    let str = "";
                    let _value = e.fromitem.value;
                    let value = "";
                    if (e.fromitem.index > -1) {
                        value = _value.province +  _value.city +  _value.district +  _value.street +  _value.business;
                    }
                    str = "FromItem<br />index = " + e.fromitem.index + "<br />value = " + value;

                    value = "";
                    if (e.toitem.index > -1) {
                        _value = e.toitem.value;
                        value = _value.province +  _value.city +  _value.district +  _value.street +  _value.business;
                    }
                    str += "<br />ToItem<br />index = " + e.toitem.index + "<br />value = " + value;
                    G("searchResultPanel").innerHTML = str;
                });
                ac.addEventListener("onconfirm", (e) => {    //鼠标点击下拉列表后的事件
                    let _value = e.item.value;
                    this.searchInput = _value.province +  _value.city +  _value.district +  _value.street +  _value.business;
                    G("searchResultPanel").innerHTML ="onconfirm<br />index = " + e.item.index + "<br />myValue = " + this.searchInput;
                    console.log(G("searchResultPanel").innerHTML)
                    this.serachAction=1
                    this.setPlace();
                });

                let setPlace = ()=>{
                    this.mapLevel =2
                    this.map.clearOverlays();   //清除地图上所有覆盖物
                    let myFun = ()=>{
                        var pp = local.getResults().getPoi(0).point;    //获取第一个智能搜索的结果
                        this.map.centerAndZoom(pp, 16);
                        this.map.addOverlay(new BMap.Marker(pp));    //添加标注
                    }
                    let local = new BMap.LocalSearch(this.map, { //智能搜索
                    onSearchComplete: myFun
                    });
                    console.log(this.searchInput)
                    local.search(this.searchInput);
                }

可以看到，在上面的代码中涉及到很多原生的操作，在百度地图的事例中，原生的操作会更多，我们最需要改的就是将es5的函数形式改为es6的函数赋值和箭头函数的形式，这样可以省去不少this指针的只想问题，其他的我们也就是吧var改成let啥的，其他的我现在图方便我也不深做研究，只说思路。其中绑定的suggestId为我们搜索的input框的id，监听的事件主要为联想的触发事件以及点击某个地点出发的搜索事件，setPlace()函数中执行了我们点击下拉列表中的某个地点执行搜索的过程，这个函数我们暂且也放在init()中，之后的需求当中我们将会有改动。

说到这里我们其实已经将搜索框的所有需求做完了，你会发现我只做了搜索并没有请求在搜索位置上的广告屏投放的数据，刚开始我也以为没做完，但是效果已经出来了这是为什么，我注意到在地图的init函数中监听了地图移动的事件，代码如下

                // zoom 变化时候
                this.map.addEventListener('zoomend', e => {
                    let cur_zoom = this.map.getZoom()
                    if (cur_zoom < l1_TO_l2_ZOOM) {
                        // 清楚所有 覆盖物
                        this.map.clearOverlays()
                        this.getMarkerAndSetLevel1(this.cityCache)
                    } else {
                        this.map.clearOverlays()
                        // this.getMarkerAndSetLevel2()
                        setTimeout(() => {
                            this.getLevel2Data()
                        },20)
                    }
                })
                // 移动地图时候
                this.map.addEventListener('moveend', e => {
                    this.showScrNum = true
                    this.showDetail = false
                    this.showCityDetail = false
                    this.showPointDetail = false
                    let cur_zoom = this.map.getZoom()
                    if (cur_zoom < l1_TO_l2_ZOOM) {
                        // 暂时没有操作
                    } else {
                        setTimeout(() => {
                            this.showLoading = true
                            this.getLevel2Data()
                        },20)
                    }
                })

这两段函数中分别监听了地图移动和中心点移动的事件，在这个事件发生的时候this.getLevel2Data()函数都会触发，我们选择地点的时候地图位置发生变化，也就调用了this.getLevel2Data()函数，而这个函数就是调用获取二级数据（指定地点广告平投放情况数据）的接口。这里我多说一点，如果没有监听这两个事件，我们获取二级数据都要传哪些值，我们只说有关地图的值，也就是选择地点的时候获取的值，我们不难想到就是有段地点的经纬度的值，具体的值和获取方法如下

                    minLat: this.map.getBounds().getSouthWest().lat,
                    minLng: this.map.getBounds().getSouthWest().lng,
                    maxLat: this.map.getBounds().getNorthEast().lat,
                    maxLng: this.map.getBounds().getNorthEast().lng,
                
这四个值代表着以定位点为中心四个角的值，它覆盖了一个范围，我们将获取这个范围内的投放广告屏数量，搜索框我们就先说到这

2.GPS定位功能
这个功能说起来就有点坑了，这就是百度地图的一个bug，如果正常的话，点击定位这个功能只是百度地图的一个控件，我们只需要一行代码即可完成，代码如下：

this.map.addControl(new BMap.GeolocationControl())

完事了，看起来都没问题，地图左下角出现定位按钮，点击定位按钮浏览器开始提示拉取定位，点击同意地图开始进行定位。一切看起来都是这么的顺利。但是后来我发现它定的位置一直是天安门，为了确定这个问题，我特地请教了我的老大，他也发现了这个问题，当时我也是懵逼的。老大给了我建议，这也是我想到的，就是利用h5获取经纬度进行定位。说的容易，但是参数到底该怎么传，在哪里获取经纬度，获取到的经纬度怎样用对于我这个第一次正经接触地图操作的人来说还是挺懵逼的。首先我们不能在登录进入地图工具的时候就获取位置信息，这样会非常怪，因为我们那个时候还没有进行定位操作，所以我们要先点击定位再获取经纬度。下面的问题，就是怎样为地图添加一个‘定位的按钮’，地图并不是普通的dom，我们不能直接写个button啥的，看了api之后，我发现了一个好东西，就是自定义控件，上面给出了一个示例，这个控件的功能是点击按钮之后地图放大两级，代码如下

// 定义一个控件类，即function    
function ZoomControl(){    
    // 设置默认停靠位置和偏移量  
    this.defaultAnchor = BMAP_ANCHOR_TOP_LEFT;    
    this.defaultOffset = new BMap.Size(10, 10);    
}  

// 通过JavaScript的prototype属性继承于BMap.Control   
ZoomControl.prototype = new BMap.Control();

// 自定义控件必须实现initialize方法，并且将控件的DOM元素返回   
// 在本方法中创建个div元素作为控件的容器，并将其添加到地图容器中   
ZoomControl.prototype.initialize = function(map){    
    // 创建一个DOM元素   
    var div = document.createElement("div");    
    // 添加文字说明    
    div.appendChild(document.createTextNode("放大2级"));    
    // 设置样式    
    div.style.cursor = "pointer";    
    div.style.border = "1px solid gray";    
    div.style.backgroundColor = "white";    
    // 绑定事件，点击一次放大两级    
    div.onclick = function(e){  
        map.zoomTo(map.getZoom() + 2);    
    }    
    // 添加DOM元素到地图中   
    map.getContainer().appendChild(div);    
    // 将DOM元素返回  
    return div;    
 }// 创建控件实例  

var myZoomCtrl = new ZoomControl();    
// 添加到地图当中    
map.addControl(myZoomCtrl);

代码可以说是非常清晰了，在原型上定义一个新的构造函数，用来构造一个我们想要的样子的自定义控件，在构造函数中，我们动态创建控件dom，并为控件添加样式，为空间添加事件处理，返回控件实例，并添加到地图上。当然了，代码风格惨不忍睹，这个我们可以稍加改动我就不多说了，但是思路已经给我们了。现在我们要做的就是和上面一样构造一个自定义控件，为控件添加样式，重点部分是：为控件添加点击事件，点击的时候获取经纬度，并根据百度地图定位的api函数经纬度定位的方式传入经纬度。虽然有个事情很low，但是我还是说一下，关于定位控件的样式，你可以看百度地图自带的定位控件在控制台里面拷贝他的样式，算是哥小技巧吧，人家做的毕竟比你自己做好看。下面我粘出主要的定位代码供大家参考

                // 定位功能
                // 自定义控件
                function ZoomControl(){
                    // 设置默认停靠位置和偏移量
                    this.defaultAnchor = BMAP_ANCHOR_BOTTOM_LEFT;
                    this.defaultOffset = new BMap.Size(10, 10);
                }
                // 通过JavaScript的prototype属性继承于BMap.Control
                ZoomControl.prototype = new BMap.Control();
                ZoomControl.prototype.initialize = (map)=>{
                    // 创建一个DOM元素
                    let div = document.createElement("div");
                    // 设置样式
                    div.className = "gps"
                    // 添加DOM元素到地图中
                    this.map.getContainer().appendChild(div);
                    // 绑定事件，点击定位
                    div.onclick = (e)=>{
                        this.getGpsPosition = 1
                        this.mapLevel = 2
                        if(navigator.geolocation){
                            navigator.geolocation.getCurrentPosition((position)=>{
                            let lat = position.coords.latitude
                            let lng = position.coords.longitude
                            this.map.clearOverlays()
                            let newPoint = new BMap.Point(lng,lat)
                            let marker = new BMap.Marker(newPoint);  // 创建标注
                            this.map.addOverlay(marker);              // 将标注添加到地图中
                            this.map.panTo(newPoint);
                            this.map.centerAndZoom(newPoint, 16);
                          })
                       }
                    }
                    // div.onclick = this.getGpsPosition()

                    // 将DOM元素返回
                    return div;
                }
                // 创建控件实例
                let myZoomCtrl = new ZoomControl();
                // 添加到地图当中
                this.map.addControl(myZoomCtrl);

这就没什么好说的了，需要注意的就是，我们要先生成dom，在示例上，经纬度会成为dom生成的两个参数，我们要先生成dom，点击之后获取经纬度，然后再进行定位操作，这样就会避免调还没定位就申请拉取位置权限的尴尬局面