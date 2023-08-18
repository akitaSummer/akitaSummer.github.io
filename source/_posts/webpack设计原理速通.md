---
title: webpack设计原理速通
date: 2023-08-13 23:26:25
tags:
	- Webpack
categories: 学习笔记
---

# 前言

最近抽空阅读了一下 webpack 源码，本文是结合源码和网上文章的总结，形成的个人学习笔记

# webpack 运行概述

如同官方的宣传图描述的一样，webpack 主要功能就是把各种文件最终打包成为浏览器可运行的 html，css 和 js
![webpack_home](/image/webpack/webpack_home.png)

而其中的运行概述流程如下
![webpack_architecture](/image/webpack/webpack_architecture.png)

主要分为以下几个阶段

- 初始化
  > - 从配置文件、 配置对象、Shell 参数中读取参数并形成 Compiler 对象
  > - 注册 plugin
  > - run!
- 构建
  > - compilition.addEntry 将会遍历所有 entry，将其的依赖全部遍历出来，并构建依赖图谱
  > - 为每个依赖创建一个 module 对象，对象里包含每个依赖的信息，然后调用 loader 来处理
  > - 保存编译后的 module 对象和依赖图谱
- 封装
  > - 根据 entry 和 module 的关系，生成 chunk
  > - 一般来说一个 chunk 对应一个 bundle，动态加载会单独生成 chunk
  > - 这个时机会将 webpack runtime 打包进去
- 输出
  > - 生成代码，并写入文件

# 初始化阶段

首先会先调用[createCompiler](https://github.com/webpack/webpack/blob/main/lib/webpack.js#L143)构建 Compiler

```javascript
const createCompiler = (rawOptions) => {
  const options = getNormalizedWebpackOptions(rawOptions); // 从配置文件、 配置对象、Shell 参数中读取参数
  applyWebpackOptionsBaseDefaults(options); // 合并默认值
  const compiler = new Compiler(
    /** @type {string} */ (options.context),
    options
  );
  new NodeEnvironmentPlugin({
    infrastructureLogging: options.infrastructureLogging,
  }).apply(compiler); // 处理 entry 配置
  if (Array.isArray(options.plugins)) {
    // 遍历plugin，调用apply钩子
    for (const plugin of options.plugins) {
      if (typeof plugin === "function") {
        plugin.call(compiler, compiler);
      } else if (plugin) {
        plugin.apply(compiler);
      }
    }
  }
  applyWebpackOptionsDefaults(options); // 合并默认值
  compiler.hooks.environment.call();
  compiler.hooks.afterEnvironment.call();
  new WebpackOptionsApply().process(options, compiler); // 加载内置各种插件
  compiler.hooks.initialize.call();
  return compiler;
};
```

然后执行[compiler.run](https://github.com/webpack/webpack/blob/main/lib/Compiler.js#L424),最终调用到[compiler.compile](https://github.com/webpack/webpack/blob/main/lib/Compiler.js#L1163)中

```javascript
	compile(callback) {
		const params = this.newCompilationParams();
		this.hooks.beforeCompile.callAsync(params, err => {
			if (err) return callback(err);

			this.hooks.compile.call(params);

			const compilation = this.newCompilation(params);

			const logger = compilation.getLogger("webpack.Compiler");

			logger.time("make hook");
            // https://github.com/webpack/webpack/blob/main/lib/EntryPlugin.js#L47
			this.hooks.make.callAsync(compilation, err => { // 触发 EntryPlugin 的 make 回调执行 compilation.addEntry
				logger.timeEnd("make hook");
				if (err) return callback(err);

				logger.time("finish make hook");
				this.hooks.finishMake.callAsync(compilation, err => {
					logger.timeEnd("finish make hook");
					if (err) return callback(err);

					process.nextTick(() => {
						logger.time("finish compilation");
						compilation.finish(err => {
							logger.timeEnd("finish compilation");
							if (err) return callback(err);

							logger.time("seal compilation");
							compilation.seal(err => { // 封装阶段的 compilation.seal
								logger.timeEnd("seal compilation");
								if (err) return callback(err);

								logger.time("afterCompile hook");
								this.hooks.afterCompile.callAsync(compilation, err => {
									logger.timeEnd("afterCompile hook");
									if (err) return callback(err);

									return callback(null, compilation);
								});
							});
						});
					});
				});
			});
		});
	}

```

# 构建阶段

这里借用掘金上的构建图
![webpack_build](/image/webpack/webpack_build.png)

首先会执行[compilation.addEntry](https://github.com/webpack/webpack/blob/main/lib/Compilation.js#L2146)，随后通过在其中调用[addModuleTree](https://github.com/webpack/webpack/blob/main/lib/Compilation.js#L2060)来完成每个 entry 的 ModuleTree 构建。

接下来就是从 entry 文件作为首个 module 开始构建，构建完成后，将 entry 文件的依赖放入列队中，然后下次循环在构建依赖，这样进行广度优先遍历，直到把所有 module 都构建好

![webpack_build_tree](/image/webpack/webpack_build_tree.png)

```javascript
    // 1
	_addEntryItem(context, entry, target, options, callback) {
        // ...

		this.addModuleTree(
			{
				context,
				dependency: entry,
				contextInfo: entryData.options.layer
					? { issuerLayer: entryData.options.layer }
					: undefined
			},
			(err, module) => {
				if (err) {
					this.hooks.failedEntry.call(entry, options, err);
					return callback(err);
				}
				this.hooks.succeedEntry.call(entry, options, module);
				return callback(null, module);
			}
		);
	}

    // 2
	addModuleTree({ context, dependency, contextInfo }, callback) {
        // ...

		this.handleModuleCreation(
			{
				factory: moduleFactory,
				dependencies: [dependency],
				originModule: null,
				contextInfo,
				context
			},
			(err, result) => {
				if (err && this.bail) {
					callback(err);
					this.buildQueue.stop();
					this.rebuildQueue.stop();
					this.processDependenciesQueue.stop();
					this.factorizeQueue.stop();
				} else if (!err && result) {
					callback(null, result);
				} else {
					callback();
				}
			}
		);
	}

    // 3
    handleModuleCreation(
		{
			factory,
			dependencies,
			originModule,
			contextInfo,
			context,
			recursive = true,
			connectOrigin = recursive
		},
		callback
	) {
		const moduleGraph = this.moduleGraph;

		const currentProfile = this.profile ? new ModuleProfile() : undefined;

		this.factorizeModule( // 构建Module
			{
				currentProfile,
				factory,
				dependencies,
				factoryResult: true,
				originModule,
				contextInfo,
				context
			},
			(err, factoryResult) => {
                // ...

                // 添加module到tree
				this.addModule(newModule, (err, module) => {
                    // ...

					moduleGraph.setIssuerIfUnset( // module依赖图谱加入新建立的module
						module,
						originModule !== undefined ? originModule : null
					);
					// ...

					this._handleModuleBuildAndDependencies( // build module
						originModule,
						module,
						recursive,
						callback
					);
				});
			}
		);
	}

    // 4
    _handleModuleBuildAndDependencies(originModule, module, recursive, callback) {
        this.buildModule(module, err => { // build 构建 这里会使用loader
            if (creatingModuleDuringBuildSet !== undefined) {
                creatingModuleDuringBuildSet.delete(module);
            }
            if (err) {
                if (!err.module) {
                    err.module = module;
                }
                this.errors.push(err);

                return callback(err);
            }

            if (!recursive) {
                this.processModuleDependenciesNonRecursive(module);
                callback(null, module);
                return;
            }

            // This avoids deadlocks for circular dependencies
            if (this.processDependenciesQueue.isProcessing(module)) {
                return callback(null, module);
            }

            this.processModuleDependencies(module, err => { // 构建完后，把当前Module的依赖放入module列队，在下次循环时构建依赖module
                if (err) {
                    return callback(err);
                }
                callback(null, module);
            });
        });
	}
```

# 封装阶段

当构建阶段完成后，就会进入封装阶段，主要为[compilation.seal](https://github.com/webpack/webpack/blob/main/lib/Compilation.js#L2788)方法的调用

```javascript
	seal(callback) {
		const finalCallback = err => {
			this.factorizeQueue.clear();
			this.buildQueue.clear();
			this.rebuildQueue.clear();
			this.processDependenciesQueue.clear();
			this.addModuleQueue.clear();
			return callback(err);
		};
		const chunkGraph = new ChunkGraph( // 构建chunk图谱
			this.moduleGraph,
			this.outputOptions.hashFunction
		);
		this.chunkGraph = chunkGraph;

        // ...


		for (const [name, { dependencies, includeDependencies, options }] of this
			.entries) { // 每一个 entry 构建一个 chunk 对象
			const chunk = this.addChunk(name);
			if (options.filename) {
				chunk.filenameTemplate = options.filename;
			}
			const entrypoint = new Entrypoint(options);
			if (!options.dependOn && !options.runtime) {
				entrypoint.setRuntimeChunk(chunk);
			}
			entrypoint.setEntrypointChunk(chunk);
			this.namedChunkGroups.set(name, entrypoint);
			this.entrypoints.set(name, entrypoint);
			this.chunkGroups.push(entrypoint);
			connectChunkGroupAndChunk(entrypoint, chunk);

			const entryModules = new Set(); // 遍历出所有 module 构建他们与chunk的关联
			for (const dep of [...this.globalEntry.dependencies, ...dependencies]) {
				entrypoint.addOrigin(null, { name }, /** @type {any} */ (dep).request);

				const module = this.moduleGraph.getModule(dep);
				if (module) {
					chunkGraph.connectChunkAndEntryModule(chunk, module, entrypoint);
					entryModules.add(module);
					const modulesList = chunkGraphInit.get(entrypoint);
					if (modulesList === undefined) {
						chunkGraphInit.set(entrypoint, [module]);
					} else {
						modulesList.push(module);
					}
				}
			}

			this.assignDepths(entryModules);

			const mapAndSort = deps =>
				deps
					.map(dep => this.moduleGraph.getModule(dep))
					.filter(Boolean)
					.sort(compareModulesByIdentifier);
			const includedModules = [
				...mapAndSort(this.globalEntry.includeDependencies),
				...mapAndSort(includeDependencies)
			];

			let modulesList = chunkGraphInit.get(entrypoint);
			if (modulesList === undefined) {
				chunkGraphInit.set(entrypoint, (modulesList = []));
			}
			for (const module of includedModules) {
				this.assignDepth(module);
				modulesList.push(module);
			}
		}

        // ...


		buildChunkGraph(this, chunkGraphInit); // 构建chunk图谱

        // ...

        // 构建树
		this.hooks.optimizeTree.callAsync(this.chunks, this.modules, err => {
			if (err) {
				return finalCallback(
					makeWebpackError(err, "Compilation.hooks.optimizeTree")
				);
			}

			this.hooks.optimizeChunkModules.callAsync(
				this.chunks,
				this.modules,
				err => {

                    // ...

					this.hooks.optimizeCodeGeneration.call(this.modules);
					this.logger.timeEnd("optimize");

					this.logger.time("module hashing");
					this.hooks.beforeModuleHash.call();
					this.createModuleHashes();
					this.hooks.afterModuleHash.call();
					this.logger.timeEnd("module hashing");

					this.logger.time("code generation");
					this.hooks.beforeCodeGeneration.call();
					this.codeGeneration(err => {  // 生成chunk对应的代码
						if (err) {
							return finalCallback(err);
						}
						this.hooks.afterCodeGeneration.call();
						this.logger.timeEnd("code generation");

						this.logger.time("runtime requirements");
						this.hooks.beforeRuntimeRequirements.call();
						this.processRuntimeRequirements();
						this.hooks.afterRuntimeRequirements.call();
						this.logger.timeEnd("runtime requirements");

						this.logger.time("hashing");
						this.hooks.beforeHash.call();
						const codeGenerationJobs = this.createHash();
						this.hooks.afterHash.call();
						this.logger.timeEnd("hashing");

						this._runCodeGenerationJobs(codeGenerationJobs, err => {
							if (err) {
								return finalCallback(err);
							}

							if (shouldRecord) {
								this.logger.time("record hash");
								this.hooks.recordHash.call(this.records);
								this.logger.timeEnd("record hash");
							}

							this.logger.time("module assets");
							this.clearAssets();

							this.hooks.beforeModuleAssets.call();
							this.createModuleAssets(); // 调用 compilation.emitAssets 方法将资 assets 信息记录到 compilation.assets 对象中
							this.logger.timeEnd("module assets");

                            // ...

							this.logger.time("create chunk assets");
							if (this.hooks.shouldGenerateChunkAssets.call() !== false) {
								this.hooks.beforeChunkAssets.call();
								this.createChunkAssets(err => { // 调用 compilation.emitAssets 方法将资 assets 信息记录到 compilation.assets 对象中
									this.logger.timeEnd("create chunk assets");
									if (err) {
										return finalCallback(err);
									}
									cont();
								});
							} else {
								this.logger.timeEnd("create chunk assets");
								cont();
							}
						});
					});
				}
			);
		});
	}

```

# 输出阶段

当 chunk 代码生成完毕后，会调用[compiler.emitAssets](https://github.com/webpack/webpack/blob/main/lib/Compiler.js#L592)，将生成的代码写入文件中，就是调用了`compiler.outputFileSystem.writeFile`来写入文件系统，具体细节比较绕，就不细看了

# 结尾

本文只是梳理了 webpack 的整体流程源码，具体细节（如 plugin，loader，chunksplit）并没有详细列出，如果感兴趣，可以查看[掘金上的文章](https://juejin.cn/post/6949040393165996040)，或者查看源码，这样会能加深对 webpack 的理解。
