---
title: Rolldown源码学习
date: 2024-03-30 16:32:40
tags:
  - rolldown
  - rust
categories: 学习笔记
---
# 前言

网上没有找到 Rolldown 的源码阅读，所以就自己简单看一看，串一下逻辑，总结成笔记。

# 入口

Rolldown 的入口位于 `packages/rolldown/src/index.ts` 中，核心就是创建一个 `RolldownBuild` js 对象，在内部绑定好对应的 bundler ：

```javascript
export const rolldown = async (input: InputOptions): Promise<RolldownBuild> => {
  return new RolldownBuild(input)
}

export class RolldownBuild {
  #inputOptions: InputOptions
  #bundler?: Bundler

  constructor(inputOptions: InputOptions) {
    // TODO: Check if `inputOptions.output` is set. If so, throw an warning that it is ignored.
    this.#inputOptions = inputOptions
  }

  async #getBundler(outputOptions: OutputOptions): Promise<Bundler> {
    if (typeof this.#bundler === 'undefined') {
      this.#bundler = await createBundler(this.#inputOptions, outputOptions)
    }
    return this.#bundler
  }

  async generate(outputOptions: OutputOptions = {}): Promise<RolldownOutput> {
    const bundler = await this.#getBundler(outputOptions)
    const output = await bundler.generate()
    return transformToRollupOutput(output)
  }

  async write(outputOptions: OutputOptions = {}): Promise<RolldownOutput> {
    const bundler = await this.#getBundler(outputOptions)
    const output = await bundler.write()
    return transformToRollupOutput(output)
  }
}

```

其中 Bundler 是 napi 的产物，接下来就会进入 rust 的代码中 `crates/rolldown_binding/src/bundler.rs` 调用的是对应的 impl

```rust
  #[instrument(skip_all)]
  #[allow(clippy::significant_drop_tightening)]
  pub async fn write_impl(&self) -> napi::Result<BindingOutputs> {
    let mut bundler_core = self.inner.try_lock().map_err(|_| {
      napi::Error::from_reason("Failed to lock the bundler. Is another operation in progress?")
    })?;

    // Bundler 的 write
    let maybe_outputs = bundler_core.write().await;

    let outputs = match maybe_outputs {
      Ok(outputs) => outputs,
      Err(errs) => return Err(Self::handle_errors(errs)),
    };

    Self::handle_warnings(outputs.warnings);

    Ok(outputs.assets.into())
  }

  #[instrument(skip_all)]
  #[allow(clippy::significant_drop_tightening)]
  pub async fn generate_impl(&self) -> napi::Result<BindingOutputs> {
    let mut bundler_core = self.inner.try_lock().map_err(|_| {
      napi::Error::from_reason("Failed to lock the bundler. Is another operation in progress?")
    })?;

    // Bundler 的 generate
    let maybe_outputs = bundler_core.generate().await;

    let outputs = match maybe_outputs {
      Ok(outputs) => outputs,
      Err(errs) => return Err(Self::handle_errors(errs)),
    };

    Self::handle_warnings(outputs.warnings);

    Ok(outputs.assets.into())
  }
```

然后就可以找到核心的 Bundler 的位置位于 `crates/rolldown/src/bundler.rs` 中了。

# Build 总览

通过上文的 js 代码可以看出，核心就是 generate 和 write 两个方法，而源码中的 write 是在跟 generate 一样在调用 bundle_up 后，将产物直接写入本地，可见 bundle_up 方法是核心：

```rust
impl Bundler {
  pub async fn write(&mut self) -> BatchedResult<RolldownOutput> {
    let dir =
      self.input_options.cwd.as_path().join(&self.output_options.dir).to_string_lossy().to_string();
    
    // 获取 generate 结果
    let output = self.bundle_up(true).await?;

    self.plugin_driver.write_bundle(&output.assets).await?;

    self.fs.create_dir_all(dir.as_path()).unwrap_or_else(|_| {
      panic!(
        "Could not create directory for output chunks: {:?} \ncwd: {}",
        dir.as_path(),
        self.input_options.cwd.display()
      )
    });
    // 写文件
    for chunk in &output.assets {
      let dest = dir.as_path().join(chunk.file_name());
      if let Some(p) = dest.parent() {
        if !self.fs.exists(p) {
          self.fs.create_dir_all(p).unwrap();
        }
      };
      self.fs.write(dest.as_path(), chunk.content().as_bytes()).unwrap_or_else(|_| {
        panic!("Failed to write file in {:?}", dir.as_path().join(chunk.file_name()))
      });
    }

    Ok(output)
  }

  pub async fn generate(&mut self) -> BatchedResult<RolldownOutput> {
    self.bundle_up(false).await
  }
}
```

bundle_up 除了 plugin hooks 执行以外，只做了 try_build 和  bundle_stage.bundle 两个事情，接下来主要查看两个函数：

```rust
  async fn bundle_up(&mut self, is_write: bool) -> BatchedResult<RolldownOutput> {
    tracing::trace!("InputOptions {:#?}", self.input_options);
    tracing::trace!("OutputOptions: {:#?}", self.output_options);
    let mut link_stage_output = self.try_build().await?;

    let mut bundle_stage = BundleStage::new(
      &mut link_stage_output,
      &self.input_options,
      &self.output_options,
      &self.plugin_driver,
    );

    let assets = bundle_stage.bundle().await?;

    self.plugin_driver.generate_bundle(&assets, is_write).await?;

    Ok(RolldownOutput { warnings: std::mem::take(&mut link_stage_output.warnings), assets })
  }
```

# try_build

try_build 做了扫描所有文件，和对他们进行了关联

```rust
  async fn try_build(&mut self) -> BatchedResult<LinkStageOutput> {
    self.plugin_driver.build_start().await?;

    // 扫描
    let scan_ret = self.scan_inner().await;

    self.call_build_end_hook(&scan_ret).await?;

    let build_info = scan_ret?;

    // 对扫描结果进行关联
    let link_stage = LinkStage::new(build_info, &self.input_options);
    Ok(link_stage.link())
  }
  
  async fn scan_inner(&mut self) -> BatchedResult<ScanStageOutput> {
    ScanStage::new(
      Arc::clone(&self.input_options),
      Arc::clone(&self.plugin_driver),
      self.fs.clone(),
      Arc::clone(&self.resolver),
    )
    .scan()
    .await
  }
```

扫描的整体逻辑如下：

```rust
  pub async fn scan(&self) -> BatchedResult<ScanStageOutput> {
    tracing::info!("Start scan stage");
    assert!(!self.input_options.input.is_empty(), "You must supply options.input to rolldown");
    // 创建 ModuleLoader
    let mut module_loader = ModuleLoader::new(
      Arc::clone(&self.input_options),
      Arc::clone(&self.plugin_driver),
      self.fs.clone(),
      Arc::clone(&self.resolver),
    );

    // 创建 rolldown 默认 runtime 的打包任务
    module_loader.try_spawn_runtime_module_task();

    // 获取用户的所有 entry
    let user_entries = self.resolve_user_defined_entries()?;

    // 递归拿到所有 entry 关联的 module
    let ModuleLoaderOutput { module_table, entry_points, symbols, runtime, warnings, ast_table } =
      module_loader.fetch_all_modules(user_entries).await?;

    tracing::debug!("Scan stage finished {module_table:#?}");

    Ok(ScanStageOutput { module_table, entry_points, symbols, runtime, warnings, ast_table })
  }
```

其中创建 `ModuleLoader` 的时候，其中包含了一个 mpsc ，因为 rolldown 使用 tokio 将所有文件的扫描变成异步任务，这个 mpsc 主要是用于接收任务结果：

```rust
  pub fn new(
    input_options: SharedNormalizedInputOptions,
    plugin_driver: SharedPluginDriver,
    fs: OsFileSystem,
    resolver: SharedResolver,
  ) -> Self {
    let (tx, rx) = tokio::sync::mpsc::unbounded_channel::<Msg>();

    let common_data = ModuleTaskCommonData {
      input_options: Arc::clone(&input_options),
      tx,
      resolver,
      fs,
      plugin_driver,
    };

    Self {
      common_data,
      rx,
      input_options,
      visited: FxHashMap::default(),
      runtime_id: None,
      remaining: 0,
      intermediate_normal_modules: IntermediateNormalModules::new(),
      external_modules: IndexVec::new(),
      symbols: Symbols::default(),
    }
  }
```
`try_spawn_runtime_module_task` 方法是将默认的 runtime 作为一个打包任务放入执行列队

```rust
  pub fn try_spawn_runtime_module_task(&mut self) -> NormalModuleId {
    *self.runtime_id.get_or_insert_with(|| {
      let id = self.intermediate_normal_modules.alloc_module_id(&mut self.symbols);
      self.remaining += 1;
      let task = RuntimeNormalModuleTask::new(id, self.common_data.tx.clone());
      tokio::spawn(async move { task.run() });
      id
    })
  }

  
  pub fn run(self) {
    tracing::trace!("process <runtime>");
    let mut builder = NormalModuleBuilder::default();

    // 默认 runtime 位置
    let source: Arc<str> =
      include_str!("../runtime/runtime-without-comments.js").to_string().into();

    // 生成 ast
    let (ast, scope, scan_result, symbol, namespace_symbol) = self.make_ast(&source);

    let runtime = RuntimeModuleBrief::new(self.module_id, &scope);

    // 构建扫描结果
    let ScanResult {
      named_imports,
      named_exports,
      stmt_infos,
      star_exports,
      default_export_ref,
      imports,
      repr_name,
      import_records: _,
      exports_kind: _,
      warnings: _,
    } = scan_result;

    builder.source = Some(source);
    builder.id = Some(self.module_id);
    builder.repr_name = Some(repr_name);
    // TODO: Runtime module should not have FilePath as source id
    builder.path = Some(ResourceId::new("runtime".to_string().into()));
    builder.named_imports = Some(named_imports);
    builder.named_exports = Some(named_exports);
    builder.stmt_infos = Some(stmt_infos);
    builder.imports = Some(imports);
    builder.star_exports = Some(star_exports);
    builder.default_export_ref = Some(default_export_ref.expect("should exist"));
    builder.import_records = Some(IndexVec::default());
    builder.scope = Some(scope);
    builder.exports_kind = Some(ExportsKind::Esm);
    builder.namespace_symbol = Some(namespace_symbol);
    builder.pretty_path = Some("<runtime>".to_string());
    builder.is_user_defined_entry = Some(false);

    // 将结果传递给 ModuleLoader
    if let Err(_err) = self.tx.send(Msg::RuntimeNormalModuleDone(RuntimeNormalModuleTaskResult {
      warnings: self.warnings,
      ast_symbol: symbol,
      builder,
      runtime,
      ast,
    })) {
      // hyf0: If main thread is dead, we should handle errors of main thread. So we just ignore the error here.
    };
  }

```

接下来就是扫描用户的所有 entry 方法 `fetch_all_modules`， 他会创建两个 table 保存 modules 和 ast，然后遍历 entry ，为每个 entry 创建一个扫描 task，最后 while mpsc，直到全部文件扫描完毕

```rust
  pub async fn fetch_all_modules(
    mut self,
    user_defined_entries: Vec<(Option<String>, ResolvedRequestInfo)>,
  ) -> BatchedResult<ModuleLoaderOutput> {
    assert!(!self.input_options.input.is_empty(), "You must supply options.input to rolldown");

    let mut errors = BatchedErrors::default();
    let mut all_warnings: Vec<BuildError> = Vec::new();

    self
      .intermediate_normal_modules
      .modules
      .reserve(user_defined_entries.len() + 1 /* runtime */);
    self
      .intermediate_normal_modules
      .ast_table
      .reserve(user_defined_entries.len() + 1 /* runtime */);

    // Store the already consider as entry module
    let mut user_defined_entry_ids = {
      let mut tmp = FxHashSet::default();
      tmp.reserve(user_defined_entries.len());
      tmp
    };

    // 遍历 entry
    let mut entry_points = user_defined_entries
      .into_iter()
      .map(|(name, info)| EntryPoint {
        name,
        // 创建扫描任务
        id: self.try_spawn_new_task(&info).expect_normal(),
        kind: EntryPointKind::UserDefined,
      })
      .inspect(|e| {
        user_defined_entry_ids.insert(e.id);
      })
      .collect::<Vec<_>>();

    let mut dynamic_import_entry_ids = FxHashSet::default();

    let mut runtime_brief: Option<RuntimeModuleBrief> = None;

    let mut panic_errors = vec![];

    // 不断从 channel 中拿取扫描结果
    while self.remaining > 0 {
      let Some(msg) = self.rx.recv().await else {
        break;
      };
      match msg {
        // 用户文件的结果
        Msg::NormalModuleDone(task_result) => {
          let NormalModuleTaskResult {
            module_id,
            ast_symbol,
            resolved_deps,
            mut builder,
            raw_import_records,
            warnings,
            ast,
          } = task_result;
          all_warnings.extend(warnings);
          // 如果有 import * from xxx 则再创建对应的异步任务
          let import_records = raw_import_records
            .into_iter()
            .zip(resolved_deps)
            .map(|(raw_rec, info)| {
              let id = self.try_spawn_new_task(&info);
              // Dynamic imported module will be considered as an entry
              if let ModuleId::Normal(id) = id {
                if matches!(raw_rec.kind, ImportKind::DynamicImport)
                  && !user_defined_entry_ids.contains(&id)
                {
                  dynamic_import_entry_ids.insert(id);
                }
              }
              raw_rec.into_import_record(id)
            })
            .collect::<IndexVec<ImportRecordId, _>>();
          builder.import_records = Some(import_records);
          builder.is_user_defined_entry = Some(user_defined_entry_ids.contains(&module_id));
          self.intermediate_normal_modules.modules[module_id] = Some(builder.build());
          self.intermediate_normal_modules.ast_table[module_id] = Some(ast);

          self.symbols.add_ast_symbol(module_id, ast_symbol);
        }
        // runtime 的结果
        Msg::RuntimeNormalModuleDone(task_result) => {
          let RuntimeNormalModuleTaskResult { ast_symbol, builder, runtime, warnings: _, ast } =
            task_result;

          self.intermediate_normal_modules.modules[runtime.id()] = Some(builder.build());
          self.intermediate_normal_modules.ast_table[runtime.id()] = Some(ast);

          self.symbols.add_ast_symbol(runtime.id(), ast_symbol);
          runtime_brief = Some(runtime);
        }
        Msg::BuildErrors(errs) => {
          errors.extend(errs);
        }
        Msg::Panics(err) => {
          panic_errors.push(err);
        }
      }
      self.remaining -= 1;
    }

    assert!(panic_errors.is_empty(), "Panics occurred during module loading: {panic_errors:?}");

    if !errors.is_empty() {
      return Err(errors);
    }

    let modules: IndexVec<NormalModuleId, NormalModule> =
      self.intermediate_normal_modules.modules.into_iter().map(Option::unwrap).collect();

    let ast_table: IndexVec<NormalModuleId, OxcProgram> =
      self.intermediate_normal_modules.ast_table.into_iter().map(Option::unwrap).collect();

    let mut dynamic_import_entry_ids = dynamic_import_entry_ids.into_iter().collect::<Vec<_>>();
    dynamic_import_entry_ids.sort_by_key(|id| &modules[*id].resource_id);

    entry_points.extend(dynamic_import_entry_ids.into_iter().map(|id| EntryPoint {
      name: None,
      id,
      kind: EntryPointKind::DynamicImport,
    }));

    Ok(ModuleLoaderOutput {
      module_table: ModuleTable {
        normal_modules: modules,
        external_modules: self.external_modules,
      },
      symbols: self.symbols,
      ast_table,
      entry_points,
      runtime: runtime_brief.expect("Failed to find runtime module. This should not happen"),
      warnings: all_warnings,
    })
  }
}


// 创建扫描任务
  fn try_spawn_new_task(&mut self, info: &ResolvedRequestInfo) -> ModuleId {
    match self.visited.entry(Arc::<str>::clone(&info.path.path)) {
      // 如果有就直接返回结果
      std::collections::hash_map::Entry::Occupied(visited) => *visited.get(),
      std::collections::hash_map::Entry::Vacant(not_visited) => {
        // external
        if info.is_external {
          let id = self.external_modules.len_idx();
          not_visited.insert(id.into());
          let ext = ExternalModule::new(id, info.path.path.to_string());
          self.external_modules.push(ext);
          id.into()
        } else {
          // 项目内文件
          let id = self.intermediate_normal_modules.alloc_module_id(&mut self.symbols);
          not_visited.insert(id.into());
          self.remaining += 1;
          let module_path = info.path.clone();

          let task = NormalModuleTask::new(
            // safety: Data in `ModuleTaskContext` are alive as long as the `NormalModuleTask`, but rustc doesn't know that.
            unsafe { self.common_data.assume_static() },
            id,
            module_path,
            info.module_type,
          );
          tokio::spawn(async move { task.run().await });
          id.into()
        }
      }
    }
  }
```

模块扫描做的事情就比较简单，加载文件，通过 oxc 将其 ast 获得，构建扫描结果

```rust
  async fn run_inner(&mut self) -> anyhow::Result<()> {
    tracing::trace!("process {:?}", self.resolved_path);

    let mut sourcemap_chain = vec![];
    let mut warnings = vec![];

    // Run plugin load to get content first, if it is None using read fs as fallback.
    let source = match load_source(
      &self.ctx.plugin_driver,
      &self.resolved_path,
      &self.ctx.fs,
      &mut sourcemap_chain,
    )
    .await
    {
      Ok(ret) => ret,
      Err(errs) => {
        self.errors.extend(errs);
        return Ok(());
      }
    };

    // Run plugin transform.
    let source: Arc<str> = match transform_source(
      &self.ctx.plugin_driver,
      &self.resolved_path,
      source,
      &mut sourcemap_chain,
    )
    .await
    {
      Ok(ret) => ret.into(),
      Err(errs) => {
        self.errors.extend(errs);
        return Ok(());
      }
    };

    let (ast, scope, scan_result, ast_symbol, namespace_symbol) = self.scan(&source);
    tracing::trace!("scan {:?}", self.resolved_path);

    let res = self.resolve_dependencies(&scan_result.import_records).await?;

    let ScanResult {
      named_imports,
      named_exports,
      stmt_infos,
      import_records,
      star_exports,
      default_export_ref,
      imports,
      exports_kind,
      repr_name,
      warnings: scan_warnings,
    } = scan_result;
    warnings.extend(scan_warnings);

    let builder = NormalModuleBuilder {
      source: Some(source),
      id: Some(self.module_id),
      repr_name: Some(repr_name),
      path: Some(ResourceId::new(Arc::<str>::clone(&self.resolved_path.path).into())),
      named_imports: Some(named_imports),
      named_exports: Some(named_exports),
      stmt_infos: Some(stmt_infos),
      imports: Some(imports),
      star_exports: Some(star_exports),
      default_export_ref,
      scope: Some(scope),
      exports_kind: Some(exports_kind),
      namespace_symbol: Some(namespace_symbol),
      module_type: self.module_type,
      pretty_path: Some(self.resolved_path.prettify(&self.ctx.input_options.cwd)),
      sourcemap_chain,
      ..Default::default()
    };

    self
      .ctx
      .tx
      .send(Msg::NormalModuleDone(NormalModuleTaskResult {
        resolved_deps: res,
        module_id: self.module_id,
        warnings,
        ast_symbol,
        builder,
        raw_import_records: import_records,
        ast,
      }))
      .expect("Send should not fail");
    tracing::trace!("end process {:?}", self.resolved_path);
    Ok(())
  }
```

在扫描完成后，我们会得到结果 `ScanStageOutput`,下一步是通过 `LinkStage` 将其关联起来

```rust
  pub fn link(mut self) -> LinkStageOutput {
    tracing::info!("Start link stage");
    self.sort_modules();

    // 根据每个模块的引入，找到他们对应的导出，然后确定导出的类型（cjs esm）
    self.determine_module_exports_kind();
    // 封装模块的各种信息，esm cjs
    self.wrap_modules();
    // 绑定模块之间的导入和导出关系
    self.bind_imports_and_exports();
    tracing::debug!("linking modules {:#?}", self.metas);
    // 为每个模块创建好自己的导出闭包
    //  CommonJS
    //   var require_foo = __commonJS((exports, module) => {
    //     ...
    //   });
    //  ESM
    //   var init_foo = __esm(() => {
    //     ...
    //   });
    self.create_exports_for_modules();
    // 处理引入和包symbol的关系
    // ESM
    // commonjs: require_foo -> var import_foo = __toESM(require_foo()) esm: init_foo -> init_foo()
    // CommonJS
    // commonjs: require_foo -> require_foo() esm: init_foo -> (init_foo(), toCommonJS(foo_exports))
    self.reference_needed_symbols();
    // 使用rayon并行标记语句是否被使用
    self.include_statements();

    LinkStageOutput {
      module_table: self.module_table,
      entries: self.entries,
      sorted_modules: self.sorted_modules,
      metas: self.metas,
      symbols: self.symbols,
      runtime: self.runtime,
      warnings: self.warnings,
      ast_table: self.ast_table,
    }
  }
```

# bundle

bundle 环节是将上一步的 module 们进行组合生成 chunks 和 asset

```rust
  pub async fn bundle(&mut self) -> BatchedResult<Vec<Output>> {
    use rayon::prelude::*;
    tracing::info!("Start bundle stage");
    // 构建chunk关联图
    let mut chunk_graph = self.generate_chunks();

    self.generate_chunk_filenames(&mut chunk_graph);
    tracing::info!("generate_chunk_filenames");

    self.compute_cross_chunk_links(&mut chunk_graph);
    tracing::info!("compute_cross_chunk_links");

    // 并行处理每个块，解决chunk间名称冲突（如变量、函数）
    chunk_graph.chunks.iter_mut().par_bridge().for_each(|chunk| {
      chunk.de_conflict(self.link_output);
    });

    // 并行处理每个模块，进行模块的最终化处理
    self
      .link_output
      .ast_table
      .iter_mut_enumerated()
      .par_bridge()
      .filter(|(id, _)| self.link_output.module_table.normal_modules[*id].is_included)
      .for_each(|(id, ast)| {
        let module = &self.link_output.module_table.normal_modules[id];
        let chunk_id = chunk_graph.module_to_chunk[module.id].unwrap();
        let chunk = &chunk_graph.chunks[chunk_id];
        let linking_info = &self.link_output.metas[module.id];
        // 确定是否需要生成命名空间变量声明，并将其添加。
        // 遍历模块中的每个语句，根据语句的类型进行相应的处理：
        // 对于导入声明，判断是否需要移除，并根据情况执行相应的操作。
        // 对于导出声明，根据不同的导出形式执行相应的操作，如移除导出声明、添加重定向导出等。
        // 对于默认导出声明，将其转换为对应的声明形式。
        // 如果需要，在程序体的最前面或最后面添加包装器，用于将模块封装成 CommonJS 或 ESM 形式。
        finalize_normal_module(
          module,
          FinalizerContext {
            canonical_names: &chunk.canonical_names,
            id: module.id,
            symbols: &self.link_output.symbols,
            linking_info,
            module,
            modules: &self.link_output.module_table.normal_modules,
            linking_infos: &self.link_output.metas,
            runtime: &self.link_output.runtime,
            chunk_graph: &chunk_graph,
          },
          ast,
        );
      });
    tracing::info!("finalizing modules");

    let chunks = block_on_spawn_all(chunk_graph.chunks.iter().map(|c| async {
      // 将最终的ast变成最终的代码和 sourcemap。
      c.render(self.input_options, self.link_output, &chunk_graph, self.output_options).await
    }))
    .into_iter()
    .collect::<Result<Vec<_>, _>>()?;

    let mut assets = vec![];
    // 使用异步生成每个 chunk
    render_chunks(self.plugin_driver, chunks).await?.into_iter().try_for_each(
      |chunk| -> Result<(), BuildError> {
        let ChunkRenderReturn { mut map, rendered_chunk, mut code } = chunk;
        if let Some(map) = map.as_mut() {
          map.set_file(Some(rendered_chunk.file_name.clone()));
          match self.output_options.sourcemap {
            SourceMapType::File => {
              let map = {
                let mut buf = vec![];
                map.to_writer(&mut buf).map_err(|e| BuildError::sourcemap_error(e.to_string()))?;
                unsafe { String::from_utf8_unchecked(buf) }
              };
              let map_file_name = format!("{}.map", rendered_chunk.file_name);
              assets.push(Output::Asset(Box::new(OutputAsset {
                file_name: map_file_name.clone(),
                source: map,
              })));
              code.push_str(&format!("\n//# sourceMappingURL={map_file_name}"));
            }
            SourceMapType::Inline => {
              let data_url =
                map.to_data_url().map_err(|e| BuildError::sourcemap_error(e.to_string()))?;
              code.push_str(&format!("\n//# sourceMappingURL={data_url}"));
            }
            SourceMapType::Hidden => {}
          }
        }
        let sourcemap_file_name = map.as_ref().map(|_| format!("{}.map", rendered_chunk.file_name));
        assets.push(Output::Chunk(Box::new(OutputChunk {
          file_name: rendered_chunk.file_name,
          code,
          is_entry: rendered_chunk.is_entry,
          is_dynamic_entry: rendered_chunk.is_dynamic_entry,
          facade_module_id: rendered_chunk.facade_module_id,
          modules: rendered_chunk.modules,
          exports: rendered_chunk.exports,
          module_ids: rendered_chunk.module_ids,
          map,
          sourcemap_file_name,
        })));
        Ok(())
      },
    )?;

    tracing::info!("rendered chunks");

    Ok(assets)
  }

  
  pub fn generate_chunks(&self) -> ChunkGraph {
    let entries_len: u32 =
      self.link_output.entries.len().try_into().expect("Too many entries, u32 overflowed.");
    // If we are in test environment, to make the runtime module always fall into a standalone chunk,
    // we create a facade entry point for it.
    let entries_len = if is_in_rust_test_mode() { entries_len + 1 } else { entries_len };

    let mut module_to_bits = index_vec::index_vec![BitSet::new(entries_len); self.link_output.module_table.normal_modules.len()];
    let mut bits_to_chunk = FxHashMap::with_capacity_and_hasher(
      self.link_output.entries.len(),
      BuildHasherDefault::default(),
    );
    let mut chunks = ChunksVec::with_capacity(self.link_output.entries.len());

    // 为 entry + 动态引入 创建 chunk
    for (entry_index, entry_point) in self.link_output.entries.iter().enumerate() {
      let count: u32 = entry_index.try_into().expect("Too many entries, u32 overflowed.");
      let mut bits = BitSet::new(entries_len);
      bits.set_bit(count);
      let module = &self.link_output.module_table.normal_modules[entry_point.id];
      let chunk = chunks.push(Chunk::new(
        entry_point.name.clone(),
        bits.clone(),
        vec![],
        ChunkKind::EntryPoint {
          is_user_defined: module.is_user_defined_entry,
          bit: count,
          module: entry_point.id,
        },
      ));
      bits_to_chunk.insert(bits, chunk);
    }

    if is_in_rust_test_mode() {
      self.determine_reachable_modules_for_entry(
        self.link_output.runtime.id(),
        entries_len - 1,
        &mut module_to_bits,
      );
    }

    // 通过 entry 确定模块是否被多个 chunk 引入，多个 module 可以被 chunk 复用
    self.link_output.entries.iter().enumerate().for_each(|(i, entry_point)| {
      self.determine_reachable_modules_for_entry(
        entry_point.id,
        i.try_into().expect("Too many entries, u32 overflowed."),
        &mut module_to_bits,
      );
    });

    let mut module_to_chunk: IndexVec<NormalModuleId, Option<ChunkId>> = index_vec::index_vec![
      None;
      self.link_output.module_table.normal_modules.len()
    ];
    
    // 1. 将模块分配给相应的 chunk
    // 2. 创建共享chunk来存储属于多个chunk的模块。
    for normal_module in &self.link_output.module_table.normal_modules {
      if !normal_module.is_included {
        continue;
      }

      let bits = &module_to_bits[normal_module.id];
      debug_assert!(
        !bits.is_empty(),
        "Empty bits means the module is not reachable, so it should bail out with `is_included: false`"
      );
      if let Some(chunk_id) = bits_to_chunk.get(bits).copied() {
        chunks[chunk_id].modules.push(normal_module.id);
        module_to_chunk[normal_module.id] = Some(chunk_id);
      } else {
        let chunk = Chunk::new(None, bits.clone(), vec![normal_module.id], ChunkKind::Common);
        let chunk_id = chunks.push(chunk);
        module_to_chunk[normal_module.id] = Some(chunk_id);
        bits_to_chunk.insert(bits.clone(), chunk_id);
      }
    }

    // 按执行顺序对每个块中的模块进行排序
    chunks.iter_mut().for_each(|chunk| {
      chunk.modules.sort_by_key(|module_id| {
        self.link_output.module_table.normal_modules[*module_id].exec_order
      });
    });

    tracing::trace!("Generated chunks: {:#?}", chunks);

    ChunkGraph { chunks, module_to_chunk }
  }
```