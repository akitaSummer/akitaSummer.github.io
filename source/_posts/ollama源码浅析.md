---
title: ollama源码浅析
date: 2024-07-21 23:37:35
tags:
  - ollama
  - llama.cpp
  - golang
categories: 学习笔记
---

# 前言

本文是在 ollama 使用之后，对其内部实现进行的简易解析。

ollama 的项目目录如下：
```bash
.
├── api // go api client
├── app // 桌面应用程序
├── auth // 鉴权
├── cmd // 命令和相关的处理程序
├── convert
├── docs
├── envconfig
├── examples
├── format
├── gpu // GPU 的检测
├── integration // 集成测试
├── llm // 用于运行 llama.cpp 的实现
├── macapp // mac 桌面应用程序
├── openai // 用于 ollama 的 OpenAI API 兼容封装
├── parser // Modelfile 解析器
├── progress // 显示加载进度
├── readline // 从终端读取输入
├── scripts // 构建和发布的脚本
├── server // go web server
├── template
├── types
├── util
├── version
├── main.go // 入口
├── Dockerfile
├── LICENSE
├── README.md
├── go.mod
└── go.sum
```
ollama 的架构简单图
![architecture](/image/ollama/ollama.png)

# APIs

go项目的入口一般是 `main.go` , ollama 的非常简单，就是注册命令行

```go
package main

import (
	"context"

	"github.com/ollama/ollama/cmd"
	"github.com/spf13/cobra"
)

func main() {
	cobra.CheckErr(cmd.NewCLI().ExecuteContext(context.Background()))
}

```
我们可以在 `cmd/` 下找到 NewCLI 的实现
```go

func NewCLI() *cobra.Command {
        // ...

	createCmd := &cobra.Command{
		Use:     "create MODEL",
		Short:   "Create a model from a Modelfile",
		Args:    cobra.ExactArgs(1),
		PreRunE: checkServerHeartbeat,
		RunE:    CreateHandler,
	}

	createCmd.Flags().StringP("file", "f", "Modelfile", "Name of the Modelfile")
	createCmd.Flags().StringP("quantize", "q", "", "Quantize model to this level (e.g. q4_0)")

        // ...

	runCmd := &cobra.Command{
		Use:     "run MODEL [PROMPT]",
		Short:   "Run a model",
		Args:    cobra.MinimumNArgs(1),
		PreRunE: checkServerHeartbeat,
		RunE:    RunHandler,
	}

	runCmd.Flags().String("keepalive", "", "Duration to keep a model loaded (e.g. 5m)")
	runCmd.Flags().Bool("verbose", false, "Show timings for response")
	runCmd.Flags().Bool("insecure", false, "Use an insecure registry")
	runCmd.Flags().Bool("nowordwrap", false, "Don't wrap words to the next line automatically")
	runCmd.Flags().String("format", "", "Response format (e.g. json)")

        // ...

	rootCmd.AddCommand(
		serveCmd,
		createCmd,
		showCmd,
		runCmd,
		pullCmd,
		pushCmd,
		listCmd,
		psCmd,
		copyCmd,
		deleteCmd,
	)

	return rootCmd
}
```
我们来看下最常用的 create 和 run 方法
```go
func CreateHandler(cmd *cobra.Command, args []string) error {
        // ...
        // 创建与 golang server 通信的 client
	client, err := api.ClientFromEnvironment()
        // ...
        // 读 modelfile
	f, err := os.Open(filename)
        // ...
	modelfile, err := parser.ParseFile(f)
        // ...
	for i := range modelfile.Commands {
		switch modelfile.Commands[i].Name {
		case "model", "adapter":
			path := modelfile.Commands[i].Args
                        // ...
                        // 计算 model 文件 hash 以及将其内容上传给 server
			digest, err := createBlob(cmd, client, path)
			if err != nil {
				return err
			}

			modelfile.Commands[i].Args = "@" + digest
		}
	}
        // ...
	request := api.CreateRequest{Name: args[0], Modelfile: modelfile.String(), Quantize: quantize}
	if err := client.Create(cmd.Context(), &request, fn); err != nil {
		return err
	}

	return nil
}

func createBlob(cmd *cobra.Command, client *api.Client, path string) (string, error) {
	bin, err := os.Open(path)
	if err != nil {
		return "", err
	}
	defer bin.Close()

	hash := sha256.New()
	if _, err := io.Copy(hash, bin); err != nil {
		return "", err
	}

	if _, err := bin.Seek(0, io.SeekStart); err != nil {
		return "", err
	}

	digest := fmt.Sprintf("sha256:%x", hash.Sum(nil))
	if err = client.CreateBlob(cmd.Context(), digest, bin); err != nil {
		return "", err
	}
	return digest, nil
}
```
create 方法做的事情就是将 model 文件注册上传到 server
```go
func RunHandler(cmd *cobra.Command, args []string) error {
	interactive := true
	// 初始化选项和变量
	opts := runOptions{
		Model:    args[0],
		WordWrap: os.Getenv("TERM") == "xterm-256color",
		Options:  map[string]interface{}{},
	}
        // ...
        // 从 server 获取模型信息
	name := args[0]
	info, err := func() (*api.ShowResponse, error) {
		showReq := &api.ShowRequest{Name: name}
		info, err := client.Show(cmd.Context(), showReq)
		var se api.StatusError
                // 没有就 pull 下来
		if errors.As(err, &se) && se.StatusCode == http.StatusNotFound {
			if err := PullHandler(cmd, []string{name}); err != nil {
				return nil, err
			}
			return client.Show(cmd.Context(), &api.ShowRequest{Name: name})
		}
		return info, err
	}()
	if err != nil {
		return err
	}

	opts.MultiModal = slices.Contains(info.Details.Families, "clip")
	opts.ParentModel = info.Details.ParentModel
	opts.Messages = append(opts.Messages, info.Messages...)

	if interactive {
		return generateInteractive(cmd, opts)
	}
	return generate(cmd, opts)
}


func generate(cmd *cobra.Command, opts runOptions) error {
        // ...

	sigChan := make(chan os.Signal, 1)
	signal.Notify(sigChan, syscall.SIGINT)

	go func() {
		<-sigChan
		cancel()
	}()

	var state *displayResponseState = &displayResponseState{}

	fn := func(response api.GenerateResponse) error {
		p.StopAndClear()

		latest = response
		content := response.Response

		displayResponse(content, opts.WordWrap, state)

		return nil
	}

	if opts.MultiModal {
		opts.Prompt, opts.Images, err = extractFileData(opts.Prompt)
		if err != nil {
			return err
		}
	}

	request := api.GenerateRequest{
		Model:     opts.Model,
		Prompt:    opts.Prompt,
		Context:   generateContext,
		Images:    opts.Images,
		Format:    opts.Format,
		System:    opts.System,
		Options:   opts.Options,
		KeepAlive: opts.KeepAlive,
	}

        // 让 server 处理， 得到返回值后调用 fn
	if err := client.Generate(ctx, &request, fn); err != nil {
		if errors.Is(err, context.Canceled) {
			return nil
		}
		return err
	}
        // ...
	return nil
}

```

# Go server
与 APIs 通信的 server 代码都在 `server/*` 文件夹下，它使用了 Gin 作为 server 端
```go
func Serve(ln net.Listener) error {
        // 日志
	level := slog.LevelInfo
        // ...
        // 二进制文件夹
	blobsDir, err := GetBlobsPath("")
        // ...
	ctx, done := context.WithCancel(context.Background())
	schedCtx, schedDone := context.WithCancel(ctx)
        // 调度器，加载模型创建 llm server ，处理请求
	sched := InitScheduler(schedCtx)
	s := &Server{addr: ln.Addr(), sched: sched}
        // 路由注册
	http.Handle("/", s.GenerateRoutes())

	slog.Info(fmt.Sprintf("Listening on %s (version %s)", ln.Addr(), version.Version))
	srvr := &http.Server{
		// Use http.DefaultServeMux so we get net/http/pprof for
		// free.
		//
		// TODO(bmizerany): Decide if we want to make this
		// configurable so it is not exposed by default, or allow
		// users to bind it to a different port. This was a quick
		// and easy way to get pprof, but it may not be the best
		// way.
		Handler: nil,
	}
        // ...

        // 启动 llm server
	if err := llm.Init(); err != nil {
		return fmt.Errorf("unable to initialize llm library %w", err)
	}
        // 启两个协程，processPending 和 processCompleted
        // processPending 待处理请求，确定是否需要加载新的模型或使用已加载的模型。它会根据当前的GPU使用情况和模型加载状态，决定是否需要卸载某个已加载的模型以腾出空间。
        // processCompleted 已完成的请求，更新模型的引用计数，并根据模型的会话持续时间决定是否设置一个过期定时器以便在会话结束后卸载模型。
	s.sched.Run(schedCtx)

	// At startup we retrieve GPU information so we can get log messages before loading a model
	// This will log warnings to the log in case we have problems with detected GPUs
	gpus := gpu.GetGPUInfo()
	gpus.LogDetails()

	err = srvr.Serve(ln)
	// If server is closed from the signal handler, wait for the ctx to be done
	// otherwise error out quickly
	if !errors.Is(err, http.ErrServerClosed) {
		return err
	}
	<-ctx.Done()
	return nil
}
```
接下来我们还是以 create 和 run 为例
```go
func (s *Server) CreateModelHandler(c *gin.Context) {
	var r api.CreateRequest
        // ...
	f, err := parser.ParseFile(sr)
	if err != nil {
		c.AbortWithStatusJSON(http.StatusBadRequest, gin.H{"error": err.Error()})
		return
	}

	ch := make(chan any)
	go func() {
		defer close(ch)
		fn := func(resp api.ProgressResponse) {
			ch <- resp
		}

		ctx, cancel := context.WithCancel(c.Request.Context())
		defer cancel()

		quantization := cmp.Or(r.Quantize, r.Quantization)
                // 创建模型
		if err := CreateModel(ctx, name, filepath.Dir(r.Path), strings.ToUpper(quantization), f, fn); err != nil {
			if errors.Is(err, errBadTemplate) {
			  ch <- gin.H{"error": err.Error(), "status": http.StatusBadRequest}
			}
			ch <- gin.H{"error": err.Error()}
		  }
	}()

	if r.Stream != nil && !*r.Stream {
		waitForStream(c, ch)
		return
	}

	streamResponse(c, ch)
}

// 解析 modelfile 创建每一层，每层生成一个 hash 二进制，生成一个 manifest 记录信息
func CreateModel(ctx context.Context, name model.Name, modelFileDir, quantization string, modelfile *parser.File, fn func(resp api.ProgressResponse)) (err error) {
	config := ConfigV2{
		OS:           "linux",
		Architecture: "amd64",
		RootFS: RootFS{
			Type: "layers",
		},
	}

	var messages []*api.Message
	parameters := make(map[string]any)

	var layers []*Layer
	for _, c := range modelfile.Commands {
		mediatype := fmt.Sprintf("application/vnd.ollama.image.%s", c.Name)

		switch c.Name {
		case "model", "adapter":
                        // ...
		case "license", "template", "system":
                        // ...
		case "message":
			role, content, ok := strings.Cut(c.Args, ": ")
			if !ok {
				return fmt.Errorf("invalid message: %s", c.Args)
			}

			messages = append(messages, &api.Message{Role: role, Content: content})
		default:
			ps, err := api.FormatParams(map[string][]string{c.Name: {c.Args}})
			if err != nil {
				return err
			}

			for k, v := range ps {
				if ks, ok := parameters[k].([]string); ok {
					parameters[k] = append(ks, v.([]string)...)
				} else if vs, ok := v.([]string); ok {
					parameters[k] = vs
				} else {
					parameters[k] = v
				}
			}
		}
	}

	var err2 error
	layers = slices.DeleteFunc(layers, func(layer *Layer) bool {
                // ...
	})

	if err2 != nil {
		return err2
	}

	if len(messages) > 0 {
		var b bytes.Buffer
		if err := json.NewEncoder(&b).Encode(messages); err != nil {
			return err
		}

		layer, err := NewLayer(&b, "application/vnd.ollama.image.messages")
		if err != nil {
			return err
		}

		layers = append(layers, layer)
	}

	if len(parameters) > 0 {
		var b bytes.Buffer
		if err := json.NewEncoder(&b).Encode(parameters); err != nil {
			return err
		}

		layer, err := NewLayer(&b, "application/vnd.ollama.image.params")
		if err != nil {
			return err
		}

		layers = append(layers, layer)
	}

	digests := make([]string, len(layers))
	for i, layer := range layers {
		digests[i] = layer.Digest
	}

	config.RootFS.DiffIDs = digests

	var b bytes.Buffer
	if err := json.NewEncoder(&b).Encode(config); err != nil {
		return err
	}

	layer, err := NewLayer(&b, "application/vnd.docker.container.image.v1+json")
	if err != nil {
		return err
	}

	for _, layer := range append(layers, layer) {
		if layer.status != "" {
			fn(api.ProgressResponse{Status: layer.status})
		}
	}

	old, _ := ParseNamedManifest(name)

	fn(api.ProgressResponse{Status: "writing manifest"})
	if err := WriteManifest(name, layer, layers); err != nil {
		return err
	}

	if !envconfig.NoPrune && old != nil {
		if err := old.RemoveLayers(); err != nil {
			return err
		}
	}

	fn(api.ProgressResponse{Status: "success"})
	return nil
}
```
create 逻辑比较简单，即解析 modelfile 后，形成层结构，每一层保存，通过 manifest 记录
```go

func (s *Server) GenerateHandler(c *gin.Context) {
	checkpointStart := time.Now()
	var req api.GenerateRequest
        // ...
	caps := []Capability{CapabilityCompletion}
	if req.Suffix != "" {
		caps = append(caps, CapabilityInsert)
	}

        // 调度一个 llm.LlamaServer 实例
	r, m, opts, err := s.scheduleRunner(c.Request.Context(), req.Model, caps, req.Options, req.KeepAlive)
	if errors.Is(err, errCapabilityCompletion) {
		c.JSON(http.StatusBadRequest, gin.H{"error": fmt.Sprintf("%q does not support generate", req.Model)})
		return
	} else if err != nil {
		handleScheduleError(c, req.Model, err)
		return
	}
        // ...

	images := make([]llm.ImageData, len(req.Images))
	for i := range req.Images {
		images[i] = llm.ImageData{ID: i, Data: req.Images[i]}
	}
        
        // 生成提示词
	prompt := req.Prompt
	if !req.Raw {
		tmpl := m.Template
                // ...
                // 用了 template 包处理
		prompt = b.String()
	}

	slog.Debug("generate request", "prompt", prompt, "images", images)

        // 跟 LlamaServer  通信，获取 Completion 结果
	ch := make(chan any)
	go func() {
		// TODO (jmorganca): avoid building the response twice both here and below
		var sb strings.Builder
		defer close(ch)
		if err := r.Completion(c.Request.Context(), llm.CompletionRequest{
			Prompt:  prompt,
			Images:  images,
			Format:  req.Format,
			Options: opts,
		}, func(cr llm.CompletionResponse) {
			res := api.GenerateResponse{
				Model:      req.Model,
				CreatedAt:  time.Now().UTC(),
				Response:   cr.Content,
				Done:       cr.Done,
				DoneReason: cr.DoneReason,
				Metrics: api.Metrics{
					PromptEvalCount:    cr.PromptEvalCount,
					PromptEvalDuration: cr.PromptEvalDuration,
					EvalCount:          cr.EvalCount,
					EvalDuration:       cr.EvalDuration,
				},
			}

			if _, err := sb.WriteString(cr.Content); err != nil {
				ch <- gin.H{"error": err.Error()}
			}

			if cr.Done {
				res.TotalDuration = time.Since(checkpointStart)
				res.LoadDuration = checkpointLoaded.Sub(checkpointStart)

				if !req.Raw {
					tokens, err := r.Tokenize(c.Request.Context(), prompt+sb.String())
					if err != nil {
						ch <- gin.H{"error": err.Error()}
						return
					}
					res.Context = append(req.Context, tokens...)
				}
			}

			ch <- res
		}); err != nil {
			ch <- gin.H{"error": err.Error()}
		}
	}()

        // 将 response 返回
	if req.Stream != nil && !*req.Stream {
		var r api.GenerateResponse
		var sb strings.Builder
		for rr := range ch {
			switch t := rr.(type) {
			case api.GenerateResponse:
				sb.WriteString(t.Response)
				r = t
			case gin.H:
				msg, ok := t["error"].(string)
				if !ok {
					msg = "unexpected error format in response"
				}

				c.JSON(http.StatusInternalServerError, gin.H{"error": msg})
				return
			default:
				c.JSON(http.StatusInternalServerError, gin.H{"error": "unexpected response"})
				return
			}
		}

		r.Response = sb.String()
		c.JSON(http.StatusOK, r)
		return
	}

	streamResponse(c, ch)
}

func (s *Scheduler) processPending(ctx context.Context) {
	for {
		select {
		case <-ctx.Done():
			// 如果上下文被取消，记录日志并退出循环
			slog.Debug("shutting down scheduler pending loop")
			return
		case pending := <-s.pendingReqCh:
			// 从挂起请求通道中获取挂起请求
			// 阻止其他请求，直到这个挂起请求被处理
			pending.schedAttempts++
			if pending.origNumCtx == 0 {
				pending.origNumCtx = pending.opts.NumCtx
			}

			// 检查请求的上下文是否已取消
			if pending.ctx.Err() != nil {
				slog.Debug("pending request cancelled or timed out, skipping scheduling")
				continue
			}
			numParallel := envconfig.NumParallel
			// TODO (jmorganca): multimodal models don't support parallel yet
			// see https://github.com/ollama/ollama/issues/4165
			if len(pending.model.ProjectorPaths) > 0 && numParallel != 1 {
				numParallel = 1
				slog.Warn("multimodal models don't support parallel requests yet")
			}

			// 内部循环，尝试加载模型并处理请求
			for {
				var runnerToExpire *runnerRef
				s.loadedMu.Lock()
				runner := s.loaded[pending.model.ModelPath]
				loadedCount := len(s.loaded)
				s.loadedMu.Unlock()
				if runner != nil {
					// 如果运行器存在且需要重新加载，将其标记为需要过期
					if runner.needsReload(ctx, pending) {
						runnerToExpire = runner
					} else {
						// 如果运行器可用，使用它处理请求
						pending.useLoadedRunner(runner, s.finishedReqCh)
						break
					}
				} else if envconfig.MaxRunners > 0 && loadedCount >= envconfig.MaxRunners {
					// 如果已达到最大运行器数量，尝试卸载一个现有的运行器
					slog.Debug("max runners achieved, unloading one to make room", "runner_count", loadedCount)
					runnerToExpire = s.findRunnerToUnload()
				} else {
					// 获取最新的 GPU 列表
					var gpus gpu.GpuInfoList
					if pending.opts.NumGPU == 0 {
						gpus = s.getCpuFn()
					} else {
						gpus = s.getGpuFn()
					}

					// 根据系统内存情况决定是否卸载其他模型以腾出空间
					if envconfig.MaxRunners <= 0 {
						// 如果未指定 MaxRunners，根据 GPU 的可靠性自动设置
						allReliable := true
						for _, gpu := range gpus {
							if gpu.UnreliableFreeMemory {
								allReliable = false
								break
							}
						}
						if allReliable {
							envconfig.MaxRunners = defaultModelsPerGPU * len(gpus)
							slog.Debug("updating default concurrency", "OLLAMA_MAX_LOADED_MODELS", envconfig.MaxRunners, "gpu_count", len(gpus))
						} else {
							slog.Info("one or more GPUs detected that are unable to accurately report free memory - disabling default concurrency")
							envconfig.MaxRunners = len(gpus)
						}
					}

					// 加载模型
					ggml, err := llm.LoadModel(pending.model.ModelPath, 0)
					if err != nil {
						pending.errCh <- err
						break
					}

					// 检查模型是否适合在系统内存中加载，或者是否需要先卸载一个模型
					if len(gpus) == 1 && gpus[0].Library == "cpu" {
						if numParallel <= 0 {
							numParallel = defaultParallel
						}

						pending.opts.NumCtx = pending.origNumCtx * numParallel

						if loadedCount == 0 {
							slog.Debug("cpu mode with first model, loading")
							s.loadFn(pending, ggml, gpus, numParallel)
							break
						}
						runnerToExpire = s.maybeFindCPURunnerToUnload(pending, ggml, gpus)
						if runnerToExpire == nil {
							slog.Debug("cpu mode with available system memory or first model, loading")
							s.loadFn(pending, ggml, gpus, numParallel)
							break
						}
					} else if loadedCount == 0 {
						// 如果没有加载的模型，加载新模型并选择最佳适配的 GPU
						slog.Debug("loading first model", "model", pending.model.ModelPath)
						g := pickBestFitGPUs(pending, ggml, gpus, &numParallel)
						if g != nil {
							gpus = g
						}
						s.loadFn(pending, ggml, gpus, numParallel)
						break
					}

					if runnerToExpire == nil {
						// 检查新模型是否适合在现有的 GPU 上加载
						availGpus := s.filterGPUsWithoutLoadingModels(gpus)
						s.updateFreeSpace(availGpus)
						fitGpus := pickBestFitGPUs(pending, ggml, availGpus, &numParallel)
						if fitGpus != nil {
							slog.Debug("new model fits with existing models, loading")
							s.loadFn(pending, ggml, fitGpus, numParallel)
							break
						}

						// 如果找不到适合的新 GPU，推迟请求处理并重新排队
						if len(availGpus) < len(gpus) {
							go func() {
								slog.Debug("delaying scheduling while other models finish loading", "attempts", pending.schedAttempts, "model", pending.model.ModelPath)
								time.Sleep(s.reschedDelay)
								s.pendingReqCh <- pending
							}()
							break
						}
						runnerToExpire = s.findRunnerToUnload()
					}
				}

				if runnerToExpire == nil {
					slog.Error("runner to expire was nil!")
					continue
				}
				// 触发过期以卸载运行器
				runnerToExpire.refMu.Lock()
				slog.Debug("resetting model to expire immediately to make room", "modelPath", runnerToExpire.modelPath, "refCount", runnerToExpire.refCount)
				if runnerToExpire.expireTimer != nil {
					runnerToExpire.expireTimer.Stop()
					runnerToExpire.expireTimer = nil
				}
				runnerToExpire.sessionDuration = 0
				if runnerToExpire.refCount <= 0 {
					s.expiredCh <- runnerToExpire
				}
				runnerToExpire.refMu.Unlock()
				// 等待卸载完成
				slog.Debug("waiting for pending requests to complete and unload to occur", "modelPath", runnerToExpire.modelPath)
				select {
				case <-ctx.Done():
					slog.Debug("shutting down scheduler pending loop")
					return
				case <-s.unloadedCh:
					slog.Debug("unload completed", "modelPath", runnerToExpire.modelPath)
					continue
				}
			}
		case <-s.unloadedCh:
			// 如果没有挂起请求，忽略卸载事件
			slog.Debug("ignoring unload event with no pending requests")
		}
	}
}

```
run 主要是根据用户的 req 使用调度器，获取到对应的 llm server ，将 llm server 生成的数据返回


# llm server

ollama 中运行 LLM 的核心是 [llama.cpp](https://github.com/ggerganov/llama.cpp)，是开发者 Georgi Gerganov 基于 Meta 的 LLaMA 模型 手写的纯 C/C++ 版本：支持CPU推理,当然也支持CUDA/OpenCL推理、具有 FP16 和 FP32 的混合精度、支持8-bit/4bit量化...

如果想更深入了解可以看看[Llama 2详解](https://zhuanlan.zhihu.com/p/649756898)和[llama.cpp源码解析--CUDA流程版本](https://zhuanlan.zhihu.com/p/665027154)

本文仅分析 ollama 中如何使用。

`llama.cpp` 与常见 C 或 C++ 项目一样使用 `cmake` 来处理编译、链接等工作。但是 `ollama` 本身是一个 go 项目，使用的是 go 构建系统编译、链接和打包其余部分，以生成 `ollama` 的应用程序和 cli。

通过利用 [go generate](https://go.dev/blog/generate)，让 go 编译系统还可以运行调用 cmake 的命令来构建 `llama.cpp`。代码位于 `llm/generate/*` 下，比如 macos：
```go
package generate

//go:generate bash ./gen_darwin.sh

```

```bash
# gen_common.sh
# common logic across linux and darwin
init_vars() {
    case "${GOARCH}" in
    "amd64")
        ARCH="x86_64"
        ;;
    "arm64")
        ARCH="arm64"
        ;;
    *)
        ARCH=$(uname -m | sed -e "s/aarch64/arm64/g")
    esac
    # 设置 LLAMACPP 目录和 CMake 默认定义
    LLAMACPP_DIR=../llama.cpp
    CMAKE_DEFS=""
    CMAKE_TARGETS="--target ollama_llama_server"

    # 根据 CGO_CFLAGS 中是否包含 -g 设置 CMake 构建类型
    if echo "${CGO_CFLAGS}" | grep -- '-g' >/dev/null; then
        CMAKE_DEFS="-DCMAKE_BUILD_TYPE=RelWithDebInfo -DCMAKE_VERBOSE_MAKEFILE=on -DLLAMA_GPROF=on -DLLAMA_SERVER_VERBOSE=on ${CMAKE_DEFS}"
    else
        CMAKE_DEFS="-DCMAKE_BUILD_TYPE=Release -DLLAMA_SERVER_VERBOSE=off ${CMAKE_DEFS}"
    fi

    # 根据系统类型设置库扩展名和编译选项
    case $(uname -s) in
    "Darwin")
        LIB_EXT="dylib"
        WHOLE_ARCHIVE="-Wl,-force_load"
        NO_WHOLE_ARCHIVE=""
        GCC_ARCH="-arch ${ARCH}"
        ;;
    "Linux")
        LIB_EXT="so"
        WHOLE_ARCHIVE="-Wl,--whole-archive"
        NO_WHOLE_ARCHIVE="-Wl,--no-whole-archive"
        GCC_ARCH=""
        ;;
    *)
        ;;
    esac

    # 设置 CUDA 架构
    if [ -z "${CMAKE_CUDA_ARCHITECTURES}" ] ; then
        CMAKE_CUDA_ARCHITECTURES="50;52;61;70;75;80"
    fi
}

git_module_setup() {
    if [ -n "${OLLAMA_SKIP_PATCHING}" ]; then
        echo "Skipping submodule initialization"
        return
    fi

    # 如果存在旧的子模块目录，先删除
    if [ -d "${LLAMACPP_DIR}/gguf" ]; then
        echo "Cleaning up old submodule"
        rm -rf ${LLAMACPP_DIR}
    fi

    # 初始化并更新子模块
    git submodule init
    git submodule update --force ${LLAMACPP_DIR}
}

apply_patches() {
    # 修改 CMakeLists.txt 文件，添加子目录
    if ! grep ollama ${LLAMACPP_DIR}/CMakeLists.txt; then
        echo 'add_subdirectory(../ext_server ext_server) # ollama' >>${LLAMACPP_DIR}/CMakeLists.txt
    fi

    # 如果存在补丁文件，应用这些补丁
    if [ -n "$(ls -A ../patches/*.diff)" ]; then
        for patch in ../patches/*.diff; do
            for file in $(grep "^+++ " ${patch} | cut -f2 -d' ' | cut -f2- -d/); do
                (cd ${LLAMACPP_DIR}; git checkout ${file})
            done
        done
        for patch in ../patches/*.diff; do
            (cd ${LLAMACPP_DIR} && git apply ${patch})
        done
    fi
}


build() {
    cmake -S ${LLAMACPP_DIR} -B ${BUILD_DIR} ${CMAKE_DEFS}
    cmake --build ${BUILD_DIR} ${CMAKE_TARGETS} -j8
}
compress() {
    echo "Compressing payloads to reduce overall binary size..."
    pids=""
    rm -rf ${BUILD_DIR}/bin/*.gz
    for f in ${BUILD_DIR}/bin/* ; do
        gzip -n --best -f ${f} &
        pids+=" $!"
    done
    # 检查 lib 目录并压缩其中的文件
    if [ -d ${BUILD_DIR}/lib ]; then
        for f in ${BUILD_DIR}/lib/* ; do
            gzip -n --best -f ${f} &
            pids+=" $!"
        done
    fi
    echo
    for pid in ${pids}; do
        wait $pid
    done
    echo "Finished compression"
}


# Keep the local tree clean after we're done with the build
cleanup() {
    (cd ${LLAMACPP_DIR}/ && git checkout CMakeLists.txt)

    if [ -n "$(ls -A ../patches/*.diff)" ]; then
        for patch in ../patches/*.diff; do
            for file in $(grep "^+++ " ${patch} | cut -f2 -d' ' | cut -f2- -d/); do
                (cd ${LLAMACPP_DIR}; git checkout ${file})
            done
        done
    fi
}

# gen_darwin.sh
#!/bin/bash
# This script is intended to run inside the go generate
# working directory must be ./llm/generate/

# TODO - add hardening to detect missing tools (cmake, etc.)

set -ex
# 打开错误检测和命令显示模式，以及命令管道失败检测
set -o pipefail
echo "Starting darwin generate script"
# 引入通用的生成脚本
source $(dirname $0)/gen_common.sh
# 初始化变量
init_vars
# 设置 Git 子模块
git_module_setup
# 应用补丁
apply_patches

# 签名函数，如果 APPLE_IDENTITY 环境变量被设置，就会对生成的文件进行代码签名
sign() {
    if [ -n "$APPLE_IDENTITY" ]; then
        codesign -f --timestamp --deep --options=runtime --sign "$APPLE_IDENTITY" --identifier ai.ollama.ollama $1
    fi
}

# 定义通用的 macOS 构建选项
COMMON_DARWIN_DEFS="-DBUILD_SHARED_LIBS=off -DCMAKE_OSX_DEPLOYMENT_TARGET=11.3 -DLLAMA_METAL_MACOSX_VERSION_MIN=11.3 -DCMAKE_SYSTEM_NAME=Darwin -DGGML_METAL_EMBED_LIBRARY=on -DGGML_OPENMP=off"

# 根据 GOARCH 变量选择构建架构
case "${GOARCH}" in
"amd64")
    # 定义 amd64 架构的通用选项
    COMMON_CPU_DEFS="${COMMON_DARWIN_DEFS} -DCMAKE_SYSTEM_PROCESSOR=${ARCH} -DCMAKE_OSX_ARCHITECTURES=${ARCH} -DGGML_METAL=off -DGGML_NATIVE=off"

    # 静态链接构建，用于 Go 二进制文件的链接
    init_vars
    CMAKE_TARGETS="--target llama --target ggml"
    CMAKE_DEFS="${COMMON_CPU_DEFS} -DGGML_BLAS=off -DGGML_ACCELERATE=off -DGGML_AVX=off -DGGML_AVX2=off -DGGML_AVX512=off -DGGML_FMA=off -DGGML_F16C=off ${CMAKE_DEFS}"
    BUILD_DIR="../build/darwin/${ARCH}_static"
    echo "Building static library"
    build

    if [ -z "$OLLAMA_SKIP_CPU_GENERATE" ]; then
        # 构建最低要求 CPU 版本，确保最大兼容性（包括 Rosetta）
        init_vars
        CMAKE_DEFS="${COMMON_CPU_DEFS} -DGGML_ACCELERATE=off -DGGML_BLAS=off -DGGML_AVX=off -DGGML_AVX2=off -DGGML_AVX512=off -DGGML_FMA=off -DGGML_F16C=off ${CMAKE_DEFS}"
        BUILD_DIR="../build/darwin/${ARCH}/cpu"
        echo "Building LCD CPU"
        build
        sign ${BUILD_DIR}/bin/ollama_llama_server
        compress

        # 构建支持 AVX 指令集的版本，提高性能
        init_vars
        CMAKE_DEFS="${COMMON_CPU_DEFS} -DGGML_ACCELERATE=off -DGGML_BLAS=off -DGGML_AVX=on -DGGML_AVX2=off -DGGML_AVX512=off -DGGML_FMA=off -DGGML_F16C=off ${CMAKE_DEFS}"
        BUILD_DIR="../build/darwin/${ARCH}/cpu_avx"
        echo "Building AVX CPU"
        build
        sign ${BUILD_DIR}/bin/ollama_llama_server
        compress

        # 构建支持 AVX2 和其他指令集的版本，进一步提高性能
        init_vars
        CMAKE_DEFS="${COMMON_CPU_DEFS} -DGGML_ACCELERATE=on -DGGML_BLAS=off -DGGML_AVX=on -DGGML_AVX2=on -DGGML_AVX512=off -DGGML_FMA=on -DGGML_F16C=on ${CMAKE_DEFS}"
        BUILD_DIR="../build/darwin/${ARCH}/cpu_avx2"
        echo "Building AVX2 CPU"
        EXTRA_LIBS="${EXTRA_LIBS} -framework Accelerate -framework Foundation"
        build
        sign ${BUILD_DIR}/bin/ollama_llama_server
        compress
    fi
    ;;
"arm64")
    # 静态链接构建，用于 Go 二进制文件的链接
    init_vars
    CMAKE_TARGETS="--target llama --target ggml"
    CMAKE_DEFS="${COMMON_DARWIN_DEFS} -DCMAKE_OSX_DEPLOYMENT_TARGET=11.3 -DCMAKE_SYSTEM_NAME=Darwin -DCMAKE_SYSTEM_PROCESSOR=${ARCH} -DCMAKE_OSX_ARCHITECTURES=${ARCH} ${CMAKE_DEFS}"
    BUILD_DIR="../build/darwin/${ARCH}_static"
    echo "Building static library"
    build

    if [ -z "$OLLAMA_SKIP_METAL_GENERATE" ]; then
        # 构建支持 Metal 框架的版本，提高性能
        init_vars
        CMAKE_DEFS="${COMMON_DARWIN_DEFS} -DCMAKE_SYSTEM_PROCESSOR=${ARCH} -DCMAKE_OSX_ARCHITECTURES=${ARCH} ${CMAKE_DEFS}"
        BUILD_DIR="../build/darwin/${ARCH}/metal"
        EXTRA_LIBS="${EXTRA_LIBS} -framework Accelerate -framework Foundation -framework Metal -framework MetalKit -framework MetalPerformanceShaders"
        build
        sign ${BUILD_DIR}/bin/ollama_llama_server
        compress
    fi
    ;;
*)
    # 如果未设置 GOARCH 或设置了不支持的值，输出错误并退出
    echo "GOARCH must be set"
    echo "this script is meant to be run from within go generate"
    exit 1
    ;;
esac

# 清理临时文件和目录
cleanup
echo "go generate completed.  LLM runners: $(cd ${BUILD_DIR}/..; echo *)"

```

简单来说，就是
- 初始化变量，为编译做准备
- CMAKE_TARGETS 将把构建目标设置为 ext_server
- 执行 build 调用 cmake

cmake 编译后，它将生成一个 ollama_llama_server 动态链接库（.dll/.so/.dylib）,将作为可执行二进制文件嵌入 go 程序。

我们先看下 ollama_llama_server 做了什么
```cpp

int main(int argc, char **argv) {
    // own arguments required by this example
    gpt_params params;
    server_params sparams;

    // struct that contains llama context and inference
    llama_server_context llama;

    server_params_parse(argc, argv, sparams, params);

    if (params.model_alias == "unknown")
    {
        params.model_alias = params.model;
    }

    // 初始化 AVX/CUDA ...
    llama_backend_init();
    // 初始化 NUMA
    llama_numa_init(params.numa);

    httplib::Server svr;

    std::atomic<server_state> state{SERVER_STATE_LOADING_MODEL};

    svr.set_default_headers({{"Server", "llama.cpp"}});

    // CORS preflight
    svr.Options(R"(.*)", [](const httplib::Request &req, httplib::Response &res) {
        res.set_header("Access-Control-Allow-Origin", req.get_header_value("Origin"));
        res.set_header("Access-Control-Allow-Credentials", "true");
        res.set_header("Access-Control-Allow-Methods", "POST");
        res.set_header("Access-Control-Allow-Headers", "*");
    });

    svr.Get("/health", [&](const httplib::Request& req, httplib::Response& res) {
        // ...
    });

    // ... 错误中间件注册

    if (!svr.bind_to_port(sparams.hostname, sparams.port))
    {
        fprintf(stderr, "\ncouldn't bind to server socket: hostname=%s port=%d\n\n", sparams.hostname.c_str(), sparams.port);
        return 1;
    }

    // Set the base directory for serving static files
    svr.set_base_dir(sparams.public_path);
    
    // ... 日志设置

    // run the HTTP server in a thread - see comment below
    std::thread t([&]()
            {
                if (!svr.listen_after_bind())
                {
                    state.store(SERVER_STATE_ERROR);
                    return 1;
                }

                return 0;
            });

    // load the model
    params.progress_callback = update_load_progress;
    params.progress_callback_user_data = (void*)&llama;

    // 使用参数加载模型
    if (!llama.load_model(params))
    {
        state.store(SERVER_STATE_ERROR);
        return 1;
    } else {
        // 初始化
        llama.initialize();
        state.store(SERVER_STATE_READY);
        LOG_INFO("model loaded", {});
    }
    const auto model_meta = llama.model_meta();

    // ... api key 中间件

    svr.Post("/completion", [&llama, &validate_api_key](const httplib::Request &req, httplib::Response &res)
            {
                res.set_header("Access-Control-Allow-Origin", req.get_header_value("Origin"));
                if (!validate_api_key(req, res)) {
                    return;
                }
                json data = json::parse(req.body);
                // 创建新的任务
                const int task_id = llama.queue_tasks.get_new_id();
                llama.queue_results.add_waiting_task_id(task_id);
                llama.request_completion(task_id, data, false, -1);
                if (!json_value(data, "stream", false)) {
                // 非流式处理
                    std::string completion_text;
                    // 等待任务结果 
                    task_result result = llama.queue_results.recv(task_id);
                    if (!result.error && result.stop) {
                        res.set_content(result.result_json.dump(-1, ' ', false, json::error_handler_t::replace), "application/json; charset=utf-8");
                    }
                    else
                    {
                        res.status = 404;
                        res.set_content(result.result_json["content"], "text/plain; charset=utf-8");
                    }
                    llama.queue_results.remove_waiting_task_id(task_id);
                } else {
                    const auto chunked_content_provider = [task_id, &llama](size_t, httplib::DataSink & sink)
                    {
                        while (true)
                        {
                            task_result result = llama.queue_results.recv(task_id);
                            if (!result.error) {
                                const std::string str =
                                    "data: " +
                                    result.result_json.dump(-1, ' ', false, json::error_handler_t::replace) +
                                    "\n\n";
                                LOG_VERBOSE("data stream", {
                                    { "to_send", str }
                                });
                                if (!sink.write(str.c_str(), str.size()))
                                {
                                    llama.queue_results.remove_waiting_task_id(task_id);
                                    return false;
                                }
                                if (result.stop) {
                                    break;
                                }
                            } else {
                                const std::string str =
                                    "error: " +
                                    result.result_json.dump(-1, ' ', false, json::error_handler_t::replace) +
                                    "\n\n";
                                LOG_VERBOSE("data stream", {
                                    { "to_send", str }
                                });
                                if (!sink.write(str.c_str(), str.size()))
                                {
                                    llama.queue_results.remove_waiting_task_id(task_id);
                                    return false;
                                }
                                break;
                            }
                        }

                        llama.queue_results.remove_waiting_task_id(task_id);
                        sink.done();
                        return true;
                    };

                    auto on_complete = [task_id, &llama] (bool)
                    {
                        // cancel
                        llama.request_cancel(task_id);
                        llama.queue_results.remove_waiting_task_id(task_id);
                    };

                    res.set_chunked_content_provider("text/event-stream", chunked_content_provider, on_complete);
                }
            });

    // ... 路由注册

    // 新任务加入队列时 process_single_task 处理该任务
    llama.queue_tasks.on_new_task(std::bind(
        &llama_server_context::process_single_task, &llama, std::placeholders::_1));
    // 多任务处理完成 on_finish_multitask 处理相关逻辑
    llama.queue_tasks.on_finish_multitask(std::bind(
        &llama_server_context::on_finish_multitask, &llama, std::placeholders::_1));
    llama.queue_tasks.on_run_slots(std::bind(
        &llama_server_context::update_slots, &llama));
    // 多任务更新
    llama.queue_results.on_multitask_update(std::bind(
        &llama_server_queue::update_multitask,
        &llama.queue_tasks,
        std::placeholders::_1,
        std::placeholders::_2,
        std::placeholders::_3
    ));

    shutdown_handler = [&](int) {
        llama.queue_tasks.terminate();
    };

    // ...

    // llama 开始处理列队信息
    llama.queue_tasks.start_loop();
    svr.stop();
    t.join();

    llama_backend_free();
    return 0;
}
```
接下来来看看 go 的部分
```go
func NewLlamaServer(gpus gpu.GpuInfoList, model string, ggml *GGML, adapters, projectors []string, opts api.Options, numParallel int) (LlamaServer, error) {
	var err error
	var cpuRunner string
	var estimate MemoryEstimate
	var systemTotalMemory uint64
	var systemFreeMemory uint64
	var systemSwapFreeMemory uint64

	systemMemInfo, err := gpu.GetCPUMem()
	if err != nil {
		slog.Error("failed to lookup system memory", "error", err)
	} else {
		systemTotalMemory = systemMemInfo.TotalMemory
		systemFreeMemory = systemMemInfo.FreeMemory
		systemSwapFreeMemory = systemMemInfo.FreeSwap
		slog.Debug("system memory", "total", format.HumanBytes2(systemTotalMemory), "free", format.HumanBytes2(systemFreeMemory), "free_swap", format.HumanBytes2(systemSwapFreeMemory))
	}

	// ... gpu 信息获取

        // 获取可用的 llm server
	availableServers := getAvailableServers()
	if len(availableServers) == 0 {
		if runtime.GOOS != "windows" {
			slog.Warn("llama server binary disappeared, reinitializing payloads")
			err = Init()
			if err != nil {
				slog.Warn("failed to reinitialize payloads", "error", err)
				return nil, err
			}
			availableServers = getAvailableServers()
		} else {
			return nil, finalErr
		}
	}
	// 根据 CPU/GPU 选择合适的 server path
	var servers []string
	if cpuRunner != "" {
		servers = []string{cpuRunner}
	} else {
		servers = serversForGpu(gpus[0]) // All GPUs in the list are matching Library and Variant
	}
	demandLib := envconfig.LLMLibrary
	if demandLib != "" {
		serverPath := availableServers[demandLib]
		if serverPath == "" {
			slog.Info(fmt.Sprintf("Invalid OLLAMA_LLM_LIBRARY %s - not found", demandLib))
		} else {
			slog.Info("user override", "OLLAMA_LLM_LIBRARY", demandLib, "path", serverPath)
			servers = []string{demandLib}
			if strings.HasPrefix(demandLib, "cpu") {
				// Omit the GPU flag to silence the warning
				opts.NumGPU = -1
			}
		}
	}

	params := []string{
		"--model", model,
		"--ctx-size", fmt.Sprintf("%d", opts.NumCtx),
		"--batch-size", fmt.Sprintf("%d", opts.NumBatch),
		"--embedding",
	}

        // .. params 各种拼接

	for i := range len(servers) {
		dir := availableServers[servers[i]]
                // ... 检查路径是否存在

		// 查找可用端口
		port := 0
		if a, err := net.ResolveTCPAddr("tcp", "localhost:0"); 
                // ...

                // 设置环境变量
		pathEnv := "LD_LIBRARY_PATH"
		// ...

		server := filepath.Join(dir, "ollama_llama_server")
		if runtime.GOOS == "windows" {
			server += ".exe"
		}

		// ...

		s := &llmServer{
			port:        port,
			cmd:         exec.Command(server, finalParams...),
			status:      NewStatusWriter(os.Stderr),
			options:     opts,
			estimate:    estimate,
			sem:         semaphore.NewWeighted(int64(numParallel)),
			totalLayers: ggml.KV().BlockCount() + 1,
			gpus:        gpus,
			done:        make(chan error, 1),
		}

		s.cmd.Env = os.Environ()
		s.cmd.Stdout = os.Stdout
		s.cmd.Stderr = s.status

                // .. 拼接 cmd

		if err = s.cmd.Start(); err != nil {
			// Detect permission denied and augment them essage about noexec
			if errors.Is(err, os.ErrPermission) {
				finalErr = fmt.Errorf("unable to start server %w.  %s may have noexec set.  Set OLLAMA_TMPDIR for server to a writable executable directory", err, dir)
				continue
			}
			msg := ""
			if s.status != nil && s.status.LastErrMsg != "" {
				msg = s.status.LastErrMsg
			}
			err = fmt.Errorf("error starting the external llama server: %v %s", err, msg)
			finalErr = err
			continue
		}

		// reap subprocess when it exits
		go func() {
			s.done <- s.cmd.Wait()
		}()

		return s, nil
	}

	slog.Error("unable to load any llama server", "error", finalErr)
	return nil, finalErr
}
```
最后我们来看看 Completion
```go

type CompletionRequest struct {
	Prompt  string
	Format  string
	Images  []ImageData
	Options *api.Options
}

type CompletionResponse struct {
	Content            string
	DoneReason         string
	Done               bool
	PromptEvalCount    int
	PromptEvalDuration time.Duration
	EvalCount          int
	EvalDuration       time.Duration
}

func (s *llmServer) Completion(ctx context.Context, req CompletionRequest, fn func(CompletionResponse)) error {
	if err := s.sem.Acquire(ctx, 1); err != nil {
		slog.Error("Failed to acquire semaphore", "error", err)
		return err
	}
	defer s.sem.Release(1)

	// 设置 NumPredict 上限，防止模型运行时间过长
	if req.Options.NumPredict < 0 || req.Options.NumPredict > 10*s.options.NumCtx {
		req.Options.NumPredict = 10 * s.options.NumCtx
	}

	request := map[string]any{
		"prompt":            req.Prompt,
		"stream":            true,
		"n_predict":         req.Options.NumPredict,
		"n_keep":            req.Options.NumKeep,
		"main_gpu":          req.Options.MainGPU,
		"temperature":       req.Options.Temperature,
		"top_k":             req.Options.TopK,
		"top_p":             req.Options.TopP,
		"tfs_z":             req.Options.TFSZ,
		"typical_p":         req.Options.TypicalP,
		"repeat_last_n":     req.Options.RepeatLastN,
		"repeat_penalty":    req.Options.RepeatPenalty,
		"presence_penalty":  req.Options.PresencePenalty,
		"frequency_penalty": req.Options.FrequencyPenalty,
		"mirostat":          req.Options.Mirostat,
		"mirostat_tau":      req.Options.MirostatTau,
		"mirostat_eta":      req.Options.MirostatEta,
		"penalize_nl":       req.Options.PenalizeNewline,
		"seed":              req.Options.Seed,
		"stop":              req.Options.Stop,
		"image_data":        req.Images,
		"cache_prompt":      true,
	}

	// 检查服务器状态
	status, err := s.getServerStatusRetry(ctx)
	if err != nil {
		return err
	} else if status != ServerStatusReady {
		return fmt.Errorf("unexpected server status: %s", status.ToString())
	}

	// json
	if req.Format == "json" {
		request["grammar"] = jsonGrammar
		if !strings.Contains(strings.ToLower(req.Prompt), "json") {
			slog.Warn("Prompt does not specify that the LLM should response in JSON, but JSON format is expected. For best results specify that JSON is expected in the system prompt.")
		}
	}

	// Handling JSON marshaling with special characters unescaped.
	buffer := &bytes.Buffer{}
	enc := json.NewEncoder(buffer)
	enc.SetEscapeHTML(false)

	if err := enc.Encode(request); err != nil {
		return fmt.Errorf("failed to marshal data: %v", err)
	}

	endpoint := fmt.Sprintf("http://127.0.0.1:%d/completion", s.port)
	serverReq, err := http.NewRequestWithContext(ctx, http.MethodPost, endpoint, buffer)
	if err != nil {
		return fmt.Errorf("error creating POST request: %v", err)
	}
	serverReq.Header.Set("Content-Type", "application/json")

	res, err := http.DefaultClient.Do(serverReq)
	if err != nil {
		return fmt.Errorf("POST predict: %v", err)
	}
	defer res.Body.Close()

	if res.StatusCode >= 400 {
		bodyBytes, err := io.ReadAll(res.Body)
		if err != nil {
			return fmt.Errorf("failed reading llm error response: %w", err)
		}
		log.Printf("llm predict error: %s", bodyBytes)
		return fmt.Errorf("%s", bodyBytes)
	}

	scanner := bufio.NewScanner(res.Body)
	buf := make([]byte, 0, maxBufferSize)
	scanner.Buffer(buf, maxBufferSize)

	// keep track of the last token generated, this is used to abort if the model starts looping
	var lastToken string
	var tokenRepeat int

	for scanner.Scan() {
		select {
		case <-ctx.Done():
			// This handles the request cancellation
			return ctx.Err()
		default:
			// 逐行读取响应数据
			line := scanner.Bytes()
			if len(line) == 0 {
				continue
			}

			evt, ok := bytes.CutPrefix(line, []byte("data: "))
			if !ok {
				return fmt.Errorf("error parsing llm response stream: %s", line)
			}

			var c completion
			if err := json.Unmarshal(evt, &c); err != nil {
				return fmt.Errorf("error unmarshalling llm prediction response: %v", err)
			}

			switch {
			case strings.TrimSpace(c.Content) == lastToken:
				tokenRepeat++
			default:
				lastToken = strings.TrimSpace(c.Content)
				tokenRepeat = 0
			}

			// 30 picked as an arbitrary max token repeat limit, modify as needed
			if tokenRepeat > 30 {
				slog.Debug("prediction aborted, token repeat limit reached")
				return ctx.Err()
			}

			// 解析并处理每行数据，将结果通过回调函数返回
			if c.Content != "" {
				fn(CompletionResponse{
					Content: c.Content,
				})
			}

			if c.Stop {
				doneReason := "stop"
				if c.StoppedLimit {
					doneReason = "length"
				}

				fn(CompletionResponse{
					Done:               true,
					DoneReason:         doneReason,
					PromptEvalCount:    c.Timings.PromptN,
					PromptEvalDuration: parseDurationMs(c.Timings.PromptMS),
					EvalCount:          c.Timings.PredictedN,
					EvalDuration:       parseDurationMs(c.Timings.PredictedMS),
				})
				return nil
			}
		}
	}

	if err := scanner.Err(); err != nil {
		if strings.Contains(err.Error(), "unexpected EOF") {
			s.Close()
			msg := ""
			if s.status != nil && s.status.LastErrMsg != "" {
				msg = s.status.LastErrMsg
			}
			return fmt.Errorf("an unknown error was encountered while running the model %s", msg)
		}

		return fmt.Errorf("error reading llm response: %v", err)
	}

	return nil
}
```