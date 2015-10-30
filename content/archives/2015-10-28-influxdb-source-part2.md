+++
categories = ["code"]
date = "2015-10-28T13:56:30+08:00"
draft = false
slug = "influxdb-source-part2"
title = "Part2 - Write Data [InfluxDB Source]"

+++

服务器启动了之后，我们就要开始通过InfluxDB提供的HTTP接口写数据了，底层实现到底是一个什么要的流程呢，让我们来看下吧。

当Server实例化的时候会加载很多service,其中有一个就是httpd，这个包封装了对外的所有http接口供用户调用。

``` golang

...

s.appendHTTPDService(c.HTTPD)

...

func (s *Server) appendHTTPDService(c httpd.Config) {
	if !c.Enabled {
		return
	}
	srv := httpd.NewService(c)
	srv.Handler.MetaStore = s.MetaStore
	srv.Handler.QueryExecutor = s.QueryExecutor
	srv.Handler.PointsWriter = s.PointsWriter
	srv.Handler.Version = s.buildInfo.Version

	// If a ContinuousQuerier service has been started, attach it.
	for _, srvc := range s.Services {
		if cqsrvc, ok := srvc.(continuous_querier.ContinuousQuerier); ok {
			srv.Handler.ContinuousQuerier = cqsrvc
		}
	}

	s.Services = append(s.Services, srv)
}


```

上面是实例化httpd service，c是httpd模块的配置，可配置的选项如下：

``` golang

type Config struct {
	Enabled          bool   `toml:"enabled"`
	BindAddress      string `toml:"bind-address"`
	AuthEnabled      bool   `toml:"auth-enabled"`
	LogEnabled       bool   `toml:"log-enabled"`
	WriteTracing     bool   `toml:"write-tracing"`
	PprofEnabled     bool   `toml:"pprof-enabled"`
	HttpsEnabled     bool   `toml:"https-enabled"`
	HttpsCertificate string `toml:"https-certificate"`
}

// 默认配置如下
func NewConfig() Config {
	return Config{
		Enabled:          true,
		BindAddress:      ":8086",
		LogEnabled:       true,
		HttpsEnabled:     false,
		HttpsCertificate: "/etc/ssl/influxdb.pem",
	}
}

```


接下来我们看看 httpd.NewService 的具体实现。在httpd的service中有个非常重要的属性，Handler。它处理所有用户的请求，influxdb借助了一个第三方库 github.com/bmizerany/pat 实现了一个简单的路由。我们特别看下写入数据的请求是怎么做的。"POST", "/write", true, true, h.serveWrite

serveWrite 函数会做些全局的准备工作，比如过来的数据包如果是压缩的进行解压，然后根据header里面的内容类型往下分发，因为之前老版本的influxdb是json body请求，所以这里会判断是json body还是最新的line格式。我们直接看line的处理函数serveWriteLine，毕竟influxdb会慢慢抛弃json格式请求。


``` golang

// serveWriteLine receives incoming series data in line protocol format and writes it to the database.
func (h *Handler) serveWriteLine(w http.ResponseWriter, r *http.Request, body []byte, user *meta.UserInfo) {
	// Some clients may not set the content-type header appropriately and send JSON with a non-json
	// content-type.  If the body looks JSON, try to handle it as as JSON instead
	if len(body) > 0 {
		var i int
		for {
			// JSON requests must start w/ an opening bracket
			if body[i] == '{' {
				h.serveWriteJSON(w, r, body, user)
				return
			}

			// check that the byte is in the standard ascii code range
			if body[i] > 32 {
				break
			}
			i += 1
		}
	}

	precision := r.FormValue("precision")
	if precision == "" {
		precision = "n"
	}

	points, err := models.ParsePointsWithPrecision(body, time.Now().UTC(), precision)
	if err != nil {
		if err.Error() == "EOF" {
			w.WriteHeader(http.StatusOK)
			return
		}
		h.writeError(w, influxql.Result{Err: err}, http.StatusBadRequest)
		return
	}

	database := r.FormValue("db")
	if database == "" {
		h.writeError(w, influxql.Result{Err: fmt.Errorf("database is required")}, http.StatusBadRequest)
		return
	}

	if di, err := h.MetaStore.Database(database); err != nil {
		h.writeError(w, influxql.Result{Err: fmt.Errorf("metastore database error: %s", err)}, http.StatusInternalServerError)
		return
	} else if di == nil {
		h.writeError(w, influxql.Result{Err: fmt.Errorf("database not found: %q", database)}, http.StatusNotFound)
		return
	}

	if h.requireAuthentication && user == nil {
		h.writeError(w, influxql.Result{Err: fmt.Errorf("user is required to write to database %q", database)}, http.StatusUnauthorized)
		return
	}

	if h.requireAuthentication && !user.Authorize(influxql.WritePrivilege, database) {
		h.writeError(w, influxql.Result{Err: fmt.Errorf("%q user is not authorized to write to database %q", user.Name, database)}, http.StatusUnauthorized)
		return
	}

	// Determine required consistency level.
	consistency := cluster.ConsistencyLevelOne
	switch r.Form.Get("consistency") {
	case "all":
		consistency = cluster.ConsistencyLevelAll
	case "any":
		consistency = cluster.ConsistencyLevelAny
	case "one":
		consistency = cluster.ConsistencyLevelOne
	case "quorum":
		consistency = cluster.ConsistencyLevelQuorum
	}

	// Write points.
	if err := h.PointsWriter.WritePoints(&cluster.WritePointsRequest{
		Database:         database,
		RetentionPolicy:  r.FormValue("rp"),
		ConsistencyLevel: consistency,
		Points:           points,
	}); influxdb.IsClientError(err) {
		h.statMap.Add(statPointsWrittenFail, int64(len(points)))
		h.writeError(w, influxql.Result{Err: err}, http.StatusBadRequest)
		return
	} else if err != nil {
		h.statMap.Add(statPointsWrittenFail, int64(len(points)))
		h.writeError(w, influxql.Result{Err: err}, http.StatusInternalServerError)
		return
	}

	h.statMap.Add(statPointsWrittenOK, int64(len(points)))
	w.WriteHeader(http.StatusNoContent)
}

```

令人震惊的是，我在阅读源代码的时候，发现一个bug，就在上面的这个函数的开头，这个for循环是有问题的，经过简单的测试确实会panic，因为handler经过了recovery，所以服务是没有问题的，我把我的测试结果贴下：


``` shell

[wal] 2015/10/28 11:39:35 Flush due to idle. Flushing 7 series with 7 points and 407 bytes from partition 1
[wal] 2015/10/28 11:39:35 write to index of partition 1 took 2.718621ms
[wal] 2015/10/28 11:39:45 Flush due to idle. Flushing 7 series with 7 points and 407 bytes from partition 1
[wal] 2015/10/28 11:39:45 write to index of partition 1 took 2.267708ms
[wal] 2015/10/28 11:39:55 Flush due to idle. Flushing 7 series with 7 points and 407 bytes from partition 1
[wal] 2015/10/28 11:39:55 write to index of partition 1 took 3.516488ms
[wal] 2015/10/28 11:40:05 Flush due to idle. Flushing 8 series with 8 points and 424 bytes from partition 1
[wal] 2015/10/28 11:40:05 write to index of partition 1 took 3.126415ms
[wal] 2015/10/28 11:40:15 Flush due to idle. Flushing 8 series with 8 points and 424 bytes from partition 1
[wal] 2015/10/28 11:40:15 write to index of partition 1 took 1.716188ms
[http] 2015/10/28 11:40:25 221.221.221.98 - - [28/Oct/2015:11:40:25 +0800] POST /write HTTP/1.1 400 57 - Mozilla/5.0 (Macintosh; Intel Mac 
OS X 10_11_1) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/46.0.2490.71 Safari/537.36 9facaff5-7d25-11e5-8001-000000000000 2.522487ms
[wal] 2015/10/28 11:40:25 Flush due to idle. Flushing 8 series with 8 points and 424 bytes from partition 1
[wal] 2015/10/28 11:40:25 write to index of partition 1 took 12.511171ms
[wal] 2015/10/28 11:40:35 Flush due to idle. Flushing 8 series with 8 points and 442 bytes from partition 1
[wal] 2015/10/28 11:40:35 write to index of partition 1 took 7.997816ms
[http] 2015/10/28 11:40:38 221.221.221.98 - - [28/Oct/2015:11:40:38 +0800] POST /write HTTP/1.1 400 45 - Mozilla/5.0 (Macintosh; Intel Mac 
OS X 10_11_1) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/46.0.2490.71 Safari/537.36 a776548d-7d25-11e5-8002-000000000000 19.391701ms
[wal] 2015/10/28 11:40:45 Flush due to idle. Flushing 8 series with 8 points and 442 bytes from partition 1
[wal] 2015/10/28 11:40:45 write to index of partition 1 took 1.890025ms
[http] 2015/10/28 11:40:50 221.221.221.98 - - [28/Oct/2015:11:40:50 +0800] POST /write HTTP/1.1 200 23 - Mozilla/5.0 (Macintosh; Intel Mac 
OS X 10_11_1) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/46.0.2490.71 Safari/537.36 af16a5b9-7d25-11e5-8003-000000000000 1.222898ms [pan
ic:runtime error: index out of range]

```
最下面一条出现了 panic，OK ，让我们来修复下吧，问题出现在 if body[i] > 32 这个判断不够严谨，我们修改的结果如下

	if body[i] > 32 || i >= len(body)-1 
	
OK，go fmt 然后给官方发PR。

接下来解析出来的points会根据提交的db写入响应的数据库。首先判断验证信息（Meta负责），然后根据配置的 consistency level写数据，handler的 PointsWriter完成数据的写操作。它会先把 points 封装成一个写入请求，然后会交给cluster去写数据。

------

2015.10.31更新：

https://github.com/influxdb/influxdb/pull/4625 在otoolep的帮助下，PR被合并了，他帮忙写了 test case。


``` 

#4538: Dropping database under a write load causes panics
#4582: Correct logging tags in cluster and TCP package. Thanks @oiooj
#4513: TSM1: panic: runtime error: index out of range
#4521: TSM1: panic: decode of short block: got 1, exp 9
#4587: Prevent NaN float values from being stored
#4596: Skip empty string for start position when parsing line protocol @Thanks @ch33hau
#4610: Make internal stats names consistent with Go style.
#4625: Correctly handle bad write requests. Thanks @oiooj.


```

以后你们用InfluxDB的时候，要想着我啊。
