# superset简介
Superset是一款轻量级的BI工具，由Airbnb的数据部门开源。

# 开发部署
官方文档：http://superset.apache.org/


## 环境
superset： 0.36.0

windows10

拉取镜像需配置代理:http://f1361db2.m.daocloud.io
 
由于windows部署安装始终会有这个[报错](https://github.com/apache/incubator-superset/issues/9785)
最终用docker部署，但是开发不方便，所以最终还是用mac去做二次开发

### docker部署安装步骤

docker search superset
docker pull amancevice/superset
docker run -d -p 8088:8088 --name superset amancevice/superset:latest  --生成容器ID 60946ede506b
docker exec -it -u root 60946ede506b fabmanager create-admin --app superset
docker exec -it 60946ede506b superset db upgrade
docker exec -it 60946ede506b superset load_examples
docker exec -it 60946ede506b superset init
docker exec -it 60946ede506b superset runserver


### mac下部署步骤

- 启动后端 superset

pip install -r requirements-dev.txt -i 国内源url

python -m flask run
默认端口起的是 **5000**

- 启动前端 superset-frontend

npm install

npm run dev-server

默认端口是 **9000**


# 二次开发已实现的功能

##  v1.0
- 增加和具体用户相关联的权限访问，不同用户看到的数据不同
- 添加数值单位，在原有支持K的基础上增加中文 千,万,亿等
- 新增两张自定义图表，top Bar （排行榜条形图）和 Mix line Bar （混合柱状条形图）

## v2.0
- 可以自由配置上钻下卷的链条
- 图表下钻，支持多级下钻
- 图表上卷，支持多级上卷

# 源码改动处

## config.py

```
数据源配置 SQLALCHEMY_DATABASE_URI = 'mysql+pymysql://root:123456@192.168.16.47:3306/superset'

sql打印，SQLALCHEMY_ECHO = True

关闭CSRF验证 WTF_CSRF_ENABLED = False

调整查询LIMIT QUERY_SEARCH_LIMIT = 1000
```

## superset/views/core.py
```
新增接口/iframe/<token>/<dashboard_id>

新增页面 /templates/superset/dashboard_hf.html
```

## TopBar数据接口

```
新增Class TopBar继承BaseViz

重写query_obj 、 get_data方法

```

## TopBar图表组件
```
- /visualizations/TopBar/images 可选图表的缩略图
- .../ReactTopBar.js   
- .../TopBar.js
- .../TopBarChartPlugin.js
- .../transformProps.js
```
依赖关系 TopBar -> ReactTopBar -> TopBarChartPlugin -> transformProps

TopBar的演示图如下

![dd](img/topbar.gif)

## 中国地图
```
CountryMap.js  'ISO'替换成 'NL_NAME_1' ,
Line:110 'NAME_2'替换为NL_NAME_1  ,'NAME_1'替换为NL_NAME_1
将china.geojson文件中的'黑龙江省'和繁体字换一下位置。
在data.forEach(..)里修改colorMap[d.country_id]=..改为colorMap[d.country_id.replace('省')]=..
在const colorFn = d => {..}
修改为 colorFn = d=>{
var namearr = d.properties.NL_NAME_1.split('|')
colorMap[namearr]||'none'
}

```
修改后的地图演示
<br/>
![dd](img/countrymap.gif)

## 与新增TopBar类似，添加Mix Line Bar

演示如下
<br/>
![](img/mixlinebar.gif)

借助echarts已有的api，在control panel中新增了两个功能
- 堆积图 （配置同一个stack值）
- x轴缩放（解决分组过多时柱子细小看不清问题）


## 点击下钻、上卷
```
以下钻饼图为例，修改superset-ui 中的NVD3Vis.js
添加两个点击事件，
chart.pie.dispatch.on('elementClick',function(ele){clicked(ele,uid)})
这里clicked引用来自superset里写的代码
具体详见github

```
饼图演示
<br/>
![cc](img/pie.gif)

ps:添加图表点击事件，让tip消失。这里还没改过来。


# 图表使用实践

## 导入数据源、表

## 创建charts
### 折线图

## SQL lab编写SQL
### 保存Sql

## 添加图表到dashboard
### 支持markdown




# 增加自定义图表

## 看已有图表的代码

这里以WordCloud为例。主要涉及到的js有如下
/src/explore/controlPanels/WordCloud  这是控制层 ，里面定义了charts左侧的组件。

superset已有的组件整理如下：
- series 
- metric
- ...
太多了。。

组件的代码 罗列几个属性
- requiresTime 是否需要时间
- controlPanelSections 为左侧组件列表，label的值为组件名称，实现代码在 /src/explore/controls.jsx
(在ui-plugin里，则是对应NVD3Controls.tsx 或其他图表包下的Controls)
- controlOverrides 是你需要覆盖已有组件里的属性值
- sectionOverrides  看Dual Line里的这块内容，对

当然，如果现有组件不能满足需求，可以自定义组件，自定义组件的代码放到和其他组件一样的位置 /src/explore/controls.jsx



注册左侧组件层
setupPlugins.ts registerValue('top_bar',TopBar)
注册图表type，有顺序
VizTypeControl.jsx  DEFAULT_ORDER=[top_bar,....]




/visualizations 下新建一个文件夹取名自己的图表名称 TopBar
- TopBar.jsx  react组件，主要实现图表的样式，检测数据类型（prop-types）

- TOpBar.less 类似CSS

- ReactTopBar.js(或者是)

- TopBarChartPlugin.js 继承图表插件父类ChartPlugin，构造方法里有 metaData(super-ui库的ChartMetaData对象【 界面上的图表名称显示为"Top Bar"，描述等】)，loadChart为加载React组件TopBar
   ps：如果是在ui-plugin里，则是对应TopBar/index.js
   
- transformProps.js 这是转换类

加载顺序 ：
TopBarChartPlugin -> ReactTopBar -> transformProps -> TOpBar


# 技术栈学习了解

```tex
\star  react prop-types 检测类型
```


```tex
\star  nvd3，编写svg图形代码  
```
对于代码的修改最好先在线平台操作学习，改完即可知道哪里变了

[agular-nvd3](http://krispo.github.io/angular-nvd3/#/)  
[plnkr](http://plnkr.co/edit/lBKFld?p=preview&preview)   
[vd3文档](http://nvd3-community.github.io/nvd3/examples/documentation.html)


```tex
\star  对图表下钻开发, 需要对redux有所了解，需要知道redux的一些概念：
```
[B站学习视频](https://www.bilibili.com/video/BV1Kt411k7Xz?from=search&seid=16472628578011502750)
[demo学习](https://github.com/prs1022/reace-redux-first-app)
[根据文档创建demo](https://create-react-app.dev/docs/getting-started/)
- State  数据，就是一个对象。redux中的state是不能直接修改的，只能通过action来修改，相当于我们在单例中定义setter方法。
- Action  redux 将每一个更改动作描述为一个action，要更改state中的内容，你需要发送action。一个action是一个简单的对象，用来描述state发生了什么变更。
```
const INCREMENT = 'INCREMENT'
const incrementAction = {"type": INCREMENT, "count": 2}
上面这就是一个action，说白了就是一个对象，根据里面的type判断做什么操作。
```
- Reducer  数据state，指示action都有了那么就是实现了。reducer就是根据action来对state进行操作。
```
const calculate = (state: ReduxState = initData, action: Action ) => {
    switch (action.type) {
        case INCREMENT:
            return {num: state.num + action.count}
        case REDUCE:
            return {num: state.num - action.count}
        default:
            return state
    }
}

export {calculate}
通过reducer操作后返回一个新的state，比如这里根据action的type分别对state.num进行加减。
```
- Store  store就是整个项目保存数据的地方，并且只能有一个,可以理解为一个数据库。创建store就是把所有reducer给它。
```
import { createStore, combineReducers } from "redux";
import { calculate } from "./calculate";

// 全局你可以创建多个reducer 在这里统一在一起
const rootReducers = combineReducers({calculate})
// 全局就管理一个store
export const store = createStore(rootReducers)
```
- dispatch  store.dispatch()是组件发出action的唯一方法。
```
store.dispatch(incrementAction);
通过store调用incrementAction，那么就直接把store里的数据修改了。
```

通常是同步的，通过redux中间件 redux-thunk可以实现异步
,如果用了中间件，所有action都会有个dispatch参数

调用action的方式
store.dispatch(async function(dispatch){
	dispatch({
		type:'',
		payload:
	})//派发一个action
})

通过dispatch进行一个方法的派发

store里的dispatch方法关联到组件

通过connect连接组件和redux数据，传递state数据和dispatch方法

总的来说，redux像是一个全局状态树，可以使得代码更加清晰



```tex
\star 在react项目中使用Echarts
```

- npm安装echarts-for-react
npm install --save echarts-for-react
npm install echarts --save

- echarts体积巨大，使用模块化加载

```
import echarts from 'echarts/lib/echarts' //必须
import 'echarts/lib/component/tooltip'
import 'echarts/lib/component/grid'
import 'echarts/lib/chart/bar'
```

- 组件化开发的福音，react组件模块化加载

```
import { pieOption, barOption, lineOption, scatterOption, mapOption, radarOption, candlestickOption } from './optionConfig/options'
const PieReact = asyncComponent(() => import(/* webpackChunkName: "PieReact" */'./EchartsDemo/PieReact'))  //饼图组件
const BarReact = asyncComponent(() => import(/* webpackChunkName: "BarReact" */'./EchartsDemo/BarReact')) //柱状图组件
const LineReact = asyncComponent(() => import(/* webpackChunkName: "LineReact" */'./EchartsDemo/LineReact'))  //折线图组件
const ScatterReact = asyncComponent(() => import(/* webpackChunkName: "ScatterReact" */'./EchartsDemo/ScatterReact'))  //散点图组件
const MapReact = asyncComponent(() => import(/* webpackChunkName: "MapReact" */'./EchartsDemo/MapReact'))  //地图组件
const RadarReact = asyncComponent(() => import(/* webpackChunkName: "RadarReact" */'./EchartsDemo/RadarReact')) //雷达图组件
const CandlestickReact = asyncComponent(() => import(/* webpackChunkName: "CandlestickReact" */'./EchartsDemo/CandlestickReact')) //k线图组件
```

- echarts-for-react 是一个非常简单的针对于ReaCt的Echarts封装插件
用法：
import ReactEcharts from 'echarts-for-react'


