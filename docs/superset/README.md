# 背景
大数据时代商业智能（BI）和数据可视化诉求更为强烈，淘宝大屏更是风靡全球！数据可视化是大数据『最后一公里』。
<br>
数据可视化分析工具大致分为3类，开源、商业和传统bi工具。
业界目前比较流行的
<br>
开源BI工具：Superset、metabase、Redash、Cboard、d3js、Spagobi等，
<br>
商业BI工具：帆软、tableau、PowerBI、SmartBI、QlinkView、QuickBI、fineBI、阿里dataV等
<br>
传统BI工具：Congos、BIEE、BO、MicroStrategydeng等。


# superset简介
Superset是一款轻量级的BI工具，由Airbnb的数据部门开源。截止目前，github累计31.3K个star。


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
- 汉化支持

## v2.0
- 可以自由配置上钻下卷的链条
- 图表下钻，支持多级下钻
- 图表上卷，支持多级上卷
- 图表联动，在对一张图表配置了联动后，类似下钻的点击会影响到其他图表


## v3.0
- 给FilterBox新增级联属性，支持类似‘省份-区域-城市’这样的层级筛选
- 添加统一登录，接入ldap账号体系内部实现无差别登陆，默认为普通用户权限

## v4.0
- 添加集成多个dashboard的门户层，根据层级菜单将多个dashboard加进来
- 增加漏斗图(Funnel)

## v5.0
- 用antv改写了top bar,添加配色组件
- 在top bar中支持指定分组值用某个具体颜色，例如支付方式‘支付宝’蓝色，'微信'绿色
- 修复了top N在作用于多个分组的bug
- pivot table合计有误 bugfix
- 设置pivot table首行冻结


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

## 新增自定义图表

###  新增数据接口（以TopBar为例）

```
新增Class TopBar继承BaseViz

class TopBar(BaseViz):

    """DIY insert chart"""

    viz_type = "top_bar"
    verbose_name = _("top bar View")
    credits = 'a <a href="https://github.com/airbnb/superset">Superset</a> original'
    is_timeseries = False
    enforce_numerical_metrics = False

    def query_obj(self):
        d = super().query_obj()
        fd = self.form_data # form_data中包含界面左侧组件内容1
        if not fd.get('all_columns'): #这个字段对应×××组件，不为空
            raise Exception('Choose Columns')

        if fd.get('all_columns'):
            d['columns'] = fd.get('all_columns') # all_columns是左侧组件名，后面会提到

        ## 修改top bar 返回数据条数，以免数据过多前段卡死
        if fd.get('chooseN'):
            d["row_limit"] = fd.get("chooseN") * 100
        else :
            d["row_limit"]=100
        return d

    def get_data(self, df: pd.DataFrame) -> VizData:
        # df是pandas的DataFrame类型
        data = {'plot_data':np.array(df).tolist(),'legend':'数据的指标名称'}
        return data

    def json_dumps(self, obj, sort_keys=False):
        return json.dumps(
            obj, default=utils.json_iso_dttm_ser, sort_keys=sort_keys, ignore_nan=True
        )

```

### TopBar图表组件

<font size=5 color='red'>文件目录</font>

```
- /visualizations/TopBar/images 可选图表的缩略图
- .../ReactTopBar.js   
- .../TopBar.js
- .../TopBarChartPlugin.js
- .../transformProps.js
```
依赖关系 TopBar -> ReactTopBar -> TopBarChartPlugin -> transformProps
以下依次为各个文件的源码

<font size=5 color='red'> ReactTopBar.js </font>
```
import { reactify } from '@superset-ui/chart';
import Component from './TopBar';
 
export default reactify(Component);

```
---
<font size=5 color='red'> TopBar.js </font>
```
import echarts from 'echarts';
import d3 from 'd3';
import PropTypes from 'prop-types';
import { Chart } from '@antv/g2';
import { getColor, groupbysum } from '../MixLineBar/MixLineBarUtil';

const propTypes = {
  width: PropTypes.number,
  height: PropTypes.number,
}; // 检查类型，其中data包含viz.py中返回的数据，width和height为图表宽高
function TopBar(element, props) {
  const {
    chartData,
    chooseN,
    colorScheme,
    labelColors,
    yAxisFormat,
    yAxisHide,
    barStacked,
    xseries,
    isBreakDown,
    orderBars,
    showLegend,
    metricLabel,
    width,
    height,
  } = props;
  let yData = chartData.plot_data;
  const divid = 'top_bar_' + new Date().getTime();
  if (yData.length == 0) {
    // 没有数据的处理
    const div = d3.select(element, props);
    var html =
      "<div id='" +
      divid +
      "' style='height:" +
      height +
      'px; width:' +
      width +
      "px;text-align: center'>No Data</div>";
    html = '<div style="align-items: center; display: flex; flex-direction: column;height:100%; justify-content: center; padding: 16px; text-align: center; "><div style="font-size: 14px; opacity: 0.5; padding-bottom: 8px;">暂无数据</div><img src="/static/assets/images/nodata.png"></div>';
    div.html(html);
    return;
  }
  const index = yData[0].length - 1;
  function ascend(x, y) {
    return y[yData[0].length - 1] - x[yData[0].length - 1];
  }
  const antData = [];
  //排序以topN为主
  if (orderBars) {
    yData.sort(function([x, y], [x1, y1]) {
      return x.localeCompare(x1);
    });
  }
  if (chooseN != undefined) {
    //按照xserise分组求和取top
    yData = groupbysum(yData, xseries, chooseN);
  }
  // 记录最大刻度值
  const maxV = {};
  let maxValue = 0;
  for (let i = 0; i < yData.length; i++) {
    const ant_item = {};
    ant_item.type = '';
    if (maxV[yData[i][xseries]] == ``) {
      maxV[yData[i][xseries]] = 0;
    }
    for (let n = 0; n < yData[i].length; n++) {
      if (n == yData[i].length - 1) {
        // 最后一列为度量
        ant_item[metricLabel] = yData[i][n];
        if (barStacked) {
          maxV[yData[i][xseries]] = maxV[yData[i][xseries]] + yData[i][n];
        } else if (yData[i][n] > maxValue) {
          maxValue = yData[i][n];
        }
      } else {
        // 第 xseries 列为label
        ant_item.label = yData[i][xseries];
        if (isBreakDown) {
          // 其他列为分组
          ant_item.type = yData[i][1 - xseries];
        }
      }
    }
    antData.push(ant_item);
  }
  if (barStacked) {
    for (const k in maxV) {
      if (maxV[k] > maxValue) {
        maxValue = maxV[k];
      }
    }
  }
  const div = d3.select(element, props);
  var html =
    "<div id='" +
    divid +
    "' style='height:" +
    height +
    'px; width:' +
    width +
    "px;text-align: center'></div>";
  div.html(html); // 给echarts添加div
  // AntV实现
  const chart = new Chart({
    container: divid,
    autoFit: true,
    height: 500,
  });
  chart.data(antData);
  chart
    .coordinate()
    .transpose()
    .scale(1, -1);
  chart.scale(metricLabel, {
    min: 0,
    max: maxValue * 1.1,
  });
  chart.axis(metricLabel, {
    position: 'right',
  });
  chart.axis('label', {
    label: {
      offset: 12,
    },
  });
  chart.tooltip({
    shared: true,
    showMarkers: false,
  });
  let adjust_type = 'dodge';
  if (barStacked) {
    adjust_type = 'stack';
  }
  if (!showLegend) {
    chart.legend(false);
  }
  // 创建图形，由type 和 value两个属性决定图形x-y
  chart
    .interval()
    .position('label*' + metricLabel)
    .label(!yAxisHide ? metricLabel : '', {
      style: {
        fill: '#8d8d8d',
        fontSize: '14',
      },
      offset: 10,
    })
    .color({
      fields: [isBreakDown ? 'type' : metricLabel],
      values: isBreakDown ? getColor(colorScheme) : getColor(colorScheme)[0],
      callback:(val)=>{
        return labelColors[val]
      },
    })
    .adjust([
      {
        type: adjust_type,
        marginRatio: 0,
      },
    ]);
  if (yAxisFormat != null && yAxisFormat != 'SMART_NUMBER') {
    let yAxisFormat_ = yAxisFormat;
    if (yAxisFormat_ == 'zh') {
      yAxisFormat_ = '.0万亿';
    } else if (yAxisFormat_ == '.2zh') {
      yAxisFormat_ = '.2万亿';
    }
    let flag0 = yAxisFormat_.indexOf('+') >= 0;
    const flag1 = yAxisFormat_.indexOf(',') >= 0;
    const flag2 = yAxisFormat_.indexOf('.') >= 0;
    const flag3 = yAxisFormat_.indexOf('%') >= 0;
    const flag3_1 = yAxisFormat_.indexOf('万亿') >= 0;
    let suffix = '';
    // 数据格式
    chart.scale(metricLabel, {
      formatter(num) {
        let result = [];
        let counter = 0;
        if (num < 0) {
          flag0 = false;
        }
        if (flag3_1) {
          if (num.toString().split('.')[0].length > 8) {
            num /= 100000000;
            suffix = '亿';
          } else if (num.toString().split('.')[0].length > 4) {
            num /= 10000;
            suffix = '万';
          }
        }
        if (flag3) {
          // 开启百分比
          num *= 100;
        }
        num = (num || 0).toString().split('');
        result = num;
        let flag4 = num.indexOf('.');
        if (flag2) {
          // 保留小数点后几位
          const ii = yAxisFormat_[yAxisFormat_.indexOf('.') + 1];
          const a = parseFloat(num.join(''));
          const b = (
            Math.round(a * Math.pow(10, ii)) / Math.pow(10, ii)
          ).toFixed(ii);
          num = b.toString().split('');
          result = num;
        }
        flag4 = num.indexOf('.');
        if (flag1) {
          result = [];
          for (var i = flag4 - 1; i >= 0; i--) {
            counter++;
            result.unshift(num[i]);
            // 开启千分位
            if (!(counter % 3) && i != 0) {
              result.unshift(',');
            }
          }
          for (var i = flag4; i < num.length; i++) {
            result.push(num[i]);
          }
        }
        if (flag3) {
          result.push('%');
        }
        if (flag0) {
          result.unshift('+');
        }
        if (suffix != '') {
          result.push(suffix);
        }
        // result.unshift("¥"); //加后缀
        result = result.join('');
        return result;
      },
    });
  }
  chart.interaction('active-region');
  // 渲染图表
  chart.render();
}
TopBar.displayName = 'Top Bar';
TopBar.propTypes = propTypes;
export default TopBar;

```
---
<font size=5 color='red'> TopBarChartPlugin.js </font>
```
import { t } from '@superset-ui/translation';
import { ChartMetadata, ChartPlugin } from '@superset-ui/chart';
import transformProps from './transformProps';
import thumbnail from './images/thumbnail.png';
 
const metadata = new ChartMetadata({
  name: t('Top Bar'),
  description: 'diy',
  credits: ['http://echarts.baidu.com/examples/editor.html?c=scatter-effect'],
  thumbnail,
});
 
export default class TopBarChartPlugin extends ChartPlugin {
  constructor() {
    super({
      metadata,
      transformProps,
      loadChart: () => import('./ReactTopBar.js'),
    });
  }
}
```
---
<font size=5 color='red'>  transformProps.js </font>
```
export default function transformProps(chartProps) {
    const {width, height, queryData, formData} = chartProps;
    console.log(chartProps);

    var xseries_name = formData["groupby"];
    var breakdown_name = formData["columns"];
    var index1 = queryData.query.indexOf(xseries_name);
    var index2 = queryData.query.indexOf(breakdown_name);
    var xseries = 0;
    if (breakdown_name.length > 0) {
        xseries = 1;
        if (index1 < index2) {
            xseries = 0
        }
    }

    const isBreakDown = breakdown_name.length > 0;
    const metricLabel = formData.metrics[0].label;
    const chartData = queryData.data;

    const {
        chooseN,
        colorScheme,
        zoomData,
        yAxisFormat,
        yAxisHide,
        barStacked,
        showLegend,
        orderBars,
        labelColors,
    } = formData;

    return {
        chartData,
        chooseN,
        colorScheme,
        labelColors,
        zoomData,
        yAxisFormat,
        yAxisHide,
        barStacked,
        xseries,
        isBreakDown,
        orderBars,
        showLegend,
        metricLabel,
        width,
        height,
    };
}
```

<font size=5 color='red'> TopBar的演示图如下 </font>

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
<font size=5 color='red'>  地图演示 </font>

<br/>
![dd](img/countrymap.gif)

## 类似地，添加Mix Line Bar、Funnel

<font size=5 color='red'>  Mix Line Bar演示 </font>

![](img/mixlinebar.gif)


借助echarts已有的api，在control panel中新增了两个功能
- 堆积图 （配置同一个stack值）
- x轴缩放（解决分组过多时柱子细小看不清问题）

<font size=5 color='red'>  Funnel演示 </font>
![](img/funnel.png)


## 图表的下钻、上卷

以下钻饼图为例，修改superset-ui 中的NVD3Vis.js
添加两个点击事件，
```
chart.pie.dispatch.on('elementClick',function(ele){clicked(ele,uid)})
```
--- 
<font size=5 color='red'> xiazuan.js </font>
```
import {xiazuan} from "./reducer/xiazuanReducer";
import {shangjuan} from "./reducer/shangjuanReducer";

var ischart = true;
var chart_id = {};
export const clicked = function clicked(d,uid) {
    // var elearr = d.element.parentElement.parentElement.parentElement.parentElement.parentElement.parentElement.parentElement.parentElement.parentElement.getAttribute("id").split('-')
    // console.log(elearr)

    if(ischart){
        import('../explore/index').then(e=>{
            e.dashStore.dispatch(xiazuan(chart_id[uid],true,-1,d.data))
        })
    }else{
        import('../dashboard/index').then(e=>{
            e.dashStore.dispatch(xiazuan(chart_id[uid],true,-1,d.data))
        })
    }

};

export const clicked2 = function clicked2(uid) {

    if(ischart){
        import('../explore/index').then(e=>{
            e.dashStore.dispatch(shangjuan(chart_id[uid],true,-1))
        })
    }else{
        import('../dashboard/index').then(e=>{
            e.dashStore.dispatch(shangjuan(chart_id[uid],true,-1))
        })
    }
}

export function import_store (element,vizType,uid) {
    var viz_type=["pie","dist_bar"];
    var intype= viz_type.indexOf(vizType);
    if( intype< 0 ){
        return ;
    }
    var isTab=false;
    if(element.parentNode != null){
        isTab = true;
    }
    if(element.offsetParent !=null && element.offsetParent.className == "dashboard-component dashboard-component-chart-holder fade-out"){
        ischart = false
    }

    if(isTab){
        var elearr = element.parentNode.id.split('-')
        chart_id[uid] = elearr[elearr.length-1]
    }else {
        var elearr = element.offsetParent.getElementsByClassName(vizType)[0].getAttribute("id").split('-')
        chart_id[uid] = elearr[elearr.length-1]
    }

    console.log('chart_id->'+chart_id)
    return chart_id[uid]
}
```
---
<font size=5 color='red'> /action/xiazuanAction.js </font>
```
import {postChartFormData} from '../../chart/chartAction';

export const initLink = {};

export function xiazuanAction(chartKey, force, dashboardId, dData) {
    return (dispatch, getState) => {
        const charts = getState().charts;
        var chart = (getState().charts || {})[chartKey];
        var fd = chart.latestQueryFormData;

        if (!fd || Object.keys(fd).length === 0) {
            return;
        }
        console.log('xiazuan')
        if ((fd.xiazuan != null && fd.xiazuan.length > 0) || fd.liandong == true) {

            // 序列的下钻链路
            var curr = fd.groupby[0];
            var nextnode = getNext(curr, fd.xiazuan);
            console.log('nextnode=>' + nextnode);

            if (nextnode != null) {
                // 进行单个图表的下钻
                console.log('chartkey>>' + chartKey);
                if (typeof initLink[chartKey] === 'undefined') {
                    initLink[chartKey] = [];
                }
                initLink[chartKey].push(dData.x);
                fd.groupby.pop();
                fd.groupby.push(nextnode);
                fd.adhoc_filters.push(buildFilter(curr, dData.x));
                dispatch(postChartFormData(fd, force, 10000, chart.id, dashboardId));
            }

            if (fd.liandong) {
                // 如果有联动，触发其他同数据源图表的变化
                const dataSource = fd.datasource;
                const id = chart.id;
                for (const chartOne in charts) {
                    if (
                        charts[chartOne].formData.datasource === dataSource &&
                        charts[chartOne].id !== id
                    ) {
                        chart = charts[chartOne];
                        fd = chart.latestQueryFormData;

                        if (!fd || Object.keys(fd).length === 0) {
                            return;
                        }

                        console.log('触发序列联动')
                        if (fd.adhoc_filters.length > 0 && fd.adhoc_filters[fd.adhoc_filters.length - 1].subject == curr) {
                            //如果 点击的维度还是 最后联动追加的维度，则只改变值
                            fd.adhoc_filters[fd.adhoc_filters.length - 1].comparator = dData.x;
                        } else {
                            fd.adhoc_filters.push(buildFilter(curr, dData.x));
                        }

                        dispatch(postChartFormData(fd, force, 10000, chart.id, dashboardId));

                    }
                }
            }

        } else if ((fd.xiazuan2 != null && fd.xiazuan2.length > 0) || fd.liandong2 == true) {
            // 带有分解的下钻链路
            var curr2 = fd.columns[0];
            var nextnode2 = getNext(curr2, fd.xiazuan2);
            console.log('nextnode2=>' + nextnode2)

            if (nextnode2 != null) {
                console.log('chartkey>>' + chartKey);
                if (typeof initLink[chartKey] === 'undefined') {
                    initLink[chartKey] = [];
                }
                initLink[chartKey].push(dData.key);
                fd.columns.pop();
                fd.columns.push(nextnode2);
                fd.adhoc_filters.push(buildFilter(curr2, dData.key));
                dispatch(postChartFormData(fd, force, 10000, chart.id, dashboardId));
            } else if (!fd.liandong2) {
                return;
            }
            if (fd.liandong2) {
                const dataSource = fd.datasource;
                const id = chart.id;
                for (const chartOne in charts) {
                    if (
                        charts[chartOne].formData.datasource === dataSource &&
                        charts[chartOne].id !== id
                    ) {
                        chart = charts[chartOne];
                        fd = chart.latestQueryFormData;

                        if (!fd || Object.keys(fd).length === 0) {
                            return;
                        }

                        console.log('触发分解联动')
                        if (fd.adhoc_filters.length > 0 && fd.adhoc_filters[fd.adhoc_filters.length - 1].subject == curr2) {
                            //如果 点击的维度还是 最后联动追加的维度，则只改变值
                            fd.adhoc_filters[fd.adhoc_filters.length - 1].comparator = dData.key;
                        } else {
                            fd.adhoc_filters.push(buildFilter(curr2, dData.key));
                        }
                        dispatch(postChartFormData(fd, force, 10000, chart.id, dashboardId));

                    }
                }
            }

        }


    };
}

/**
 * 下钻衍生的过滤条件
 * @param name
 * @param val
 */
function buildFilter(name, val) {
    const adhoc_filter = {};
    adhoc_filter.clause = 'WHERE';
    adhoc_filter.comparator = val;
    adhoc_filter.expressionType = 'SIMPLE';
    adhoc_filter.fromFormData = true;
    adhoc_filter.isExtra = false;
    adhoc_filter.operator = '==';
    adhoc_filter.subject = name;
    return adhoc_filter;
}

/**
 * 获取链路的指定节点的下一节点
 * @param curr 当前节点
 * @param link 链路
 */
function getNext(curr, link) {
    const index = link.indexOf(curr);
    if (index == -1 || index == link.length - 1) {
        return null;
    }
    return link[index + 1];
}

```
---
<font size=5 color='red'>  /reducer/xiazuanReducer.js </font>
```
import * as actions from '../action/xiazuanAction';

export function xiazuan(chartList = 1, force = false, dashboardId,dData) {
  return (dispatch, getState) => {
    // eslint-disable-next-line no-undef
    dispatch(actions.xiazuanAction(chartList, force, dashboardId,dData));
  };
}
```

<font size=5 color='red'>  饼图演示 </font>

![cc](img/pie2.gif)

ps:下钻的时候tips没有及时消失，添加图表点击事件，让tip消失。


# 图表使用实践

## 导入数据源、表

添加数据源，支持多数据源，导入数据源后从已有的表中添加一张，可以设置cache，支持SQL修改

## 创建charts

选择一个数据源，选择一个图表类型

## SQL lab编写SQL

选择数据库、模式，直接写SQL模式发起查询

## 添加图表到dashboard

将制作好的图表添加到已有或新建的dashboard，完成看板制作

## 丰富dashboard

添加已有图表到dashboard中，自定义调整位置和大小；
支持图表联动配置；
支持过滤器指定图表变化；
支持markdown，可以添加视频；
支持修改css样式，夜间模式；


# 源码分析

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
- sectionOverrides  看Dual Line里的这块内容

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
 
- python flask框架
- 熟悉Flask—login组件

- 常用ES6 语法
[语法传送门](https://www.jianshu.com/p/c827598cd8b7)
- react语言熟悉,了解React的html与js混用的写法，了解组件，了解react的生命周期
[react入门](http://www.ruanyifeng.com/blog/2015/03/react.html)

- 摘出几句敲黑板的话，加深一下理解：
1、HTML 语言直接写在 JavaScript 语言之中，不加任何引号，这就是 JSX 的语法，它允许 HTML 与 JavaScript 的混写
2、React 允许将代码封装成组件（component），然后像插入普通 HTML 标签一样，在网页中插入这个组件。
3、组件并不是真实的 DOM 节点，而是存在于内存之中的一种数据结构，叫做虚拟 DOM （virtual DOM）只有当它插入文档以后，才会变成真实的 DOM 。根据 React 的设计，所有的 DOM 变动，都先在虚拟 DOM 上发生，然后再将实际发生变动的部分，反映在真实 DOM上，这种算法叫做 DOM diff ，它可以极大提高网页的性能表现。
4、



- react prop-types 检测类型


- nvd3，编写svg图形代码  

对于代码的修改最好先在线平台操作学习，改完即可知道哪里变了

[agular-nvd3](http://krispo.github.io/angular-nvd3/#/)  
[plnkr](http://plnkr.co/edit/lBKFld?p=preview&preview)   
[vd3文档](http://nvd3-community.github.io/nvd3/examples/documentation.html)



-  对图表下钻开发, 需要对redux有所了解，需要知道redux的一些概念：

[B站学习视频](https://www.bilibili.com/video/BV1Kt411k7Xz?from=search&seid=16472628578011502750)
[demo学习](https://github.com/prs1022/reace-redux-first-app)
[根据文档创建demo](https://create-react-app.dev/docs/getting-started/)
- State  数据，就是一个对象。redux中的state是不能直接修改的，只能通过action来修改，相当于我们在单例中定义setter方法。
- Action  redux 将每一个更改动作描述为一个action，要更改state中的内容，你需要发送action。一个action是一个简单的对象，用来描述state发生了什么变更。

```
const INCREMENT = 'INCREMENT'
const incrementAction = {"type": INCREMENT, "count": 2}
```
上面这就是一个action，说白了就是一个对象，根据里面的type判断做什么操作。

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
```
通过reducer操作后返回一个新的state，比如这里根据action的type分别对state.num进行加减。

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
```
通过store调用incrementAction，那么就直接把store里的数据修改了。

通常是同步的，通过redux中间件 redux-thunk可以实现异步
,如果用了中间件，所有action都会有个dispatch参数

调用action的方式
```
store.dispatch(async function(dispatch){
	dispatch({
		type:'',
		payload:
	})//派发一个action
})
```

通过dispatch进行一个方法的派发

store里的dispatch方法关联到组件

通过connect连接组件和redux数据，传递state数据和dispatch方法

总的来说，redux像是一个全局状态树，可以使得代码更加清晰



```text
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


