# filecoin 源码分析：启动流程

- [filecoin 源码分析：启动流程](#filecoin-源码分析启动流程)
  - [准备工作](#准备工作)
    - [环境](#环境)
    - [源码目录](#源码目录)
    - [设计思想](#设计思想)
  - [cmd模块](#cmd模块)
    - [入口](#入口)
    - [BackupCmd](#backupcmd)
    - [DaemonCmd](#daemoncmd)
  - [node模块](#node模块)
  - [chain模块](#chain模块)

## 准备工作

### 环境

分析源码：lotus，filecoin的go语言实现，版本号如下

```bash

Daemon:  1.11.0-dev+mainnet+git.715176698.dirty+api1.3.0
Local: lotus version 1.11.0-dev+mainnet+git.715176698.dirty

```

### 源码目录

在前面的笔记中有解读：https://gitee.com/xingzjx/blockchain/blob/master/filecoin/filecoin_source_guide.md

### 设计思想

在阅读源码之前，建议先部署编译filecoin，以及多看Filecoin白皮书，熟悉官方开发文档。这样有利于理解源码设计。

## cmd模块

cmd目录是应用程序入口，里面包含多个子模块，比如lotus，lotus-wallet,lotus-seal-worker,lotus-storage-miner。这里只分析lotus目录。

### 入口

在执行命令**lotus daemon**的时候，会启动**cmd/lotus/main.go**，该方法核心逻辑

- 定义Commond，分别是DaemonCmd和BackupCmd
- 启动分布式追踪，jaeger
- RunApp:最后调用urfave库v2，启动命令行应用

**cmd/lotus/main.go**

```go
func main() {
	api.RunningNodeType = api.NodeFull

	lotuslog.SetupLogLevels()

	local := []*cli.Command{
		DaemonCmd,
		backupCmd,
	}
	if AdvanceBlockCmd != nil {
		local = append(local, AdvanceBlockCmd)
	}

	jaeger := tracing.SetupJaegerTracing("lotus")
	defer func() {
		if jaeger != nil {
			jaeger.Flush()
		}
	}()

	for _, cmd := range local {
		cmd := cmd
		originBefore := cmd.Before
		cmd.Before = func(cctx *cli.Context) error {
			trace.UnregisterExporter(jaeger)
			jaeger = tracing.SetupJaegerTracing("lotus/" + cmd.Name)

			if originBefore != nil {
				return originBefore(cctx)
			}
			return nil
		}
	}
	ctx, span := trace.StartSpan(context.Background(), "/cli")
	defer span.End()

	interactiveDef := isatty.IsTerminal(os.Stdout.Fd()) || isatty.IsCygwinTerminal(os.Stdout.Fd())

	app := &cli.App{
		Name:                 "lotus",
		Usage:                "Filecoin decentralized storage network client",
		Version:              build.UserVersion(),
		EnableBashCompletion: true,
		Flags: []cli.Flag{
			&cli.StringFlag{
				Name:    "repo",
				EnvVars: []string{"LOTUS_PATH"},
				Hidden:  true,
				Value:   "~/.lotus", // TODO: Consider XDG_DATA_HOME
			},
			&cli.BoolFlag{
				Name:  "interactive",
				Usage: "setting to false will disable interactive functionality of commands",
				Value: interactiveDef,
			},
			&cli.BoolFlag{
				Name:  "force-send",
				Usage: "if true, will ignore pre-send checks",
			},
		},

		Commands: append(local, lcli.Commands...),
	}

	app.Setup()
	app.Metadata["traceContext"] = ctx
	app.Metadata["repoType"] = repo.FullNode

	lcli.RunApp(app)
}
```
### BackupCmd 

启动backup服务，执行数据备份相关逻辑。

**cmd/lotus/backup.go**

```go
var backupCmd = lcli.BackupCmd("repo", repo.FullNode, func(cctx *cli.Context) (lcli.BackupAPI, jsonrpc.ClientCloser, error) {
	return lcli.GetFullNodeAPI(cctx)
})
```

**cli/backup.go**

```go
func BackupCmd(repoFlag string, rt repo.RepoType, getApi BackupApiFn) *cli.Command {
	var offlineBackup = func(cctx *cli.Context) error {
		logging.SetLogLevel("badger", "ERROR") // nolint:errcheck

		repoPath := cctx.String(repoFlag)
		r, err := repo.NewFS(repoPath)
		if err != nil {
			return err
		}

		ok, err := r.Exists()
		if err != nil {
			return err
		}
		if !ok {
			return xerrors.Errorf("repo at '%s' is not initialized", cctx.String(repoFlag))
		}

		lr, err := r.LockRO(rt)
		if err != nil {
			return xerrors.Errorf("locking repo: %w", err)
		}
		defer lr.Close() // nolint:errcheck

		mds, err := lr.Datastore(context.TODO(), "/metadata")
		if err != nil {
			return xerrors.Errorf("getting metadata datastore: %w", err)
		}

		bds, err := backupds.Wrap(mds, backupds.NoLogdir)
		if err != nil {
			return err
		}

		fpath, err := homedir.Expand(cctx.Args().First())
		if err != nil {
			return xerrors.Errorf("expanding file path: %w", err)
		}

		out, err := os.OpenFile(fpath, os.O_CREATE|os.O_WRONLY, 0644)
		if err != nil {
			return xerrors.Errorf("opening backup file %s: %w", fpath, err)
		}

		if err := bds.Backup(out); err != nil {
			if cerr := out.Close(); cerr != nil {
				log.Errorw("error closing backup file while handling backup error", "closeErr", cerr, "backupErr", err)
			}
			return xerrors.Errorf("backup error: %w", err)
		}

		if err := out.Close(); err != nil {
			return xerrors.Errorf("closing backup file: %w", err)
		}

		return nil
	}

	var onlineBackup = func(cctx *cli.Context) error {
		api, closer, err := getApi(cctx)
		if err != nil {
			return xerrors.Errorf("getting api: %w (if the node isn't running you can use the --offline flag)", err)
		}
		defer closer()

		err = api.CreateBackup(ReqContext(cctx), cctx.Args().First())
		if err != nil {
			return err
		}

		fmt.Println("Success")

		return nil
	}

	return &cli.Command{
		Name:  "backup",
		Usage: "Create node metadata backup",
		Description: `The backup command writes a copy of node metadata under the specified path

Online backups:
For security reasons, the daemon must be have LOTUS_BACKUP_BASE_PATH env var set
to a path where backup files are supposed to be saved, and the path specified in
this command must be within this base path`,
		Flags: []cli.Flag{
			&cli.BoolFlag{
				Name:  "offline",
				Usage: "create backup without the node running",
			},
		},
		ArgsUsage: "[backup file path]",
		Action: func(cctx *cli.Context) error {
			if cctx.Args().Len() != 1 {
				return xerrors.Errorf("expected 1 argument")
			}

			if cctx.Bool("offline") {
				return offlineBackup(cctx)
			}

			return onlineBackup(cctx)
		},
	}
}
```

**lib/backups/datastore.go**

```go
func Wrap(child datastore.Batching, logdir string) (*Datastore, error) {
	ds := &Datastore{
		child: child,
	}

	if logdir != NoLogdir {
		ds.closing, ds.closed = make(chan struct{}), make(chan struct{})
		ds.log = make(chan Entry)

		if err := ds.startLog(logdir); err != nil {
			return nil, err
		}
	}

	return ds, nil
}
```
**lib/backups/log.go**

```go
func (d *Datastore) startLog(logdir string) error {
	if err := os.MkdirAll(logdir, 0755); err != nil && !os.IsExist(err) {
		return xerrors.Errorf("mkdir logdir ('%s'): %w", logdir, err)
	}

	files, err := ioutil.ReadDir(logdir)
	if err != nil {
		return xerrors.Errorf("read logdir ('%s'): %w", logdir, err)
	}

	var latest string
	var latestTs int64

	for _, file := range files {
		fn := file.Name()
		if !strings.HasSuffix(fn, ".log.cbor") {
			log.Warn("logfile with wrong file extension", fn)
			continue
		}
		sec, err := strconv.ParseInt(fn[:len(".log.cbor")], 10, 64)
		if err != nil {
			return xerrors.Errorf("parsing logfile as a number: %w", err)
		}

		if sec > latestTs {
			latestTs = sec
			latest = file.Name()
		}
	}

	var l *logfile
	if latest == "" {
		l, latest, err = d.createLog(logdir)
		if err != nil {
			return xerrors.Errorf("creating log: %w", err)
		}
	} else {
		l, latest, err = d.openLog(filepath.Join(logdir, latest))
		if err != nil {
			return xerrors.Errorf("opening log: %w", err)
		}
	}

	if err := l.writeLogHead(latest, d.child); err != nil {
		return xerrors.Errorf("writing new log head: %w", err)
	}

	go d.runLog(l)

	return nil
}
```

### DaemonCmd

DaemonCmd是lotus启动后的核心逻辑，包括解析启动参数，p2p连接，同步数据等都在这里定义的逻辑。




## node模块

## chain模块