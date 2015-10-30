+++
categories = ["code"]
date = "2015-10-31T22:46:56+08:00"
draft = false
slug = "cluster-service"
title = "Part3 - Cluster [InfluxDB Source]"

+++

### cluster组成

* PointsWriter  *cluster.PointsWriter
* ShardWriter   *cluster.ShardWriter
* ShardMapper   *cluster.ShardMapper
* ClusterService

上面的这三个writer会在server启动的时候进行初始化，供其他服务加载的时候可以使用，调用的最多的就是封装好的最上层的一个writer `PointsWriter` , 当然它依赖于下面的 `ShardWriter`，而 `ShardMapper`则是`QueryExecutor`依赖的。


### cluster service

如上面所讲，除了3个非常重要的writer外，还有一个service。
这个service在服务启动的时候会监听一个TCP端口，在这个端口上只会接收两种请求：

``` golang
		// Delegate message processing by type.
		switch typ {
		case writeShardRequestMessage:
			s.statMap.Add(writeShardReq, 1)
			err := s.processWriteShardRequest(buf)
			if err != nil {
				s.Logger.Printf("process write shard error: %s", err)
			}
			s.writeShardResponse(conn, err)
		case mapShardRequestMessage:
			s.statMap.Add(mapShardReq, 1)
			err := s.processMapShardRequest(conn, buf)
			if err != nil {
				s.Logger.Printf("process map shard error: %s", err)
				if err := writeMapShardResponseMessage(conn, NewMapShardResponse(1, err.Error())); err != nil {
					s.Logger.Printf("process map shard error writing response: %s", err.Error())
				}
			}
		default:
			s.Logger.Printf("cluster service message type not found: %d", typ)
		}

```

writeShard最终调用的是 s.TSDBStore.WriteToShard ，而mapShard最终调用的是s.TSDBStore.CreateMapper


### cluster.PointsWriter

它只做了一件事，就是将提交过来的所有点，进行mapShard（根据db和rp），所有的shard逻辑信息都存放在meta里面，如果没有shard就创建，然后写数据w.TSDBStore.WriteToShard。这里最关键的就是map的方法：

``` golang

func (w *PointsWriter) MapShards(wp *WritePointsRequest) (*ShardMapping, error) {

	// holds the start time ranges for required shard groups
	timeRanges := map[time.Time]*meta.ShardGroupInfo{}

	rp, err := w.MetaStore.RetentionPolicy(wp.Database, wp.RetentionPolicy)
	if err != nil {
		return nil, err
	}
	if rp == nil {
		return nil, influxdb.ErrRetentionPolicyNotFound(wp.RetentionPolicy)
	}

	for _, p := range wp.Points {
		timeRanges[p.Time().Truncate(rp.ShardGroupDuration)] = nil
	}

	// holds all the shard groups and shards that are required for writes
	for t := range timeRanges {
		sg, err := w.MetaStore.CreateShardGroupIfNotExists(wp.Database, wp.RetentionPolicy, t)
		if err != nil {
			return nil, err
		}
		timeRanges[t] = sg
	}

	mapping := NewShardMapping()
	for _, p := range wp.Points {
		sg := timeRanges[p.Time().Truncate(rp.ShardGroupDuration)]
		sh := sg.ShardFor(p.HashID())
		mapping.MapPoint(&sh, p)
	}
	return mapping, nil
}

```



### cluster.ShardMapper

它基本上只做了一件事情，那就是CreateMapper：


``` golang

// CreateMapper returns a Mapper for the given shard ID.
func (s *ShardMapper) CreateMapper(sh meta.ShardInfo, stmt influxql.Statement, chunkSize int) (tsdb.Mapper, error) {
	// Create a remote mapper if the local node doesn't own the shard.
	if !sh.OwnedBy(s.MetaStore.NodeID()) || s.ForceRemoteMapping {
		// Pick a node in a pseudo-random manner.
		conn, err := s.dial(sh.Owners[rand.Intn(len(sh.Owners))].NodeID)
		if err != nil {
			return nil, err
		}
		conn.SetDeadline(time.Now().Add(s.timeout))

		return NewRemoteMapper(conn, sh.ID, stmt, chunkSize), nil
	}

	// If it is local then return the mapper from the store.
	m, err := s.TSDBStore.CreateMapper(sh.ID, stmt, chunkSize)
	if err != nil {
		return nil, err
	}

	return m, nil
}

```
不过它 封装了 local 和 remote请求

### cluster.ShardWriter

它是PointsWriter的一部分，因为它只负责shard的写请求。而local的请求直接交给了tsdb来处理。

这里为了提高性能，在往其他shard写数据的时候，实现了一个连接池：

``` glolang 

func (w *ShardWriter) dial(nodeID uint64) (net.Conn, error) {
	// If we don't have a connection pool for that addr yet, create one
	_, ok := w.pool.getPool(nodeID)
	if !ok {
		factory := &connFactory{nodeID: nodeID, clientPool: w.pool, timeout: w.timeout}
		factory.metaStore = w.MetaStore

		p, err := pool.NewChannelPool(1, 3, factory.dial)
		if err != nil {
			return nil, err
		}
		w.pool.setPool(nodeID, p)
	}
	return w.pool.conn(nodeID)
}


type connFactory struct {
	nodeID  uint64
	timeout time.Duration

	clientPool interface {
		size() int
	}

	metaStore interface {
		Node(id uint64) (ni *meta.NodeInfo, err error)
	}
}

func (c *connFactory) dial() (net.Conn, error) {
	if c.clientPool.size() > maxConnections {
		return nil, errMaxConnectionsExceeded
	}

	ni, err := c.metaStore.Node(c.nodeID)
	if err != nil {
		return nil, err
	}

	if ni == nil {
		return nil, fmt.Errorf("node %d does not exist", c.nodeID)
	}

	conn, err := net.DialTimeout("tcp", ni.Host, c.timeout)
	if err != nil {
		return nil, err
	}

	// Write a marker byte for cluster messages.
	_, err = conn.Write([]byte{MuxHeader})
	if err != nil {
		conn.Close()
		return nil, err
	}

	return conn, nil

```

是用pool实现的，感兴趣的可以研究下，非常不错。 文件：client_pool.go 

其实cluster服务就是对meta和tsdb在写数据和shard方面的一个封装。