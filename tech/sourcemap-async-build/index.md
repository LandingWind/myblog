# source map异步构建


# 需求背景
大型Webpack项目构建时，加入source map构建严重影响构建速度，而且容易导致运行内存不足进而导致构建失败
source map简单了解 [https://segmentfault.com/a/1190000020213957](https://segmentfault.com/a/1190000020213957)


# 目前方案
划分为两次构建，第一次构建时不加入source map构建选项，可延后的第二次构建中进行环境相同的构建，使用构建产物中的source map文件
## 缺陷 和 风险


- 为保证两次构建的产物对应，我们需要采用完全相同的环境去构建



- 【重点】为了明确两次构建产物的联系，我们需要通过构建产物相同的Hash

![截屏2020-06-29 14.10.49.png](https://intranetproxy.alipay.com/skylark/lark/0/2020/png/300858/1593413397744-f7d6be58-1e06-433e-a7f6-06ba2f887379.png#align=left&display=inline&height=545&margin=%5Bobject%20Object%5D&name=%E6%88%AA%E5%B1%8F2020-06-29%2014.10.49.png&originHeight=1090&originWidth=1202&size=289712&status=done&style=shadow&width=601)

   - 影响Hash的环境因素和构建参数很多 可参考[分析文档](https://www.notion.so/createHash-f6ba5634b09146b7b7c0e1d5f56d3421)（根据webpack4-createHash源码得到）



   - 【Bug&Fix】关于一直以来的两次构建Hash不一致问题，我终于排查到是因为antd包的一个时间戳key，引发ModuleId不一致，进而引发ChunkId不一致（相关[FixPR](https://github.com/cnpm/npminstall/pull/329/files)）

![截屏2020-06-29 16.54.44.png](https://intranetproxy.alipay.com/skylark/lark/0/2020/png/300858/1593420912075-f4067e71-990e-4ad9-989a-cf850fcd3c79.png#align=left&display=inline&height=191&margin=%5Bobject%20Object%5D&name=%E6%88%AA%E5%B1%8F2020-06-29%2016.54.44.png&originHeight=382&originWidth=1556&size=102623&status=done&style=none&width=778)

   - 引发Hash不一致的本质问题没有解决，我们难以管控中间所有的非确定性因子，今天antd pro的Ellipsis构建用到了timestamp，明天语雀的加密可能就用到了md5+时间戳，后天说不定哪个产品构建用起了随机数...不可控是关键




- 消耗不必要的计算资源：两次构建的第二次构建，webpack打包流程中，uglify之前包括uglify本身都是无意义的，因为我们需要的只是source map文件



- source map构建过程中的排查成本和维护成本高
   - 排查成本：source map出现问题时，一般是对应不上无法使用，由于构建环境不一致，难以排查问题
   - 维护成本：如果需要更新修复的sourcemap，不能直接更新，因为无法比对验证其正确性（除非有专人肉眼识别），只能再跑两次构建



- source map的配置能力弱：罗列一下我觉得需要的配置
   - 配置什么chunk asset需要做source map
   - 配置不同的chunk asset做不同的source map策略，比如关键文件需要知道行信息和列信息，一般文件只需要知道行信息



- 没有cache能力：Hash一致的chunk asset不必要重复生成source map文件，而应该去集中的**source map cache**中调取



# 解决方案
## 临时方案
临时方案不一定是简单的方案，但一定是缺陷相对多的方案。在debug的过程中，我大致有以下几种解决问题的临时方案思路


### 监控不确定性因子
构建过程中non-determinate的因子是引发HashId改变的关键，那最直观的解决方法就是去除这些不确定因子。


#### 相关技术
最直观直接调用webpack提供的[Javascript Parser Hooks](https://webpack.js.org/api/parser/)
```javascript
// 解析到 assigned 时判断是否为不确定性因子
a += b;
parser.hooks.assigned.for('a').tap('MyPlugin', expression => {
  if(isDeterminate(expression.split(notation))) // notation: 表达式中间的一些符号
  {
    // do something
  } else {
   	// do something 
  }
});
// 判断是否确定性
function isDeterminate(val) {
	// 通过正则表达式判断是否携带 时间戳
}
// 类似这样写几个hooks...
```


另外或者从AST树介入去查找，效率低但会更可靠
> ### program
> `SyncBailHook`
> Get access to the abstract syntax tree (AST) of a code fragment
> Parameters: `ast` `comments`



还有可以从[@babel-parser](https://github.com/babel/babylon)中介入，传入解析时候的配置参数
bebel-parser还是提供了许多的个性化的解析能力 参考options源码 [https://github.com/babel/babel/blob/master/packages/babel-parser/src/options.js](https://github.com/babel/babel/blob/master/packages/babel-parser/src/options.js)
#### 
#### 有什么问题？

- 时间戳还比较好检测，随机数怎么判断呢
- 其他可能潜在的不确定因子怎么办
- 那么多ast node对值做一个正则判断，计算资源和时间效率我想是不可以接受的
- 检测到了不确定因子修改为确定值，是否会影响业务逻辑？（加密参数的话直接崩了）又是否会影响source map列坐标的准确性？本末颠倒了




---

### 比对更改产物Hash
在debug hash不一致的难捱时间里，我写过强制更改产物Hash的插件[webpack-hashCorrect-plugin](https://code.alipay.com/mike.wjk/webpack-hashCorrect-plugin)


#### 主要思路
写了一个通过AST的tokens比对JavaScript代码一致性的[jscode-compare](https://github.com/wkk5194/jscode-compare)模块，通过这一个模块对比两次构建产物是否一致，如果一致，则直接修改对应chunk asset source map的Hash保证相同；如果不一致则抛出错误
比较的内容是什么？参考生成的ast.json [https://github.com/wkk5194/jscode-compare/blob/master/test/demo-ast.json#L668](https://github.com/wkk5194/jscode-compare/blob/master/test/demo-ast.json#L668)


#### 有什么问题？

- AST仅仅比对tokens列表是有局限性的，严格来讲无法保证jscode完全的一致性，保证的是逻辑的一致性，所以对source map的准确性有影响




---

### 注入自己管控的createHash流程
参看webpack的createHash源码 [https://github.com/webpack/webpack/blob/master/lib/Compilation.js#L2404](https://github.com/webpack/webpack/blob/master/lib/Compilation.js#L2404)
构建产物加hash的目的是cache，bigfish用的一般是chunkhash或者contenthash
上面抛了hash的影响因子偏多，如果不强调hash依赖于模块调用关系的话，我们可以通过构建一套自己的createHash流程保证一致


#### 有什么问题？

- 没有模块依赖的hash变动了，单纯依赖产物内容
- 自己的hash生成规则，怎么制定，感觉十分麻烦
- 侵入webpack createHash比较困难，需要考虑webpack本身的版本迭代，毕竟不是公司自己的模块




---

## 可控方案
**
**大纲思想：**两次构建 => 一次构建
临时方案大多只能解决一时的问题，或者无法全部克服当前方案的哪些缺陷和风险，原因是没有跳出两次构建这个圈子，两次构建造成的限制太大
我个人觉得两次构建是一种反直觉的方式在解决source map生成的速度慢、OOM等问题
或者说这是一种强依赖Webpack本身的方式


而直觉的解决方案应该是 **从Webpack中提取source-map构建过程，将其异步延迟构建**
总的流程应该只进行 **一次构建**


### 总体流程


### Webpack流程图-侵入点


参考了webpack官方的sourcemap插件，侵入到_compilation-seal阶段即可，根据情况选择侵入afterOptimizeChunkAssets还是optimizeChunkAssets_
![截屏2020-06-29 17.14.37.png](https://intranetproxy.alipay.com/skylark/lark/0/2020/png/300858/1593422100019-12e0c0eb-6874-473e-9925-c9b7766ef401.png#align=left&display=inline&height=1834&margin=%5Bobject%20Object%5D&name=%E6%88%AA%E5%B1%8F2020-06-29%2017.14.37.png&originHeight=1834&originWidth=1834&size=958114&status=done&style=none&width=1834)


# 可控方案-技术系分
## Case1：持久化Webpack Source
### node方案
侵入点：_afterOptimizeChunkAssets_
_官方的sourceMap插件的源码提供了思路_
```javascript
// SourceMapDevToolPlugin.js

// 在afterOptimizeChunkAssets这个hook中添加事件
// ...把chunk的所有file添加到files中集中处理
const files = [];
for (const chunk of chunks) {
	for (const file of chunk.files) {
		if (matchObject(file)) {
			files.push({
				file,
				chunk
			});
		}
	}
}
// ...在files.foreach中
// 获取source
// 查阅webpack-sources https://github.com/webpack/webpack-sources
// Source.prototype.source() -> String | Buffer
// Returns the represented source code as string or Buffer (for binary Sources).
const asset = compilation.getAsset(file).source; 

// cache中有可能已经有sourcemap
const cache = assetsCache.get(asset);

// 如果没有，就generate sourcemap
// 创建一个task
const task = getTaskForFile(
	file,
	asset,
	chunk,
	options,
	compilation
);
// into func getTaskForFile()
// sourceAndMap: 可以查阅webpack-sources
// Source.prototype.sourceAndMap(options?: Object) -> {
//	 source: String | Buffer,
//	 map: Object
// }
if (asset.sourceAndMap) {
	const sourceAndMap = asset.sourceAndMap(options);
	sourceMap = sourceAndMap.map;
	source = sourceAndMap.source;
} else {
	sourceMap = asset.map(options);
	source = asset.source();
}

// 在tasks.foreach中就生成了*.map文件
```
我们对每一个asset做序列化，抛给sourcemap计算群，后面的流程是反序列化，调用source-map（其实按照官方插件的思路，直接调用webpack-sources的相应接口即可）


### esbuild方案
侵入点：optimizeChunkAssets
云谦老师的esbuild-webpack-plugin给了启发思路
可以跑两次esbuild，去掉缓慢的terser...一举两得
#### 代码展示流程
```javascript
import { startService, Service } from 'esbuild';
// 启动esbuild service
service = await startService();

const { source, map } = assetSource.sourceAndMap();
// 跑esbuild的minify服务
const result = await ESBuildPlugin.service.transform(source, {
  options,
  minify: true,
  sourcemap: false,
  sourcefile: file, // filename
});
// update emit asset

// 序列化 source,map
function serialize(content, outputPath) {
	// ...
}
serialize(source, sourceOutputPath)
serialize(map, mapOutputPath)
// 抛给sourcemap计算集群

---
// sourcemap计算函数中
// 反序列化
const source = deserialize(sourceOutputPath)
const map = deserialize(mapOutputPath)
// 跑esbuild的minify+sourcemap服务
const result = await ESBuildPlugin.service.transform(source, {
  options,
  minify: true,
  sourcemap: options.devtool,
  sourcefile: file, // filename
});
// 抛给sourcemap CDN
putCDN(file, result.jsSourceMap)

```


## Case2：持久化AST
case1方案中调用source-map库的Consumer生成AST，持久化AST，抛给Sourcemap计算群
接下来可以选择Node方案或者Rust方案

