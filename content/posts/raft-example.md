+++
title = "raft-example学习笔记"
date = 2019-01-21T14:08:13+08:00
tags = ["raft"]
categories = [""]
draft = false
+++

[raft-example](https://github.com/etcd-io/etcd/tree/master/contrib/raftexample)是etcd中raft库的一个实现示例，它包含了etcd raft库的基本用法，以及如何利用它来实现一个简单的支持HTTP API的KV存储数据库。

首先来看这个应用的目录结构：

```text
.
├── Procfile
├── README.md
├── doc.go
├── httpapi.go
├── kvstore.go
├── kvstore_test.go
├── listener.go
├── main.go
├── raft.go
├── raftexample
└── raftexample_test.go
```

其中`main.go`是应用的主入口，它会解析命令行参数，初始化raft节点实例，以及启动和提供HTTP API服务；`httpapi.go`中包含了这个kv存储系统的HTTP API的处理函数实现；`kvstore.go`中则提供了kv数据库与raft节点之间交互的桥梁，包括把更新状态同步传输到其他raft节点上，以及从raft状态机中读取提交已消息应用在kv字典上，读取快照载入等等；`raft.go`中包含了raft节点的实现，它封装了与底层raft协议通信的方法，以及包含消息接收，WAL写入，快照同步，节点变更等的处理，对于应用来说只需去关心这个raft节点提供的事件通道（通过channel来收发）进行状态同步即可。

首先看主入口`main.go`的实现：

```go
func main() {
	//...

	proposeC := make(chan string)
	defer close(proposeC)
	confChangeC := make(chan raftpb.ConfChange)
	defer close(confChangeC)

	// raft provides a commit stream for the proposals from the http api
	var kvs *kvstore
	getSnapshot := func() ([]byte, error) { return kvs.getSnapshot() }
	commitC, errorC, snapshotterReady := newRaftNode(*id, strings.Split(*cluster, ","), *join, getSnapshot, proposeC, confChangeC)

	kvs = newKVStore(<-snapshotterReady, proposeC, commitC, errorC)

	// the key-value http handler will propose updates to raft
	serveHttpKVAPI(kvs, *kvport, confChangeC, errorC)
}
```

略过了前面的一些参数解析的代码，它首先创建了两个channel：`proposeC`和`confChangeC`，接着根据传入参数来建立一个raft节点实例，构造函数会返回三个channel：`commitC`，`errorC`和`confChangeC`，这些channel在下面会作为参数来传入kv存储的构造函数，这些channel会作为消息通道来完成应用与raft节点之间的消息传递，后面会提到。

再来看`raft.go`中`newRaftNode`的实现：

```go
func newRaftNode(id int, peers []string, join bool, getSnapshot func() ([]byte, error), proposeC <-chan string,
	confChangeC <-chan raftpb.ConfChange) (<-chan *string, <-chan error, <-chan *snap.Snapshotter) {

	commitC := make(chan *string)
	errorC := make(chan error)

	rc := &raftNode{
		proposeC:    proposeC,
		confChangeC: confChangeC,
		commitC:     commitC,
		errorC:      errorC,
		id:          id,
		peers:       peers,
		join:        join,
		waldir:      fmt.Sprintf("raftexample-%d", id),
		snapdir:     fmt.Sprintf("raftexample-%d-snap", id),
		getSnapshot: getSnapshot,
		snapCount:   defaultSnapshotCount,
		stopc:       make(chan struct{}),
		httpstopc:   make(chan struct{}),
		httpdonec:   make(chan struct{}),

		snapshotterReady: make(chan *snap.Snapshotter, 1),
		// rest of structure populated after WAL replay
	}
	go rc.startRaft()
	return commitC, errorC, rc.snapshotterReady
}
```

这个函数接收6个参数：

- id：raft节点的唯一标识。
- peers：raft集群节点的列表。
- join：标识raft节点是否正在加入一个已在集群。
- getSnapshot：获取应用状态机快照的函数，由应用来提供。
- proposeC：接收应用发来的提案消息（kv更新）的通道。
- confChangeC：接收集群配置变更消息的通道。

函数会返回两个关键变量：

- commitC：用来通知应用准备提交的通道。
- errorC：接收raft节点错误的通道。

然后看`startRaft()`方法的实现：

```go
func (rc *raftNode) replayWAL() *wal.WAL {
	log.Printf("replaying WAL of member %d", rc.id)
	snapshot := rc.loadSnapshot()
	w := rc.openWAL(snapshot)
	_, st, ents, err := w.ReadAll()
	if err != nil {
		log.Fatalf("raftexample: failed to read WAL (%v)", err)
	}
	rc.raftStorage = raft.NewMemoryStorage()
	if snapshot != nil {
		rc.raftStorage.ApplySnapshot(*snapshot)
	}
	rc.raftStorage.SetHardState(st)

	// append to storage so raft starts at the right place in log
	rc.raftStorage.Append(ents)
	// send nil once lastIndex is published so client knows commit channel is current
	if len(ents) > 0 {
		rc.lastIndex = ents[len(ents)-1].Index
	} else {
		rc.commitC <- nil
	}
	return w
}

func (rc *raftNode) startRaft() {
	if !fileutil.Exist(rc.snapdir) {
		if err := os.Mkdir(rc.snapdir, 0750); err != nil {
			log.Fatalf("raftexample: cannot create dir for snapshot (%v)", err)
		}
	}
	rc.snapshotter = snap.New(zap.NewExample(), rc.snapdir)
	rc.snapshotterReady <- rc.snapshotter

	oldwal := wal.Exist(rc.waldir)
	rc.wal = rc.replayWAL()

	rpeers := make([]raft.Peer, len(rc.peers))
	for i := range rpeers {
		rpeers[i] = raft.Peer{ID: uint64(i + 1)}
	}
	c := &raft.Config{
		ID:                        uint64(rc.id),
		ElectionTick:              10,
		HeartbeatTick:             1,
		Storage:                   rc.raftStorage,
		MaxSizePerMsg:             1024 * 1024,
		MaxInflightMsgs:           256,
		MaxUncommittedEntriesSize: 1 << 30,
	}

	if oldwal {
		rc.node = raft.RestartNode(c)
	} else {
		startPeers := rpeers
		if rc.join {
			startPeers = nil
		}
		rc.node = raft.StartNode(c, startPeers)
	}

	rc.transport = &rafthttp.Transport{
		Logger:      zap.NewExample(),
		ID:          types.ID(rc.id),
		ClusterID:   0x1000,
		Raft:        rc,
		ServerStats: stats.NewServerStats("", ""),
		LeaderStats: stats.NewLeaderStats(strconv.Itoa(rc.id)),
		ErrorC:      make(chan error),
	}

	rc.transport.Start()
	for i := range rc.peers {
		if i+1 != rc.id {
			rc.transport.AddPeer(types.ID(i+1), []string{rc.peers[i]})
		}
	}

	go rc.serveRaft()
	go rc.serveChannels()
}
```

这个方法主要实现了以下几个步骤：

1. 重放WAL日志，首先检查是否有可用快照，将快照应用到raft内存状态机中，然后读取快照之后插入的日志进行同步，并更新raft节点`lastIndex`（日志中最后一条entry的索引值），这个步骤还会设置当前raft当前的状态（Term，Commit Index等）。
2. 完成日志重放后，初始化raft配置（包括刚刚与WAL同步的内存状态机），让raft协议启动或重启一个新的节点，配置并初始化raft的传输通道。
3. 启动后台运行的goroutine，这个goroutine会持续监听来自底层raft的事件和状态变更，以及处理来自应用的更新提案。

`serveChannels()`方法比较长，因此分成了几个部分。

```go
snap, err := rc.raftStorage.Snapshot()
if err != nil {
    panic(err)
}
rc.confState = snap.Metadata.ConfState
rc.snapshotIndex = snap.Metadata.Index
rc.appliedIndex = snap.Metadata.Index
```

这个方法首先会获取raft内存状态机的快照信息，并把快照中的几个状态同步到raft节点自身上，其中`appliedIndex`即为最后应用到状态机中的日志索引位置（它被初始化为snapshot index，因为状态机要从这个位置开始同步）。

```go
// send proposals over raft
go func() {
	confChangeCount := uint64(0)

	for rc.proposeC != nil && rc.confChangeC != nil {
		select {
		case prop, ok := <-rc.proposeC:
			if !ok {
				rc.proposeC = nil
			} else {
				// blocks until accepted by raft state machine
				rc.node.Propose(context.TODO(), []byte(prop))
			}

		case cc, ok := <-rc.confChangeC:
			if !ok {
				rc.confChangeC = nil
			} else {
				confChangeCount++
				cc.ID = confChangeCount
				rc.node.ProposeConfChange(context.TODO(), cc)
			}
		}
	}
	// client closed channel; shutdown raft if not already
	close(rc.stopc)
}()
```

接下来这部分则实现了从应用端接收提案请求（`proposeC`），并将指令提交到raft协议上，以及类似地，接收配置变更请求并提交到raft上。

```go
// event loop on raft state machine updates
for {
	select {
	case <-ticker.C:
		rc.node.Tick()

	// store raft entries to wal, then publish over commit channel
	case rd := <-rc.node.Ready():
		rc.wal.Save(rd.HardState, rd.Entries)
		if !raft.IsEmptySnap(rd.Snapshot) {
			rc.saveSnap(rd.Snapshot)
			rc.raftStorage.ApplySnapshot(rd.Snapshot)
			rc.publishSnapshot(rd.Snapshot)
		}
		rc.raftStorage.Append(rd.Entries)
		rc.transport.Send(rd.Messages)
		if ok := rc.publishEntries(rc.entriesToApply(rd.CommittedEntries)); !ok {
			rc.stop()
			return
		}
		rc.maybeTriggerSnapshot()
		rc.node.Advance()

	case err := <-rc.transport.ErrorC:
		rc.writeError(err)
		return

	case <-rc.stopc:
		rc.stop()
		return
	}
}
```

最后程序会在一个主循环中处理消息事件，节点通过`Ready()`方法从通道中获取到Ready对象，raft节点做的事情就是把ready中的日志条目写到WAL中，检查ready中是否包括快照，若包括快照则把快照同步到自身以及应用的状态机中（分别对应`rc.raftStorage.ApplySnapshot(rd.Snapshot)`和`rc.publishSnapshot(rd.Snapshot)`），把消息发送到其他节点，接下来在`rc.publishEntries(rc.entriesToApply(rd.CommittedEntries))`这一步中，raft节点会把`appliedIndex`到`Commit Index`之间的日志提交到应用状态机上。

 在`rc.maybeTriggerSnapshot()`这一步raft节点会检测`appliedIndex-snapshotIndex`的值是否大于快照的阈值（在这里是10000），如果是则调用应用提供的获取快照函数，把新的快照持久化到硬盘中，并将`snapshotIndex`设置为`appliedIndex`。

紧接着下一步调用`rc.node.Advance()`来通知raft节点当前的ready对象已经处理完成，准备好接收下一个可用的ready对象。

```go
func (rc *raftNode) publishEntries(ents []raftpb.Entry) bool {
	for i := range ents {
		switch ents[i].Type {
		case raftpb.EntryNormal:
			if len(ents[i].Data) == 0 {
				break
			}
			s := string(ents[i].Data)
			select {
			case rc.commitC <- &s:
			case <-rc.stopc:
				return false
			}

		case raftpb.EntryConfChange:
			//...
		}

		// after commit, update appliedIndex
		rc.appliedIndex = ents[i].Index

		// special nil commit to signal replay has finished
		if ents[i].Index == rc.lastIndex {
			select {
			case rc.commitC <- nil:
			case <-rc.stopc:
				return false
			}
		}
	}
	return true
}
```

`publishEntries()`的逻辑如上，raft节点通过`commitC`把commit entry的数据同步到应用状态机，并更新`appliedIndex`的值，如果出现`ents[i].Index == rc.lastIndex`（`lastIndex`在之前提到，是初始化时最后一个日志条目的索引值），说明日志重放已经完成。

raft节点的逻辑基本已经过完了，接下来再从`kvstore.go`中观察kv数据库是如何跟raft节点进行状态同步的。

应用通过`newKVStore()`函数来构造一个`kvstore`对象：

```go
func newKVStore(snapshotter *snap.Snapshotter, proposeC chan<- string, commitC <-chan *string, errorC <-chan error) *kvstore {
	s := &kvstore{proposeC: proposeC, kvStore: make(map[string]string), snapshotter: snapshotter}
	// replay log into key-value map
	s.readCommits(commitC, errorC)
	// read commits from raft into kvStore map until error
	go s.readCommits(commitC, errorC)
	return s
}

func (s *kvstore) readCommits(commitC <-chan *string, errorC <-chan error) {
	for data := range commitC {
		if data == nil {
			// done replaying log; new data incoming
			// OR signaled to load snapshot
			snapshot, err := s.snapshotter.Load()
			if err == snap.ErrNoSnapshot {
				return
			}
			if err != nil {
				log.Panic(err)
			}
			log.Printf("loading snapshot at term %d and index %d", snapshot.Metadata.Term, snapshot.Metadata.Index)
			if err := s.recoverFromSnapshot(snapshot.Data); err != nil {
				log.Panic(err)
			}
			continue
		}

		var dataKv kv
		dec := gob.NewDecoder(bytes.NewBufferString(*data))
		if err := dec.Decode(&dataKv); err != nil {
			log.Fatalf("raftexample: could not decode message (%v)", err)
		}
		s.mu.Lock()
		s.kvStore[dataKv.Key] = dataKv.Val
		s.mu.Unlock()
	}
	if err, ok := <-errorC; ok {
		log.Fatal(err)
	}
}
```

在`newKVStore()`里面，调用了`readCommits()`方法来同步提交日志，程序会不断从`commitC`通道中读取数据，当`data`为nil时代表一个特殊信号，说明raft节点已经完成日志回放了或者raft节点触发应用端来进行快照同步，为什么要进行两次`readCommits()`的调用？因为第一次需要进行日志回放（之前raft节点`publishEntries()`的实现中当`ents[i].Index == rc.lastIndex`时会发送nil到`commitC`），而第二次启动一个goroutine调用`readCommits()`来持续把提交日志应用到状态机上（实测当触发snapshot后程序会阻塞在第一个`readCommits`上，因为此时会一直卡在channel的读取上而不会退出，所以此处应该是有点问题的）。程序接收到数据后用解码器反序列化为kv对象，再把变更更新到`kvStore`这个map上面。

再回过头来看kv数据库读取和更新的实现，这时候便一目了然了：

```go
func (s *kvstore) Lookup(key string) (string, bool) {
	s.mu.RLock()
	v, ok := s.kvStore[key]
	s.mu.RUnlock()
	return v, ok
}

func (s *kvstore) Propose(k string, v string) {
	var buf bytes.Buffer
	if err := gob.NewEncoder(&buf).Encode(kv{k, v}); err != nil {
		log.Fatal(err)
	}
	s.proposeC <- buf.String()
}
```

