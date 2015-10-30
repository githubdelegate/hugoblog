+++
categories = ["code"]
date = "2015-10-27T15:55:15+08:00"
draft = false
slug = "influxdb-source-part1"
title = "Part1 - Server Run [InfluxDB Source]"

+++

2015年10月23日，我离开了老东家。虽然拿到了几家公司的offer，也在犹豫不决，心里挺烦的，打算找点事情来做，麻痹一下自己。于是打算阅读InfluxDB的源码，一是前公司使用的规模还是很大的，二是这个数据库设计的思路还是不错的，非常适合监控系统使用，几乎就是为数据采集存储而生的。相信以后的路上也会用到它，当然最重要的是学习他们的设计思想。

我打算分包来阅读分析，毕竟InfluxDB发展到今天已经是一个非常庞大的工程了，我们按照InfluxDB的启动流程开始阅读。我们在服务器上运行 `./influxd run `的时候其实是cmd包在和用户交互。

cmd下有 influx（客户端命令行）、influx_inspect（查看influxdb情况）、influx_stress（压力测试）和influxd（服务端）。我们目前只关心influxd服务端。

influxd 下有4个子包： `backup`  `help` `restore` `run`。
我们关心run这个命令。

``` golang

func (m *Main) Run(args ...string) error {
	name, args := ParseCommandName(args)
		
	// Extract name from args.
	switch name {
	case "", "run":
		cmd := run.NewCommand()
		
		// Tell the server the build details.
		cmd.Version = version
		cmd.Commit = commit
		cmd.Branch = branch
		cmd.BuildTime = buildTime
		
		if err := cmd.Run(args...); err != nil {
			return fmt.Errorf("run: %s", err)
		}

		signalCh := make(chan os.Signal, 1)
		signal.Notify(signalCh, os.Interrupt, syscall.SIGTERM)
		m.Logger.Println("Listening for signals")

		// Block until one of the signals above is received
		select {
		case <-signalCh:
			m.Logger.Println("Signal received, initializing clean shutdown...")
			go func() {
				cmd.Close()
			}()
		}

		// Block again until another signal is received, a shutdown timeout elapses,
		// or the Command is gracefully closed
		m.Logger.Println("Waiting for clean shutdown...")
		select {
		case <-signalCh:
			m.Logger.Println("second signal received, initializing hard shutdown")
		case <-time.After(time.Second * 30):
			m.Logger.Println("time limit reached, initializing hard shutdown")
		case <-cmd.Closed:
			m.Logger.Println("server shutdown completed")
		}

		// goodbye.
....

```

``` golang

func NewCommand() *Command {
	return &Command{
		closing: make(chan struct{}),
		Closed:  make(chan struct{}),
		Stdin:   os.Stdin,
		Stdout:  os.Stdout,
		Stderr:  os.Stderr,
	}
}

```

run中的command 定义了两个状态 `closing` 和 `closed` ，类型为chan struct{}，这样做的好处是当用golang builtin 的函数close掉的时候方便接到通知，处理一些扫尾工作。

``` golang

select {
	case <-signalCh:
		m.Logger.Println("second signal received, initializing hard shutdown")
	case <-time.After(time.Second * 30):
		m.Logger.Println("time limit reached, initializing hard shutdown")
	case <-cmd.Closed:
		m.Logger.Println("server shutdown completed")
	}
	
```

执行了run后，command开始解析参数，写PID，解析配置，验证配置的有效性，启动Server实例。注意：command line 的配置会覆盖配置文件中的参数。server中有两个函数非常重要，一个是 `func NewServer(c *Config, buildInfo *BuildInfo) (*Server, error) `,另一个是 `func (s *Server) Open() error ` 。NewServer用来实例化Server，Open用于启动Server。

非常值的一提的是，influxdb自己实现了一个基于tcp连接的端口复用服务，它允许多个服务监听在一个tcp端口上，根据自己定义的tcp第一个数据包来判断tcp连接类型，然后重写了accept函数接收mux分发下来的请求。相当于自己基于TCP实现了一套非常简单的协议。

``` golang 

		ln, err := net.Listen("tcp", s.BindAddress)
		if err != nil {
			return fmt.Errorf("listen: %s", err)
		}
		s.Listener = ln

		// The port 0 is used, we need to retrieve the port assigned by the kernel
		if strings.HasSuffix(s.BindAddress, ":0") {
			s.MetaStore.Addr = ln.Addr()
		}

		// Multiplex listener.
		mux := tcp.NewMux()
		s.MetaStore.RaftListener = mux.Listen(meta.MuxRaftHeader)
		s.MetaStore.ExecListener = mux.Listen(meta.MuxExecHeader)
		s.MetaStore.RPCListener = mux.Listen(meta.MuxRPCHeader)

		s.ClusterService.Listener = mux.Listen(cluster.MuxHeader)
		s.SnapshotterService.Listener = mux.Listen(snapshotter.MuxHeader)
		s.CopierService.Listener = mux.Listen(copier.MuxHeader)
		go mux.Serve(ln)

```

具体mux的TCP实现大家可以参考tcp包，后面有时间我会整理下。在Server中有两个属性非常重要，一个是 `MetaStore` 另一个是 `TSDBStore`, 他们会随着Server的实例化而实例化，随着Server的启动而启动。Server中属性众多，大家一点一点看，不要想着一下子全看完很容易看晕的。

