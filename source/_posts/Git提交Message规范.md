---
title: Git提交Message规范
date: 2019-03-21 08:11:22
tags:
 - Git
---

本文包含以下内容：

* 配置Git commitizen规范Git提交内容

<!-- more -->

#### 1. 为什么需要commitizen规范
在Git commit时，必要的message是用以描述本次提交的内容的，可以让团队成员清楚的知道本次提交的内容，也可以清晰的表现出不同的分支间的差异，甚至在版本回退时可以简洁明了的找出需要回退的版本，试想如果在版本回退的时候看到一大段糟心的 Commit，是不是会难受。
所以，不以规矩，不成方圆。

#### 3.公式
先来看看提交的公式

```
<type>(<scope>): <subject>
```
* type用以描述commit的类别
* scope用以描述影响的范围
* subject是本次提交的描述信息

#### 2.如何配置

1. brew install npm 安装npm

2. sudo npm install -g commitizen 全局安装commitizen

3. 项目中执行初始化 npm init 生成package.json

4. 项目中执行安装cz-customizable ， npm install cz-customizable --save-dev

5. 项目目录下创建自定义cz-config.js

    ```
    'use strict';
 
    module.exports = {
     
        types: [
            {value: 'feat', name: 'feat:     新功能'},
            {value: 'refactor', name: 'refactor: 代码重构'},
            {value: 'WIP', name: 'WIP:      暂时提交'},
            {value: 'fix', name: 'fix:      bug修复'},
            {value: 'perf', name: 'perf:     性能优化'},
            {value: 'docs', name: 'docs:     文档变更'},
            {value: 'style', name: 'style:    代码格式化'},
            {value: 'test', name: 'test:     添加测试用例'},
            {value: 'chore', name: 'chore:    构建过程或辅助工具的变动'},
            {value: 'revert', name: 'revert:   回滚'}
        ],
     
        scopes: [
            {name: 'api'},
            {name: 'service'},
            {name: 'dao'}
        ],
     
        // 比较适合db项目
        // allowTicketNumber: true,
        // isTicketNumberRequired: true,
        // ticketNumberPrefix: 'JIRA-',
        // ticketNumberRegExp: '\\d{1,5}',
     
        // it needs to match the value for field type. Eg.: 'fix'
        /*
         scopeOverrides: {
         fix: [
     
         {name: 'merge'},
         {name: 'style'},
         {name: 'e2eTest'},
         {name: 'unitTest'}
         ]
         },
         */
        // override the messages, defaults are as follows
        messages: {
            type: '选择当前的提交类型:',
            scope: '\n影响面:',
            // used if allowCustomScopes is true
            customScope: '自定义影响面:',
            subject: '简单描述这次提交:\n',
            body: '详细描述这次提交. 试用 "|" 换行:\n',
            breaking: '列举主要变化点(可选):\n',
            footer: 'List any ISSUES CLOSED by this change (optional). E.g.: #31, #34:\n',
            confirmCommit: '确认提交?'
        },
     
        allowCustomScopes: true,
        allowBreakingChanges: ['fix'],
        // skip any questions you want
        skipQuestions: ['body', 'footer'],
     
        // limit subject length
        subjectLimit: 100,
        // min subject length
        subjectMin:1
    };
    ```

6. 修改package.json中的配置信息

    ```
    "config": {
      "commitizen": {
        "path": "node_modules/cz-customizable"
      },
      "cz-customizable": {
        "config": "cz-config.js"
      }
    }
    ```

7. 替换node_modules/cz-customizable中questions.js

    ```
    'use strict';
     
    var fs = require('fs');
    var path = require('path');
    var buildCommit = require('./buildCommit');
    var log = require('./logger');
     
    var isNotWip = function(answers) {
      return answers.type.toLowerCase() !== 'wip';
    };
     
    var cwd = fs.realpathSync(process.cwd());
    var packageData = require(path.join(cwd, 'package.json'));
     
    function isValidateTicketNo(value, config) {
      if (!value) {
        return config.isTicketNumberRequired ? false : true;
      }
      if (!config.ticketNumberRegExp) {
        return true;
      }
      var reg = new RegExp(config.ticketNumberRegExp);
      if (value.replace(reg, '') !== '') {
        return false;
      }
      return true;
    }
     
    module.exports = {
     
      getQuestions: function(config, cz) {
     
        // normalize config optional options
        var scopeOverrides = config.scopeOverrides || {};
        var messages = config.messages || {};
        var skipQuestions = config.skipQuestions || [];
     
        messages.type = messages.type || 'Select the type of change that you\'re committing:';
        messages.scope = messages.scope || '\nDenote the SCOPE of this change (optional):';
        messages.customScope = messages.customScope || 'Denote the SCOPE of this change:';
        if (!messages.ticketNumber) {
          if (config.ticketNumberRegExp) {
            messages.ticketNumber = messages.ticketNumberPattern || 'Enter the ticket number following this pattern (' + config.ticketNumberRegExp + ')\n';
          } else {
            messages.ticketNumber = 'Enter the ticket number:\n';
          }
        }
        messages.subject = messages.subject || 'Write a SHORT, IMPERATIVE tense description of the change:\n';
        messages.body = messages.body || 'Provide a LONGER description of the change (optional). Use "|" to break new line:\n';
        messages.breaking = messages.breaking || 'List any BREAKING CHANGES (optional):\n';
        messages.footer = messages.footer || 'List any ISSUES CLOSED by this change (optional). E.g.: #31, #34:\n';
        messages.confirmCommit = messages.confirmCommit || 'Are you sure you want to proceed with the commit above?';
     
        var questions = [
          {
            type: 'list',
            name: 'type',
            message: messages.type,
            choices: config.types
          },
          {
            type: 'list',
            name: 'scope',
            message: messages.scope,
            choices: function(answers) {
              var scopes = [];
              if (scopeOverrides[answers.type]) {
                scopes = scopes.concat(scopeOverrides[answers.type]);
              } else {
                scopes = scopes.concat(config.scopes);
              }
              if (config.allowCustomScopes || scopes.length === 0) {
                scopes = scopes.concat([
                  new cz.Separator(),
                  { name: 'empty', value: false },
                  { name: 'custom', value: 'custom' }
                ]);
              }
              return scopes;
            },
            when: function(answers) {
              var hasScope = false;
              if (scopeOverrides[answers.type]) {
                hasScope = !!(scopeOverrides[answers.type].length > 0);
              } else {
                hasScope = !!(config.scopes && (config.scopes.length > 0));
              }
              if (!hasScope) {
                answers.scope = 'custom';
                return false;
              } else {
                return isNotWip(answers);
              }
            }
          },
          {
            type: 'input',
            name: 'scope',
            message: messages.customScope,
            when: function(answers) {
              return answers.scope === 'custom';
            }
          },
          {
            type: 'input',
            name: 'ticketNumber',
            message: messages.ticketNumber,
            when: function() {
              return !!config.allowTicketNumber; // no ticket numbers allowed unless specifed
            },
            validate: function (value) {
              return isValidateTicketNo(value, config);
            }
          },
          {
            type: 'input',
            name: 'subject',
            message: messages.subject,
            validate: function(value) {
              var limit = config.subjectLimit || 100;
              if (value.length > limit) {
                return 'Exceed limit: ' + limit;
              }
              const minLength = config.subjectMin || 0;
              if (value.length < minLength) {
                  return 'Less than minimum length:' + minLength;
              }
              return true;
            },
            filter: function(value) {
              return value.charAt(0).toLowerCase() + value.slice(1);
            }
          },
          {
            type: 'input',
            name: 'body',
            message: messages.body
          },
          {
            type: 'input',
            name: 'breaking',
            message: messages.breaking,
            when: function(answers) {
              if (config.allowBreakingChanges && config.allowBreakingChanges.indexOf(answers.type.toLowerCase()) >= 0) {
                return true;
              }
              return false; // no breaking changes allowed unless specifed
            }
          },
          {
            type: 'input',
            name: 'footer',
            message: messages.footer,
            when: isNotWip
          },
          {
            type: 'expand',
            name: 'confirmCommit',
            choices: [
              { key: 'y', name: 'Yes', value: 'yes' },
              { key: 'n', name: 'Abort commit', value: 'no' },
              { key: 'e', name: 'Edit message', value: 'edit' }
            ],
            message: function(answers) {
              var SEP = '###--------------------------------------------------------###';
              log.info('\n' + SEP + '\n' + buildCommit(answers, config) + '\n' + SEP + '\n');
              return messages.confirmCommit;
            }
          }
        ];
     
        questions = questions.filter(function(item) {
          return !skipQuestions.includes(item.name);
        });
     
        return questions;
      }
    };
    ```

8. 使用git cz取代git commit -m 进行commit message编辑；

