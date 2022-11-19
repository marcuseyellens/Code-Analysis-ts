[![npm version](https://badge.fury.io/js/code-analysis-ts.svg)](https://www.npmjs.com/package/code-analysis-ts)
[![Downloads](https://img.shields.io/npm/dm/code-analysis-ts.svg)](https://www.npmjs.com/package/code-analysis-ts)
# code-analysis-ts

[code-analysis-ts](https://www.npmjs.com/package/code-analysis-ts)是一款TS代码依赖扫描分析工具，可用于生成代码依赖分析报告，实现代码评分，代码告警，“脏”代码拦截等功能。支持CLI/API两种使用模式，通过npm script可以快速集成到业务CI自动化流程中。

## Install

```javascript
npm install code-analysis-ts --save-dev
// or
yarn add code-analysis-ts --dev    
```
## Config

新建 analysis.config.js 配置文件:
```javascript
const { execSync } = require('child_process');                        // 子进程操作
const DefaultBranch = 'master';                                       // 默认分支常量
function getGitBranch() {                                             // 获取当然分支
    try{
        const res = execSync('git branch');
        return res.toString("utf8").replace('*','').trim();
    }catch(e){
        return DefaultBranch;
    }
}

module.exports = {
    scanSource: [{                                                    // 必须，待扫描源码的配置信息
        name: 'Market',                                                    // 必填，项目名称
        path: ['src'],                                                     // 必填，需要扫描的文件路径（基准路径为配置文件所在路径）
        format: null,                                                      // 可选, 文件路径格式化函数,默认为null,一般不需要配置
        httpRepo: `https://gitlab.xxx.com/xxx/-/blob/${getGitBranch()}/`   // 可选，项目gitlab/github url的访问前缀，用于点击行信息跳转，不填则不跳转
    }],                                                                 
    analysisTarget: 'framework',                                      // 必须，要分析的目标依赖名
    analysisPlugins: [],                                              // 可选，自定义分析插件，默认为空数组，一般不需要配置
    blackList: ['app.localStorage.set'],                              // 可选，需要标记的黑名单api，默认为空数组
    browserApis: ['window','document','history','location'],          // 可选，要分析的BrowserApi，默认为空数组
    reportDir: 'report',                                              // 可选，生成代码分析报告的目录，默认为'report',不支持多级目录配置
    reportTitle: 'Market代码依赖分析报告',                               // 可选，分析报告标题，默认为'代码依赖分析报告'
    isScanVue: true,                                                  // 可选，是否要扫描分析vue中的ts代码，默认为false
    scorePlugin: 'default',                                           // 可选，评分插件: Function|'default'|null, default表示运行默认插件，默认为null表示不评分
    alarmThreshold: 90                                                // 可选，开启代码告警的阈值分数(0-100)，默认为null表示关闭告警逻辑 (CLI模式生效)
}
```
## Mode
### 1. cli

```javascript
// package.json 片段，添加bin command到npm script
...
"scripts": {
    "analysis": "ca analysis"
}
...

$ npm run analysis
// or
$ yarn analysis        
```
### 2. api

```javascript
const analysis = require('code-analysis-ts');

async function scan() {
    try{
        const { report, diagnosisInfos } = await analysis({
            scanSource: [{                                                    // 必须，待扫描源码的配置信息
                name: 'Market',                                                    // 必填，项目名称
                path: ['src'],                                                     // 必填，需要扫描的文件路径（基准路径为配置文件所在路径）
                format: null,                                                      // 可选, 文件路径格式化函数,默认为null,一般不需要配置
                httpRepo: `https://gitlab.xxx.com/xxx/-/blob/${xxx}/`              // 可选，项目gitlab/github url的访问前缀，用于点击行信息跳转，不填则不跳转
            }],                                                                 
            analysisTarget: 'framework',                                      // 必须，要分析的目标依赖名
            analysisPlugins: [],                                              // 可选，自定义分析插件，默认为空数组，一般不需要配置
            blackList: ['app.localStorage.set'],                              // 可选，需要标记的黑名单api，默认为空数组
            browserApis: ['window','document','history','location'],          // 可选，要分析的BrowserApi，默认为空数组
            reportDir: 'report',                                              // 可选，生成代码分析报告的目录，默认为'report',不支持多级目录配置
            reportTitle: 'Market代码依赖分析报告',                               // 可选，分析报告标题，默认为'代码依赖分析报告'
            isScanVue: true,                                                  // 可选，是否要扫描分析vue中的ts代码，默认为false
            scorePlugin: 'default',                                           // 可选，评分插件: Function|'default'|null, default表示运行默认插件，默认为null表示不评分
        });                                                                          
        console.log(report);
        console.log(diagnosisInfos);
    }catch(e){
        console.log(e);
    }
};

scan();
```
## Demo

[code-demo](https://github.com/liangxin199045/code-demo)演示如何使用code-analysis-ts的demo项目,使用github pages部署代码分析报告

## scorePlugin说明
配置文件中的scorePlugin配置项属于“函数插件”，使用者可以自定义代码评分插件来消费分析产物，评分插件需要对分析产物数据结构及属性有一定理解。下面是一个demo:
```javascript
// scorePlugin.js
// 评分插件
exports.myScoreDeal = function (analysisContext){
    // console.log(analysisContext);
    const { pluginsQueue, browserQueue, parseErrorInfos } = analysisContext;
    const mapNames = pluginsQueue.map(item=>item.mapName).concat(browserQueue.map(item=>item.mapName));
    
    let score = 100;            // 初始分数
    let message =[];            // 代码建议

    // 黑名单API扣分处理
    if(mapNames.length>0){
        mapNames.forEach((item)=>{
            Object.keys(analysisContext[item]).forEach((sitem)=>{
                if(analysisContext[item][sitem].isBlack){
                    score = score - 5;
                    message.push(sitem + ' 属于黑名单api，请勿使用');
                }
            })
        })
    }
    // 解析AST异常的扣分处理
    if(parseErrorInfos.length >0){
        score = score - 3*parseErrorInfos.length;
        let tempMessage ='';
        tempMessage = parseErrorInfos.length + ' 个文件解析&分析AST时发生错误，请修复';
        message.push(tempMessage);
    }

    // 最低0分
    if(score <0)score =0;

    return {
        score: score,
        message: message
    }
}

//analysis.config.js
const { myScoreDeal } = require('./scorePlugin.js');            // 自定义评分插件

module.exports = {
    ...
    scorePlugin: myScoreDeal,
    ...
}
```
## analysisPlugin说明
自定义分析插件，分析工具默认自带的插件有type分析，method分析，默认api分析三个插件，如果开发者有更多分析指标的诉求，可以开发特定分析插件(比如分析Class类型的api，分析用于三目运算符表达式中的api,分析导入再导出api等场景)，开发分析插件需要对源码和分析工具架构及生命周期有一定的理解。

## diagnosisInfos诊断日志说明
诊断日志是在代码分析过程中插件及关键节点产生的错误信息记录，可以帮助开发者调试自定义插件，快速定位代码文件，代码行，AST节点等相关错误信息。