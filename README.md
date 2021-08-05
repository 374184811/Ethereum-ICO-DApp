13. 众筹DApp的部署
到目前为止，我们已经熟悉了众筹智能合约的开发、编译、部署、自动化测试，也熟悉了众筹 DApp 的框架、如何从智能合约读取数据、如何向智能合约提交数据，做出了可用的众筹 DApp。那么开发完成之后，怎么把这个 DApp 放出去给用户使用呢？这就涉及到了 DApp 的部署。
众筹 DApp 本质上是架构在 Next.js 之上的 WEB 应用，启动时依赖 Node.js 运行环境，我们可以把 Node.js 服务的部署方法迁移过来，比如服务进程管理、日志管理、配置管理。
13.1 配置管理
理论上，所有和代码逻辑无关的内容都应该放在配置文件里面，而不是直接硬编码写在源文件中，比如 web3 配置 provider时传入的主机名和端口。Node.js 应用中做配置管理，通常使用config，如果使用 config 改造我们的项目，要执行如下步骤。
1.	安装依赖
npm install --save config@1.30.0
2.	创建配置文件
在项目根目录下创建 config 目录，然后在其中创建 3 个文件：
mkdir config
touch config/default.js
touch config/development.js
touch config/production.js
3.	提取配置项
在众筹 DApp 中，我们可以把web3 provider的地址放到配置文件里面，对应的配置文件为：config/default.js
module.exports = {
providerUrl: 'http://localhost:8545'
};
config/development.js，可以覆盖 default 里面的配置，如果不需要，export 空对象即可：
module.exports = {};
config/production.js，可以覆盖 default 里面的配置，如果不需要，export 空对象即可，在设置了 NODE_ENV=production 时生效:
module.exports = {
    providerUrl: 'https://server.domain.com:8546'
};
需要特别说明的是 config 模块只适用于管理后端配置，那前端的配置怎么办呢？部分敏感的配置信息传给前端也是很大的安全隐患，所以我们可以利用Next.js 内置的配置暴露机制支持给前端和后端不同的配置，在项目根目录下创建 next.config.js，然后在其中输入如下代码：
const config = require('config'); 
module.exports = { 
// 只有后端可用的配置 
serverRuntimeConfig: {
   mySecret: 'secret'  // 不能暴露给前端的机密信息
}, 
// 前后端都可用的配置 
publicRuntimeConfig: { 
providerUrl: config.get('providerUrl')
}
};
4.	引用配置文件
要修改多少代码取决于我们把哪些内容放到了配置文件里，对于服务端，我们需要修改 scripts 下面的两个脚本：
deploy.js & sample.js
const web3 = new Web3(new 
	Web3.providers.HttpProvider('http://localhost:8545'));
const config = require('config');
const web3 = new Web3(new Web3.providers.HttpProvider(config.get('providerUrl')));
而前端代码中使用配置的方式是，在众筹 DApp 中只需要修改 libs/web3.js 即可：
import Web3 from 'web3';
import getConfig from 'next/config';
const { serverRuntimeConfig, publicRuntimeConfig } = getConfig();

let web3;

if(typeof window !== 'undefined' && typeof window.web3 !== 'undefined'){
    web3 = new Web3(window.web3.currentProvider);
} else {
    web3 = new Wb3(new Web3.providers.HttpProvider('http://localhost:8545'));
    web3 = new Wb3(new Web3.providers.HttpProvider(publicRuntimeConfig.providerUrl));
}

export default web3;
5.	回归测试
代码改动之后，重新启动服务，浏览页面，试用几个功能即可，保证一切正常。
13.2 日志管理
众筹 DApp 中暂时只需要记录服务进程的启动日志、运行时抛出的异常即可，日志目录创建好，可以提高项目的可移植性： 
	在根目录下创建 logs 目录； 
	在 logs 目录下创建 .gitkeep 空文件，并且提交该文件； 
	然后把这个目录加到 .gitignore 里面
13.3 服务进程管理
DApp 服务进程管理非 pm2莫属了，不管 DApp 部署在常规虚拟机、服务器还是在 docker 容器里面，都可以使用 pm2 来管理服务进程，使用 pm2 来管理服务进程的步骤如下：
1.	安装依赖
npm install --save-dev pm2
2.	添加配置文件
根目录下创建 pm2.json，输入如下代码：
{
    "apps": [
        {
            "name": "ico-dapp",
            "script": "./server.js",
            "out_file": "./logs/out.log",
            "error_file": "./logs/error.log",
            "log_date_format": "YYYY-MM-DD HH:mm:ss",
            "instances": 0,
            "exec_mode": "cluster",
            "max_memory_restart": "500M",
            "merge_logs": true,
            "env": {
                "NODE_ENV": "production"
            }
        }
    ]
}
3.	修改启动命令
直接替换 package.json 中原有的 start 命令如下：
"start": "NODE_ENV=production node server.js"
"start": "pm2 restart pm2.json"
这里我们使用了 pm2 restart，而不是 pm2 start，是为了兼容之前已经启动过服务部署新版本时的情况。
13.4 无情自动化
把工作流中的各个环节使用自动化的命令串起来，能提高效率，还能减少过多人工操作导致的失误，具体到智能合约 + DApp 项目中，合约部署时，需要重新编译，需要跑通所有的测试；DApp 部署时需要部署最新的代码，并且代码是构建过的。用流程化的语言来描述如下：
	合约编译 --> 合约自动化测试 --> 合约部署
	DApp 构建 --> DApp 部署
使用 npm script 可以方便的实现上面的两条主线：合约部署、DApp 部署。修改 package.json 如下：
"scripts": {
    "compile": "node scripts/compile.js",
    "pretest": "npm run compile",
    "test": "./node_modules/mocha/bin/mocha tests/",
    "predeploy": "npm run compile",
    "predeploy": "npm run test",
    "deploy": "node scripts/deploy.js",
    "dev": "node server.js",
    "build": "next build",
    "prestart": "npm run build",
    "start": "NODE_ENV=production node server.js"
  },
这样，之后我们做部署工作时，只要记住两条命令就可以了：
	如果要部署智能合约，执行：npm run deploy
	如果要部署 DApp，执行：npm run start
