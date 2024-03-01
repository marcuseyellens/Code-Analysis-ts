[![npm version](https://badge.fury.io/js/code-analysis-ts.svg)](https://www.npmjs.com/package/code-analysis-ts)
[![Downloads](https://img.shields.io/npm/dm/code-analysis-ts.svg)](https://www.npmjs.com/package/code-analysis-ts)
# code-analysis-ts

[code-analysis-ts](https://www.npmjs.com/package/code-analysis-ts) It is a front-end code analysis tool for the realization of code call analysis reports, code scoring, code alarms, "dirty call" interception, API trend analysis and other application scenarios. Support CLI/API two modes of usage and can be quickly integrated into the front-end engineering system , used to solve the front-end dependency governance of large-scale web applications .

## Install

```javascript
npm install code-analysis-ts --save-dev
// or
yarn add code-analysis-ts --dev    
```
## Config

create new analysis.config.js configuration file:
```javascript
const { execSync } = require('child_process');                        // subprocess operation
const DefaultBranch = 'master';                                       // default branch constants
function getGitBranch() {                                             // get current branch
    try{
        const branchName = execSync('git symbolic-ref --short -q HEAD', {
            encoding: 'utf8'
        }).trim();
        // console.log(branchName);
        return branchName;
    }catch(e){
        return DefaultBranch;
    }
}

module.exports = {
    scanSource: [{                                                    // Mandatory, configuration information for source code to be scanned
        name: 'Market',                                                    // Required, Project Name
        path: ['src'],                                                     // Required, the path of the file to be scanned (the base path is the path where the configuration file is located)
        packageFile: 'package.json',                                       // Optional, package.json file path configuration, used to collect dependent version information
        format: null,                                                      // Optional, file path formatting function, default is null, generally do not need to configure
        httpRepo: `https://gitlab.xxx.com/xxx/-/blob/${getGitBranch()}/`   // Optional, access prefix for project gitlab/github url, used for clicking on the line information to jump, if not fill in, then not jumping
    }],                                                                 
    analysisTarget: 'framework',                                      // Mandatory, the name of the target dependency to be analyzed
    analysisPlugins: [],                                              // Optional, custom analytics plugin, defaults to an empty array, generally no configuration required
    blackList: ['app.localStorage.set'],                              // Optional, blacklist api to be tagged, defaults to empty array
    browserApis: ['window','document','history','location'],          // Optional, the BrowserApi to analyze, defaults to an empty array
    reportDir: 'report',                                              // Optional, generate code analysis report directory, the default is 'report', does not support multi-level directory configuration
    reportTitle: 'Market Dependency Call Analysis Report',                               // Optional, title of the analysis report, default is 'Dependency Call Analysis Report'
    isScanVue: true,                                                  // Optional, whether to scan and analyze the ts code in vue, default is false
    scorePlugin: 'default',                                           // Optional, scoring plugin: Function|'default'|null, default means run default plugin, default is null means no scoring
    alarmThreshold: 90                                                // Optional, turn on the threshold score (0-100) for code alerts, the default is null to turn off the alerting logic (CLI mode is in effect)
}


```
## Mode
### 1. cli

```javascript
// package.json fragment, add the bin command to the npm script
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
const analysis = require('code-analysis-ts');                                   // Code Analysis Package
const { execSync } = require('child_process');                                  // subprocess operation
const DefaultBranch = 'master';                                                 // Default branch constants
function getGitBranch() {                                                       // Get current branch
    try{
        const branchName = execSync('git symbolic-ref --short -q HEAD', {
            encoding: 'utf8'
        }).trim();
        // console.log(branchName);
        return branchName;
    }catch(e){
        return DefaultBranch;
    }
}

async function scan() {
    try{
        const { report, diagnosisInfos } = await analysis({
            scanSource: [{                                                    // Mandatory, configuration information for source code to be scanned
                name: 'Market',                                                    // Required, Project Name
                path: ['src'],                                                     // Required, the path of the file to be scanned (the base path is the path where the configuration file is located)
                packageFile: 'package.json',                                       // Optional, package.json file path configuration, used to collect dependent version information
                format: null,                                                      // Optional, file path formatting function, default is null, generally do not need to configure
                httpRepo: `https://gitlab.xxx.com/xxx/-/blob/${getGitBranch()}/`   // Optional, access prefix for project gitlab/github url, used for clicking on the line information to jump, if not fill in, then not jumping
            }],                                                                 
            analysisTarget: 'framework',                                      // Mandatory, the name of the target dependency to be analyzed
            analysisPlugins: [],                                              // Optional, custom analytics plugin, defaults to an empty array, generally no configuration required
            blackList: ['app.localStorage.set'],                              // Optional, blacklist api to be tagged, defaults to empty array
            browserApis: ['window','document','history','location'],          // Optional, the BrowserApi to analyze, defaults to an empty array
            reportDir: 'report',                                              // Optional, generate code analysis report directory, the default is 'report', does not support multi-level directory configuration
            reportTitle: 'Market Dependency Call Analysis Report',                               // Optional, title of the analysis report, default is 'Dependency Call Analysis Report'
            isScanVue: true,                                                  // Optional, whether to scan and analyze the ts code in vue, default is false
            scorePlugin: 'default',                                           // Optional, scoring plugin: Function|'default'|null, default means run default plugin, default is null means no scoring
        });                                                                          
        // console.log(report);
        // console.log(diagnosisInfos);
    }catch(e){
        console.log(e);
    }
};

scan();
```
## Demo

[code-demo](https://github.com/liangxin199045/code-demo) demonstrate how to use code-analysis-ts demo project, use github pages to deploy code analysis reports

## scorePlugin's description
Configuration file in the scorePlugin configuration item belongs to the "function plugin", the user can customize the code score plugin to consume the analysis of the product, the score plugin needs to analyze the product of the data structure and attributes have a certain understanding. Here is a demo:
```javascript
// scorePlugin.js
// Rating Plugin
exports.myScoreDeal = function (analysisContext){
    // console.log(analysisContext);
    const { pluginsQueue, browserQueue, parseErrorInfos } = analysisContext;
    const mapNames = pluginsQueue.map(item=>item.mapName).concat(browserQueue.map(item=>item.mapName));
    
    let score = 100;            // initial score
    let message =[];            // Code Suggestion

    // Blacklisting API demerit points processing
    if(mapNames.length>0){
        mapNames.forEach((item)=>{
            Object.keys(analysisContext[item]).forEach((sitem)=>{
                if(analysisContext[item][sitem].isBlack){
                    score = score - 5;
                    message.push(sitem + ' Belongs to the blacklisted api, do not use it');
                }
            })
        })
    }
    // Analyzing the handling of demerit points for AST anomalies
    if(parseErrorInfos.length >0){
        score = score - 3*parseErrorInfos.length;
        let tempMessage ='';
        tempMessage = parseErrorInfos.length + ' error occurred while parsing & analyzing the AST of a file, please fix it!';
        message.push(tempMessage);
    }

    // Minimum 0 points
    if(score <0)score =0;

    return {
        score: score,
        message: message
    }
}

//analysis.config.js
const { myScoreDeal } = require('./scorePlugin.js');            // Custom Scoring Plugin

module.exports = {
    ...
    scorePlugin: myScoreDeal,
    ...
}
```
## analysisPlugin explanation
Custom analysis plug-ins, analysis tools built-in plug-ins type analysis, method analysis, default api analysis of the three plug-ins, if the developer has more analysis of the demand for indicators, you can develop a specific analysis plug-ins (such as the analysis of Class type of api, analysis of the api used in the expression of the three operators, analysis of the import and then export the api and other scenarios), the development of analysis plug-ins need to analyze the source code and analysis of the architecture of the tool and the lifecycle have a certain degree of understanding.

## custom library of plugins
[code-analysis-plugins](https://www.npmjs.com/package/code-analysis-plugins)It is a library of analysis plug-ins accompanying the analysis tool, which is used to share some commonly used indicator analysis plug-ins.

## diagnosisInfos's diagnostic log description
Diagnostic log is in the code analysis process plug-ins and key nodes generated by the error information records , can help developers debug custom plug-ins , quickly locate the code file , code lines , AST nodes and other related error information .

## vue_temp_ts_dir is what on the catalog?
If the configuration switch for scanning TS in Vue is turned on, the tool will extract the TS fragments in Vue for transit TS processing, which is a temp temporary directory that will be destroyed at the end of the analysis.