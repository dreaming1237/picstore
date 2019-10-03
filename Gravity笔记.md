最近学习了开源项目，gravity，记录如下：

### 概述&准备工作
1. Gravity 是一款数据复制组件，提供全量、增量数据同步，以及向消息队列发布数据更新。
2. [项目地址](https://github.com/moiot/gravity)
3. [Quick start](https://github.com/moiot/gravity/blob/master/docs/2.0/01-quick-start-en.md)
4. 配置MySQL环境
5. 配置Go语言环境

### Demo
1. 按照[Quick start](https://github.com/moiot/gravity/blob/master/docs/2.0/01-quick-start-en.md)一步步配置即可

### 效果展示
1. 在向test_source_table插入数据之前，test_source_table和test_target_table皆为空。如下：

	```
	mysql> select * from test_source_table;
	Empty set (0.00 sec)
	
	mysql> select * from test_target_table;
	Empty set (0.00 sec)
	```
2. 向test\_source\_table中插入一条数据

	```
	mysql> insert into test_source_table (id) values (1);
	Query OK, 1 row affected (0.00 sec)
	
	mysql> select * from test_source_table;
	+----+
	| id |
	+----+
	|  1 |
	+----+
	1 row in set (0.00 sec)
	```
3. 查看target表

	```
	mysql> select * from test_target_table;
	+----+
	| id |
	+----+
	|  1 |
	+----+
	1 row in set (0.00 sec)
	```

4. 至此，demo演示成功，不禁打上一个问号，Gravity究竟是如何将数据同步过去的？为此，在git上翻看了官方文档，找到一张架构图如下：

![](https://raw.githubusercontent.com/dreaming1237/picstore/master/WechatIMG4.png)

  高，实在是高！大道至简，就是这么简洁。可是我想了解的事细节，找了半天，放弃了，看来还是得自己读源码了，作为一个Golang新手，只能硬着头皮看（DEBUG）了。
  
### 源码走读
1. 如何开始呢？Quick start 中启动的命令

    ```
    cd gravity && make
    bin/gravity -config config.toml
    ```
  - cd gravity && make 其实是执行到了makefile

    ```
    build:
	$(GOBUILD) -ldflags '$(LDFLAGS)' -o bin/gravity cmd/gravity/main.go
    ```
  - 所以，我们来看一下gravity/cmd/gravity/main.go中的方法

    ```
      ......
      // 1 parse config file  
	  if err := cfg.ConfigFromFile(cfg.ConfigFile); err != nil {
			  log.Fatalf("failed to load config from file: %v",     errors.ErrorStack(err))
		  }
	  ......
	  // 2 start server
	  server, err := app.NewServer(cfg.PipelineConfig)
	  .......
	  err = server.Start()
	
    ```
  
    1. 主要是解析之前的config.toml，并写****入PipelineConfigV3中
    2. 这一步是关键，主要在这里启动了server,start会根据config来初始化
    
2. 进入到Server的NewServer中，需要看ParsePlugins这个至关重要的函数
  
    ```
    func ParsePlugins(pipelineConfig config.PipelineConfigV3) (*Server, error) {

	    server := Server{}

	    // output
	    plugin, err := registry.GetPlugin(registry.OutputPlugin, pipelineConfig.OutputPlugin.Type)
	    output, ok := plugin.(core.Output)
	    server.Output = output

	    // scheduler
	    plugin, err = registry.GetPlugin(registry.SchedulerPlugin,     pipelineConfig.SchedulerPlugin.Type)
	    scheduler, ok := plugin.(core.Scheduler)
	    server.Scheduler = scheduler


	    // emitters
	    fs, err := filters.NewFilters(pipelineConfig.FilterPlugins)
	    server.filters = fs
	    server.Emitter = e

    	// input
    	inputPlugins := pipelineConfig.InputPlugin
    	input, err := newInput(pipelineConfig.PipelineName, inputPlugins)
    	server.Input = input

	    return &server, nil
    }
    ```
    1. 可以看出这里主要是根据config.toml文件生成我们需要的output、scheduler、emitter、input，这些Plugin都可以根据配置的不同生成具体的struct,例如MySQLOutput,mysqlStreamInputPlugin
3. 目前来看，上边架构图的几个主要的类，都已经解析出来，再看main.go的main方法，newServer之后就是start,start又做了一些什么事情呢？

    ```
    func (s *Server) Start() error {

	    switch o := s.Output.(type) {
	      case core.SynchronousOutput:
		      if err := o.Start(); err != nil {
			      return errors.Trace(err)
		      }
	      case core.AsynchronousOutput:
		      if err := o.Start(s.Scheduler); err != nil {
			      return errors.Trace(err)
		      }   
	      default:
		      return errors.Errorf("output is an invalid type")
	      }

	    if err := s.Scheduler.Start(s.Output); err != nil {
		    return errors.Trace(err)
	    }

	    if err := s.PositionCache.Start(); err != nil {
		    return errors.Trace(err)
	    }


	    if err := s.Input.Start(s.Emitter, s.Output.GetRouter(),       s.PositionCache); err != nil {
		    return errors.Trace(err)
	    }

	    return nil
    }
    ```
    1. 可以看到这里 将output、scheduler、input都start起来了
4. 至此，上述架构图的各个主要的组件都start起来了，可是怎么串起来呢？又是怎么接收binlog然后再调度，再转发，再最后写入到目标数据库的呢？到处都是goroutine,channel，打断点Debug都不太清楚上哪里打，这可是有点麻烦。放个风，喝杯茶，回来继续。
    1. 既然是一条数据的处理，按照架构图上画的，起点应该在Input。带着猜测，继续撸gravity/pkg/inputs/mysqlstream/input.go->mysqlStreamInputPlugin.Start()
    
      ```
      func (plugin *mysqlStreamInputPlugin) Start(emitter core.Emitter, router core.Router, positionCache position_cache.PositionCacheInterface) error {

      	// binlog checker
      	plugin.binlogChecker, err = binlog_checker.NewBinlogChecker(
      		plugin.pipelineName,
      		plugin.probeDBConfig,
      		plugin.probeSQLAnnotation,
      		BinlogProbeInterval,
      		plugin.cfg.DisableBinlogChecker)
      	if err != nil {
      		return errors.Trace(err)
      	}
      	if err := plugin.binlogChecker.Start(); err != nil {
      		return errors.Trace(err)
      	}
      
      	// binlog tailer
      	gravityServerID := utils.GenerateRandomServerID()
      	plugin.ctx, plugin.cancel = context.WithCancel(context.Background())
      	plugin.binlogTailer, err = NewBinlogTailer(
      		plugin.pipelineName,
      		plugin.cfg,
      		gravityServerID,
      		positionCache,
      		sourceSchemaStore,
      		sourceDB,
      		emitter,
      		router,
      		plugin.binlogChecker,
      		nil)
      	if err != nil {
      		return errors.Trace(err)
      	}
      
      	if err := plugin.binlogTailer.Start(); err != nil {
      		return errors.Trace(err)
      	}
      
      	return nil
      }
      ```
      2. 等等，binlogChecker又是什么鬼？
      
      ```
      func (checker *binlogChecker) Start() error {
    	......
    	checker.run()
    	......
    	return nil
    }
    
      func (checker *binlogChecker) run() {
      	ticker := time.NewTicker(checker.checkInterval)
      	checker.wg.Add(1)
      	go func() {
      		defer func() {
      			ticker.Stop()
      			checker.wg.Done()
      		}()
      
      		for {
      			select {
      			case <-ticker.C:
      				checker.probe()
      			case <-checker.stop:
      				return
      			}
      		}
      	}()
      }
      
      func (checker *binlogChecker) probe() {
      	retryCnt := 3
      	err := retry.Do(func() error {
      		queryWithPlaceHolder := fmt.Sprintf(
      			"%sUPDATE %s SET offset = ?, update_time_at_gravity = ? WHERE name = ?",
      			checker.annotation,
      			binlogCheckerFullTableNameV2)
      		_, err := checker.sourceDB.Exec(queryWithPlaceHolder, sentOffset, time.Now(), checker.pipelineName)
      		return err
      	}, retryCnt, 1*time.Second)
      }
      ```
      可以看出，binlogChecker层层调用，其实是做了一个定时和DB的心跳检测，定时更新gravity_heartbeat_v2，见证奇迹的时刻：
      
      ```
      mysql> use _gravity;
	    Reading table information for completion of table and column names
	    You can turn off this feature to get a quicker startup with -A
	    
	    Database changed
	    mysql> show tables;
	    +----------------------+
	    | Tables_in__gravity   |
	    +----------------------+
	    | gravity_heartbeat_v2 |
	    | gravity_positions    |
	    +----------------------+
	    2 rows in set (0.00 sec)
	    
	    mysql> select * from gravity_heartbeat_v2;
	    +-----------------+--------+----------------------------+----------------------------+
	    | name            | offset | update_time_at_gravity     | update_time_at_source      |
	    +-----------------+--------+----------------------------+----------------------------+
	    | mysql2mysqlDemo |    354 | 2019-09-25 18:56:45.262373 | 2019-09-25 18:56:45.262575 |
	    +-----------------+--------+----------------------------+----------------------------+
	    1 row in set (0.00 sec)

      ```
      3. 那binlogTailer又是做什么的呢？跟进去看看。binlogTailer的start太长了，限于篇幅，我就不贴代码了（更重要的是我还没弄明白不同Event的确切含义，就不在这里班门弄斧了）。这里有两行很重要的代码：
      
      ```
	      streamer, err := tailer.getBinlogStreamer(binlogPositionValue.CurrentPosition.BinlogGTID)
	      
	      // in getBinlogStreamer
	      streamer, err := tailer.binlogSyncer.StartSyncGTID(gs)
      ```
      再看
      
      ```
      e, err := streamer.GetEvent(tailer.ctx)
      ```
      大概其就明白了，应该是借助go-mysql的binlogSyncer来注册一个回调，等到有binlog需要同步的时候，通过channel返回。然后，在通过tailer.AppendMsgTxnBuffer->FlushMsgTxnBuffer->tailer.emitter.Emit->msgSubmitter.SubmitMsg提交到scheduler的workerQueues里边
5. 继续阅读scheduler。等等，我们好像没有在config.toml中指定scheduler吧，这个scheduler从哪里来的呢？这就得看Server.go中的ParsePlugins了
  
      ```
      // scheduler
    	if pipelineConfig.SchedulerPlugin == nil {
    		pipelineConfig.SchedulerPlugin = &config.GenericPluginConfig{
    			Type:   "batch-table-scheduler",
    			Config: batch_table_scheduler.DefaultConfig,
    		}
    	}
      ```
      1. 原来，当我们没有设置scheduler的时候，初始化会为我们默认一个batch-table-scheduler。
      现在已经将msg提交到了scheduler中，那怎么才能传递到output呢？还记得刚才在Server.start()中，scheduler也被start了吗？让我们看看start里的代码：
      
      ```
      func (scheduler *batchScheduler) Start(output core.Output) error {
	
      	scheduler.workerQueues = make([]chan []*core.Msg, scheduler.cfg.NrWorker)
      	for i := 0; i < scheduler.cfg.NrWorker; i++ {
      		scheduler.workerQueues[i] = make(chan []*core.Msg, scheduler.cfg.QueueSize)
      		scheduler.workerWg.Add(1)
      		go func(q chan []*core.Msg, workerIndex int) {
      				......
      					err := retry.Do(func() error {
      					  // 1.call output execute
      						err := scheduler.syncOutput.Execute(msgBatch)
      						if err != nil {
      							metrics.SchedulerRetryCounter.WithLabelValues(env.PipelineName).Add(1)
      							log.Warnf("error execute sync output, retry. %v", err)
      						}
      						return err
      					}, scheduler.cfg.NrRetries, scheduler.cfg.RetrySleep)
      
      				......
      		}(scheduler.workerQueues[i], i)
      	}
      
      	return nil
      }
      ```
      2. 看到这个go+for循环，有没有很熟悉？对的，就是靠着for循环不断的轮训，最终将msg丢给了output。上边注释1的地方，至此，终于将binlog传递给了output
6. 接下来，我们继续看看output是怎么工作的。MySQLOutput中的execute就比较简单了。

      ```
      func (output *MySQLOutput) Execute(msgs []*core.Msg) error {

      	for _, batch := range batches {
      		......
      		err := output.sqlExecutor.Execute(batch, targetTableDef)
      		......
      	}
      
      	return nil
      }
      ```
      最终调用mysqlInsertIgnoreEngine.Execute->InternalTxnTaggerCfg.ExecWithInternalTxnTag->db.Exec将Data写入目标数据表中。
7. 总结起来就是这样：
    ![](https://github.com/dreaming1237/picstore/blob/master/WechatIMG5.png?raw=true)

