### 写入总体流程

```go
// httpd/handler.go
Route{
			"write", // Data-ingest route.
			"POST", "/write", true, writeLogEnabled, h.serveWriteV1,
		}


// serveWriteV1 handles v1 style writes.
func (h *Handler) serveWriteV1(w http.ResponseWriter, r *http.Request, user meta.User) {
	precision := r.URL.Query().Get("precision")
	switch precision {
	case "", "n", "ns", "u", "ms", "s", "m", "h":
		// it's valid
	default:
		err := fmt.Sprintf("invalid precision %q (use n, u, ms, s, m or h)", precision)
		h.httpError(w, err, http.StatusBadRequest)
	}

	db := r.URL.Query().Get("db")
	rp := r.URL.Query().Get("rp")

	h.serveWrite(db, rp, precision, w, r, user)
}
```

write 的 handler 函数 ServerWriteV1 中，主要解析 URL，将时间粒度、db、rp 解析出来，传递给 serveWrite 函数。

```go
// serveWrite receives incoming series data in line protocol format and writes
// it to the database.
func (h *Handler) serveWrite(database, retentionPolicy, precision string, w http.ResponseWriter, r *http.Request, user meta.User) {
  // 前置工作
  // 1. 校验 database 是否为空或是否存在
  // 2. 用户鉴权
  // 3. 解压 http body
  // 4. 从 body 中解析出 points
  
  // 开始写 points
  	writePoints := func() error {
		switch pw := h.PointsWriter.(type) {
		case pointsWriterWithContext:
			var npoints, nvalues int64
			ctx := context.WithValue(context.Background(), coordinator.StatPointsWritten, &npoints)
			ctx = context.WithValue(ctx, coordinator.StatValuesWritten, &nvalues)

			// for now, just store the number of values used.
			err := pw.WritePointsWithContext(ctx, database, retentionPolicy, consistency, user, points)
			atomic.AddInt64(&h.stats.ValuesWrittenOK, nvalues)
			if err != nil {
				return err
			}
			return nil
		default:
			return h.PointsWriter.WritePoints(database, retentionPolicy, consistency, user, points)
		}
	}
  
  // 返回客户端结果
}
```

开源 influxdb 只支持单机版本，这里写入默认走的 WritePointsWithContext，里面会调用到 WritePointsPrivilegedWithContext，这里我们关注函数里面主要部分：

```go
// points_writer.go
func (w *PointsWriter) WritePointsPrivilegedWithContext(ctx context.Context, database, retentionPolicy string, consistencyLevel models.ConsistencyLevel, points []models.Point) error {
  // 将 point 分配到对应的 sg.shard 里
	shardMappings, err := w.MapShards(&WritePointsRequest{Database: database, RetentionPolicy: retentionPolicy, Points: points})
	if err != nil {
		return err
	}
  
  // 按 shard 并发写入各自的 points
	ch := make(chan error, len(shardMappings.Points))
	for shardID, points := range shardMappings.Points {
		go func(ctx context.Context, shard *meta.ShardInfo, database, retentionPolicy string, points []models.Point) {
			
			err := w.writeToShardWithContext(ctx, shard, database, retentionPolicy, points)
			if err == tsdb.ErrShardDeletion {
				err = tsdb.PartialWriteError{Reason: fmt.Sprintf("shard %d is pending deletion", shard.ID), Dropped: len(points)}
			}

			ch <- err
		}(ctx, shardMappings.Shards[shardID], database, retentionPolicy, points)
	}
}
```

重点关注下 MapShards 函数，这个函数非常关键，主要作用是将每个 point 分配到其对应的 shard 里，我们来看下具体实现：

```go
// points_writer.go
func (w *PointsWriter) MapShards(wp *WritePointsRequest) (*ShardMapping, error) {
  list := make(sgList, 0, 8)
	min := time.Unix(0, models.MinNanoTime)
	if rp.Duration > 0 {
		min = time.Now().Add(-rp.Duration)
	}
  
  // 遍历所有 point, 看起对应的 sg 是否创建，没有则创建
  for _, p := range wp.Points {
		// time 不在 rp 范围内或对应的 sg 已存在
		if p.Time().Before(min) || list.Covers(p.Time()) {
			continue
		}

		// 创建新的 sg
		sg, err := w.MetaClient.CreateShardGroup(wp.Database, wp.RetentionPolicy, p.Time())
		if err != nil {
			return nil, err
		}
    
		list = list.Append(*sg)
	}
  
  mapping := NewShardMapping(len(wp.Points))
	for _, p := range wp.Points {
		sg := list.ShardGroupAt(p.Time())

    // 将 point 分配给对应的 shard
		sh := sg.ShardFor(p)
		mapping.MapPoint(&sh, p)
	}
	return mapping, nil
}
```

继续返回到上一层函数，points 分发完毕后，会将 points 按 shard 并发写入，入口函数为 writeToShardWithContext，最终会调到 shard.WritePointsWithContext，具体实现：

```go
// shard.go
func (s *Shard) WritePointsWithContext(ctx context.Context, points []models.Point) error {
  // 校验哪些 filed 需要新建以及创建倒排索引
  points, fieldsToCreate, err := s.validateSeriesAndFields(points)
	if err != nil {
		if _, ok := err.(PartialWriteError); !ok {
			return err
		}
		
		writeError = err
	}
  
  // add any new fields and keep track of what needs to be saved
	if err := s.createFieldsAndMeasurements(fieldsToCreate); err != nil {
		return err
	}

  // 调用 engine 写入 points
  engine.WritePoints(points)
}
```

这里注意 validateSeriesAndFields 函数内对调用 CreateSeriesListIfNotExists 创建倒排索引，这一部分内容将放在索引分析文章中详细展开。points 最终的写入是在 engine.WritePointsWithContext 中具体呈现：

```go
// engine.go
func (e *Engine) WritePointsWithContext(ctx context.Context, points []models.Point) error {
  // fieldKey->values
  values := make(map[string][]Value, len(points))
  
  // 遍历所有 points，并将 point 对应的所有field 对应的值插入 values
  
  // first try to write to the cache
	if err := e.Cache.WriteMulti(values); err != nil {
		return err
	}

	if e.WALEnabled {
		if _, err := e.WAL.WriteMulti(values); err != nil {
			return err
		}
	}
}
```

由于一个 point 可能会有多个 field 字段，因此这里会有两层循环遍历 point 及其对应的 fields，并将这些 field 对应的值插入的 fieldKey->values 的映射关系中，然后将 values 写入 Cache，最后再写 WAL. 分析 WAL 会专门展开来讲。

到此，写入的这条线路已经梳理完毕，当然其中涉及的一些细节由于篇幅原因直接略过了，后面对于写入中涉及的重点部分会单独分析。
