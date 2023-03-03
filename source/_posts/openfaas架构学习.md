---
title: openfaas架构学习
date: 2022-12-22 11:11:11
tags:
  - openfaas
  - serverless
  - golang
categories: 学习笔记
---

# 前言

Serverless 的核心思想是让作为计算资源的服务器不再成为用户所关注的一种资源。其目的是提高应用交付的效率，降低应用运营的工作量和成本。openfaas 是开源社区中比较有名的 Serverless 框架，本文主要通过剖析 openfaas 项目架构，从而进一步理解 Serverless

# 架构概览

官方架构图：
![architecture](/image/openfaas/architecture.png)
openfaas 主要的组件有以下四个：

1. Gateway
2. Provider
3. Queue-worker
4. Watchdog
   接下来我们依次分析一下他们的作用

# Gateway

https://github.com/openfaas/faas/tree/master/gateway
gateway 是整个项目的网关层，他的目的是为函数的调用提供一个路由代理转发的作用，并且内置了[ui 界面](https://github.com/openfaas/faas/tree/master/gateway/assets)，其中包含了 endpoints，prometheus 和 alertManager 三个部分。

- endpoints

endpoints 是主要处理 http 请求，提供了[安全验证](https://github.com/openfaas/faas/blob/4604271076a96ae70b4488afdc516f62fd802dd7/gateway/main.go#L53)

```golang
// 读取密钥

if config.UseBasicAuth {
                var readErr error
                reader := auth.ReadBasicAuthFromDisk{
                        SecretMountPath: config.SecretMountPath,
                }
                credentials, readErr = reader.Read()

                if readErr != nil {
                        log.Panicf(readErr.Error())
                }
        }

 // 使用中间件校验 https://github.com/openfaas/faas-provider/blob/9b2cd43cf3fee26a2952990cbfc33eb8511fd2b8/serve.go#L46
handlers.FunctionReader = auth.DecorateWithBasicAuth(handlers.FunctionReader, credentials)
handlers.DeployHandler = auth.DecorateWithBasicAuth(handlers.DeployHandler, credentials)
handlers.DeleteHandler = auth.DecorateWithBasicAuth(handlers.DeleteHandler, credentials)
handlers.UpdateHandler = auth.DecorateWithBasicAuth(handlers.UpdateHandler, credentials)
handlers.ReplicaReader = auth.DecorateWithBasicAuth(handlers.ReplicaReader, credentials)
handlers.ReplicaUpdater = auth.DecorateWithBasicAuth(handlers.ReplicaUpdater, credentials)
handlers.InfoHandler = auth.DecorateWithBasicAuth(handlers.InfoHandler, credentials)
handlers.SecretHandler = auth.DecorateWithBasicAuth(handlers.SecretHandler, credentials)
handlers.LogHandler = auth.DecorateWithBasicAuth(handlers.LogHandler, credentials)

// 中间件的实现 https://github.com/openfaas/faas-provider/blob/bc0f26cae70c1b3ba2a595cea8bfb8c04925d5c6/auth/basic_auth.go
func DecorateWithBasicAuth(next http.HandlerFunc, credentials *BasicAuthCredentials) http.HandlerFunc {
        return func(w http.ResponseWriter, r *http.Request) {

                user, password, ok := r.BasicAuth()

                const noMatch = 0
                if !ok ||
                        user != credentials.User ||
                        subtle.ConstantTimeCompare([]byte(credentials.Password), []byte(password)) == noMatch {

                        w.Header().Set("WWW-Authenticate", `Basic realm="Restricted"`)
                        w.WriteHeader(http.StatusUnauthorized)
                        w.Write([]byte("invalid credentials"))
                        return
                }

                next.ServeHTTP(w, r)
        }
}
```

[代理转发](https://github.com/openfaas/faas/blob/4604271076a96ae70b4488afdc516f62fd802dd7/gateway/main.go#L138)

```golang
faasHandlers.ListFunctions = handlers.MakeForwardingProxyHandler(reverseProxy, forwardingNotifiers, urlResolver, nilURLTransformer, serviceAuthInjector)
faasHandlers.DeployFunction = handlers.MakeForwardingProxyHandler(reverseProxy, forwardingNotifiers, urlResolver, nilURLTransformer, serviceAuthInjector)
faasHandlers.DeleteFunction = handlers.MakeForwardingProxyHandler(reverseProxy, forwardingNotifiers, urlResolver, nilURLTransformer, serviceAuthInjector)
faasHandlers.UpdateFunction = handlers.MakeForwardingProxyHandler(reverseProxy, forwardingNotifiers, urlResolver, nilURLTransformer, serviceAuthInjector)
faasHandlers.FunctionStatus = handlers.MakeForwardingProxyHandler(reverseProxy, forwardingNotifiers, urlResolver, nilURLTransformer, serviceAuthInjector)

faasHandlers.InfoHandler = handlers.MakeInfoHandler(handlers.MakeForwardingProxyHandler(reverseProxy, forwardingNotifiers, urlResolver, nilURLTransformer, serviceAuthInjector))
faasHandlers.SecretHandler = handlers.MakeForwardingProxyHandler(reverseProxy, forwardingNotifiers, urlResolver, nilURLTransformer, serviceAuthInjector)

// https://github.com/openfaas/faas/blob/88eea5f62e1d6a1812265793eacee6213b74af80/gateway/handlers/forwarding_proxy.go#L20
// 解析url -> forwardRequest转发请求 -> notifier打印日志
func MakeForwardingProxyHandler(proxy *types.HTTPClientReverseProxy,
        notifiers []HTTPNotifier,
        baseURLResolver middleware.BaseURLResolver,
        urlPathTransformer middleware.URLPathTransformer,
        serviceAuthInjector middleware.AuthInjector) http.HandlerFunc {

        writeRequestURI := false
        if _, exists := os.LookupEnv("write_request_uri"); exists {
                writeRequestURI = exists
        }

        return func(w http.ResponseWriter, r *http.Request) {
                baseURL := baseURLResolver.Resolve(r)
                originalURL := r.URL.String()
                requestURL := urlPathTransformer.Transform(r)

                for _, notifier := range notifiers {
                        notifier.Notify(r.Method, requestURL, originalURL, http.StatusProcessing, "started", time.Second*0)
                }

                start := time.Now()

                statusCode, err := forwardRequest(w, r, proxy.Client, baseURL, requestURL, proxy.Timeout, writeRequestURI, serviceAuthInjector)

                seconds := time.Since(start)
                if err != nil {
                        log.Printf("error with upstream request to: %s, %s\n", requestURL, err.Error())
                }

                for _, notifier := range notifiers {
                        notifier.Notify(r.Method, requestURL, originalURL, statusCode, "completed", seconds)
                }
        }
}
```

[ui](https://github.com/openfaas/faas/tree/master/gateway/assets)

- Prometheus
  https://github.com/prometheus/prometheus
  Prometheus 是一个开源开源监控告警库，目的是监控各个函数的指标
- alertManager
  alertManager 负责[管理扩缩容](https://github.com/openfaas/faas/blob/88eea5f62e1d6a1812265793eacee6213b74af80/gateway/handlers/alerthandler.go#L75)，并及时通知给 Prometheus

```golang
// 获取副本数 -> 计算新副本数 -> 设置为新副本数
func scaleService(alert requests.PrometheusInnerAlert, service scaling.ServiceQuery, defaultNamespace string) error {
        var err error

        serviceName, namespace := middleware.GetNamespace(defaultNamespace, alert.Labels.FunctionName)

        if len(serviceName) > 0 {
                queryResponse, getErr := service.GetReplicas(serviceName, namespace)
                if getErr == nil {
                        status := alert.Status

                        newReplicas := CalculateReplicas(status, queryResponse.Replicas, uint64(queryResponse.MaxReplicas), queryResponse.MinReplicas, queryResponse.ScalingFactor)

                        log.Printf("[Scale] function=%s %d => %d.\n", serviceName, queryResponse.Replicas, newReplicas)
                        if newReplicas == queryResponse.Replicas {
                                return nil
                        }

                        updateErr := service.SetReplicas(serviceName, namespace, newReplicas)
                        if updateErr != nil {
                                err = updateErr
                        }
                }
        }
        return err
}
```

# Provider

https://github.com/openfaas/faas-provider
本质上是操作 k8s 的 api，主要功能就是获取/缩放/调用函数。
![provider_interface](/image/openfaas/provoder_interface.png)
但是其本质上是一个 interface，实现这个 interface 即可接入 openfaas，这样你可以更改你的容器编排工具，官方提供了 [k8s](https://github.com/openfaas/faas-netes) 和 [swarm](https://github.com/openfaas/faas-swarm) 的实现。
![provider_interface](/image/openfaas/provider_k8s.png)
实现者只要引入 provider 包，然后调用该 [serve](https://github.com/openfaas/faas-provider/blob/master/serve.go) 方法，传入构造好的 [handlers](https://github.com/openfaas/faas-provider/blob/97d35d2841b56d5fc41644d5d5b8a432afb2c741/types/config.go#L14) 作为参数，就能够和 gateway 打通
[以 k8s 为例](https://github.com/openfaas/faas-netes/blob/master/main.go#L218)

```golang
bootstrapHandlers := providertypes.FaaSHandlers{
        FunctionProxy:        proxy.NewHandlerFunc(config.FaaSConfig, functionLookup),
        DeleteHandler:        handlers.MakeDeleteHandler(config.DefaultFunctionNamespace, kubeClient),
        DeployHandler:        handlers.MakeDeployHandler(config.DefaultFunctionNamespace, factory),
        FunctionReader:       handlers.MakeFunctionReader(config.DefaultFunctionNamespace, listers.DeploymentInformer.Lister()),
        ReplicaReader:        handlers.MakeReplicaReader(config.DefaultFunctionNamespace, listers.DeploymentInformer.Lister()),
        ReplicaUpdater:       handlers.MakeReplicaUpdater(config.DefaultFunctionNamespace, kubeClient),
        UpdateHandler:        handlers.MakeUpdateHandler(config.DefaultFunctionNamespace, factory),
        HealthHandler:        handlers.MakeHealthHandler(),
        InfoHandler:          handlers.MakeInfoHandler(version.BuildVersion(), version.GitCommit),
        SecretHandler:        handlers.MakeSecretHandler(config.DefaultFunctionNamespace, kubeClient),
        LogHandler:           logs.NewLogHandlerFunc(k8s.NewLogRequestor(kubeClient, config.DefaultFunctionNamespace), config.FaaSConfig.WriteTimeout),
        ListNamespaceHandler: handlers.MakeNamespacesLister(config.DefaultFunctionNamespace, config.ClusterRole, kubeClient),
}
faasProvider.Serve(&bootstrapHandlers, &config.FaaSConfig)
```

provider 主要提供了手动触发和 alertManager 触发两种形式，可以根据每秒请求量，内存和 cpu 使用量或者自定义区间来进行扩缩容。

# Watchdog

https://github.com/openfaas/of-watchdog
Watchdog 是一个小型的 httpsever，每一个项目都会变成一个 docker，Watchdog 负责代理 provider 与函数进行通信，并且会进行健康检查和超时。
![watch_dog](/image/openfaas/watch_dog.png)
[健康检查]()

```golang
func makeHealthHandler() func(http.ResponseWriter, *http.Request) {
        return func(w http.ResponseWriter, r *http.Request) {
                switch r.Method {
                case http.MethodGet:
                        if atomic.LoadInt32(&acceptingConnections) == 0 || lockFilePresent() == false {
                                w.WriteHeader(http.StatusServiceUnavailable)
                                return
                        }

                        w.WriteHeader(http.StatusOK)
                        w.Write([]byte("OK"))

                default:
                        w.WriteHeader(http.StatusMethodNotAllowed)
                }
        }
}
```

[触发函数](https://github.com/openfaas/of-watchdog/blob/47d5c31a5a182999ea3f904d187355023ad75955/main.go#L248)

```golang
func (f *ForkFunctionRunner) Run(req FunctionRequest) error {
        log.Printf("Running: %s", req.Process)
        start := time.Now()

        var cmd *exec.Cmd
        var ctx context.Context
        if f.ExecTimeout > time.Millisecond*0 {
                var cancel context.CancelFunc
                ctx, cancel = context.WithTimeout(context.Background(), f.ExecTimeout)
                defer cancel()
        } else {
                ctx = context.Background()
        }

        cmd = exec.CommandContext(ctx, req.Process, req.ProcessArgs...)

        if req.InputReader != nil {
                defer req.InputReader.Close()
                cmd.Stdin = req.InputReader
        }

        cmd.Env = req.Environment
        cmd.Stdout = req.OutputWriter

        errPipe, _ := cmd.StderrPipe()

        // Prints stderr to console and is picked up by container logging driver.
        bindLoggingPipe("stderr", errPipe, os.Stderr, f.LogPrefix, f.LogBufferSize)

        if err := cmd.Start(); err != nil {
                return err
        }

        err := cmd.Wait()
        done := time.Since(start)
        if err != nil {
                return fmt.Errorf("%s exited: after %.2fs, error: %s", req.Process, done.Seconds(), err)
        }

        log.Printf("%s done: %.2fs secs", req.Process, done.Seconds())

        return nil
}
```

# Queue-worker

https://github.com/openfaas/nats-queue-worker
Queue-worker 主要用于处理异步函数，正常的函数调用逻辑如下：
![sync_queue](/image/openfaas/sync_queue.png)
在 Queue-worker 中的异步函数调用逻辑如下：
![async_queue_1](/image/openfaas/async_queue_1.png)
![async_queue_2](/image/openfaas/async_queue_2.png)
gateway[异步转发逻辑](https://github.com/openfaas/faas/blob/88eea5f62e1d6a1812265793eacee6213b74af80/gateway/handlers/queue_proxy.go#L25)

```golang
// 解析请求体 -> 获取 CallbackURL -> 构建异步 QueueRequest -> 放入 Queue 列队中
func MakeQueuedProxy(metrics metrics.MetricOptions, queuer ftypes.RequestQueuer, pathTransformer middleware.URLPathTransformer, defaultNS string, functionQuery scaling.FunctionQuery) http.HandlerFunc {
return func(w http.ResponseWriter, r \*http.Request) {
if r.Body != nil {
defer r.Body.Close()
}

                body, err := ioutil.ReadAll(r.Body)

                if err != nil {
                        http.Error(w, err.Error(), http.StatusBadRequest)
                        return
                }

                callbackURL, err := getCallbackURLHeader(r.Header)
                if err != nil {
                        http.Error(w, err.Error(), http.StatusBadRequest)
                        return
                }

                vars := mux.Vars(r)
                name := vars["name"]

                queueName, err := getQueueName(name, functionQuery)
                if err != nil {
                        http.Error(w, err.Error(), http.StatusInternalServerError)
                        return
                }

                req := &ftypes.QueueRequest{
                        Function:    name,
                        Body:        body,
                        Method:      r.Method,
                        QueryString: r.URL.RawQuery,
                        Path:        pathTransformer.Transform(r),
                        Header:      r.Header,
                        Host:        r.Host,
                        CallbackURL: callbackURL,
                        QueueName:   queueName,
                }

                if len(queueName) > 0 {
                        log.Printf("Queueing %s to: %s\n", name, queueName)
                }

                if err = queuer.Queue(req); err != nil {
                        fmt.Printf("Queue error: %v\n", err)
                        http.Error(w, err.Error(), http.StatusInternalServerError)
                        return
                }

                w.WriteHeader(http.StatusAccepted)
        }

}
```

# 200 行实现一个 mini-faas

```golang
// main.go 主要为一个 http-server
package main

import (
"fmt"
"net/http"

        "akt-faas/function_module"
        r "akt-faas/router"

        "github.com/gin-gonic/gin"

)

var router r.Router

func main() {

        router = r.Router{
                Paths: map[string]function_module.FunctionModule{},
        }

        r := gin.Default()

        r.GET("/function", func(c *gin.Context) {
                html := "<h1>Functions</h1>"
                for key, value := range router.Paths {
                        html += fmt.Sprint(`<a href="\function\`, key, `">`, value.Name, "</a><br>")
                }
                c.Header("Content-Type", "text/html; charset=utf-8")
                c.String(http.StatusOK, html)
        })

        r.POST("/function", func(c *gin.Context) {
                var body function_module.FunctionModule
                if err := c.BindJSON(&body); err != nil {
                        c.JSON(http.StatusServiceUnavailable, gin.H{
                                "message": err,
                        })
                        return
                }
                err := body.Build()
                if err != nil {
                        c.JSON(http.StatusServiceUnavailable, gin.H{
                                "message": "error2:" + err.Error(),
                        })
                        return
                }
                router.Insert(body.Path, body)
                c.JSON(http.StatusOK, gin.H{
                        "message": "success build function " + body.Name,
                })
        })

        r.DELETE("/function", func(c *gin.Context) {
                var body function_module.FunctionModule
                if err := c.BindJSON(&body); err != nil {
                        c.JSON(http.StatusServiceUnavailable, gin.H{
                                "message": "error",
                        })
                        return
                }
                err := router.Delete(body.Path)
                if !err {
                        c.JSON(http.StatusNotImplemented, gin.H{
                                "message": "not found function",
                        })
                        return
                }
                err1 := body.Delete()
                if err1 != nil {
                        c.JSON(http.StatusNotImplemented, gin.H{
                                "message": "delete failed",
                        })
                        return
                }
                c.JSON(http.StatusOK, gin.H{
                        "message": "success delete " + body.Path,
                })
        })

        r.GET("/function/:path", func(c *gin.Context) {
                path := c.Param("path")
                funcModule, ok := router.Find(path)
                if !ok {
                        c.JSON(http.StatusNotImplemented, gin.H{
                                "message": "not found",
                        })
                        return
                }
                resp, err := funcModule.Run()
                if err != nil {
                        c.JSON(http.StatusFailedDependency, gin.H{
                                "function_error": err.Error(),
                        })
                        return
                }
                html := fmt.Sprintf("<h2>function_name:%v</h2><h3>response_data:%v</h3>", funcModule.Name, resp)
                c.Header("Content-Type", "text/html; charset=utf-8")
                c.String(http.StatusOK, html)
        })

        r.Run() // listen and serve on 0.0.0.0:8080 (for windows "localhost:8080")

}
```

```golang
// router.go 注册相关函数
package router

import "akt-faas/function_module"

type Router struct {
        Paths map[string]function_module.FunctionModule
}

func (this *Router) Insert(path string, f function_module.FunctionModule) {
        this.Paths[path] = f
}

func (this *Router) Delete(path string) (err bool) {
        _, ok := this.Paths[path]
        if ok {
                delete(this.Paths, path)
                return true
        } else {
                return false
        }
}

func (this *Router) Find(path string) (f function_module.FunctionModule, err bool) {
        fn, ok := this.Paths[path]
        return fn, ok
}
```

```golang
// function_module.go 创建/触发/删除函数
package function_module

import (
        "encoding/base64"
        "fmt"
        "os"
        "os/exec"
)

type FunctionModule struct {
        Name     string
        Language string
        Source   string
        Method   string
        Path     string
        Cpu      string
        Memory   string
}

func (F *FunctionModule) Build() (err error) {
        fileData, err := base64.StdEncoding.DecodeString(F.Source)
        if err != nil {
                return err
        }
        path := "data"
        if _, err = os.Stat(path); err == nil {
                fmt.Println("path exists 1", path)
        } else {
                fmt.Println("path not exists ", path)
                err = os.MkdirAll(path, os.ModePerm)

                if err != nil {
                        fmt.Println("Error creating directory")
                        fmt.Println(err)
                        return err
                }
        }
        file, err1 := os.Create("data/source.tar.gz")
        if err1 != nil {
                return err1
        }
        file.Write(fileData)

        dockerFile := fmt.Sprint("templates/", F.Language, ".dockerfile")
        fmt.Println(dockerFile)
        funcName := fmt.Sprint("function-", F.Name)
        cmd := exec.Command("docker", "build", "-f", dockerFile, "-t", funcName, "data/")
        err2 := cmd.Run()
        if err2 != nil {
                return err2
        }
        os.Remove("data/source.tar.gz")
        return nil
}

func (F *FunctionModule) Run() (resp string, err error) {
        cpuConfig := fmt.Sprint("--cpus=", F.Cpu)
        memConfig := fmt.Sprint("--memory=", F.Memory)
        funcName := fmt.Sprint("function-", F.Name)
        cmd := exec.Command("docker", "run", "--rm", cpuConfig, memConfig, funcName)
        out, err := cmd.CombinedOutput()
        //
        if err != nil {
                return "", err
        }
        return string(out), nil
}

func (F *FunctionModule) Delete() (err error) {
        funcName := fmt.Sprint("function-", F.Name)
        cmd := exec.Command("docker", "rmi", funcName)
        err = cmd.Run()
        return err
}
```
