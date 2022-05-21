# logrus源码解析

## 1 介绍

​	logrus是Go语言编写的结构化日志工具，拥有丰富的日志API，支持自定义格式输出同时也支持hook操作，是目前比较流行的Go语言日志记录工具。

主要功能如下：

- 与标准库log API兼容
- 支持标准日志格式等级：Trace、Debug、Info、Warn、Error、Fatal、Panic
- 支持基于环境变量，设置不同的日志格式类型、日志记录输出方式、日志等级
- 日志记录中的Fields项格式是type Fields map[string]interface{}，支持多种值类型
- 支持公共信息记录，诸如IP来源、URL类型的信息
- 将日志记录器用作io.Writer，用于http server的ErrorLog日志记录
- 支持钩子操作（在日志等级高于某个级别，比如错误级别以上，触发邮件或者短信通知）
- 支持自定义格式化（logrus.TextFormatter、logrus.JSONFormatter）以及第三方格式化输出，比如logstash
- Fatal处理器（在os.Exit(1)退出应用之前，可以做出相关做的操作
- 并发安全，基于mutex互斥写操作，可以基于logger.SetNoLock()取消

**示例：**

```go
package main

import (
	log "github.com/sirupsen/logrus"
)

func main() {
	log.WithFields(log.Fields{
		"animal": "cat",
	}).Info("A cat appears")
}
// 输出
INFO[0000] A cat appears                                 animal=cat
```



​	logrus的文件内容主要集中在`exported.go, logger.go, entry.go, formatter.go, hooks.go `这五个文件中，`logger.go`文件主要定义了logrus的Logger对象的结构，以及此对象实现的方法；而`exported.go`则通过使用一个包装好的`stdLogger`对象对外提供日志记录的方法，最终的日志输出是有Entry对象来实现的，日志的输出格式由`formatter.go`来控制，而日志的hook方法则有`hooks.go`来控制。

## 2 核心数据结构分析

### 2.1 Logger结构体

```go
type Logger struct {
	// The logs are `io.Copy`'d to this in a mutex. It's common to set this to a
	// file, or leave it default which is `os.Stderr`. You can also set this to
	// something more adventurous, such as logging to Kafka.
	Out io.Writer
	// Hooks for the logger instance. These allow firing events based on logging
	// levels and log entries. For example, to send errors to an error tracking
	// service, log to StatsD or dump the core on fatal errors.
	Hooks LevelHooks
	// All log entries pass through the formatter before logged to Out. The
	// included formatters are `TextFormatter` and `JSONFormatter` for which
	// TextFormatter is the default. In development (when a TTY is attached) it
	// logs with colors, but to a file it wouldn't. You can easily implement your
	// own that implements the `Formatter` interface, see the `README` or included
	// formatters for examples.
	Formatter Formatter

	// Flag for whether to log caller info (off by default)
	ReportCaller bool

	// The logging level the logger should log at. This is typically (and defaults
	// to) `logrus.Info`, which allows Info(), Warn(), Error() and Fatal() to be
	// logged.
	Level Level
	// Used to sync writing to the log. Locking is enabled by Default
	mu MutexWrap
	// Reusable empty entry
	entryPool sync.Pool
	// Function to exit the application, defaults to `os.Exit()`
	ExitFunc exitFunc
}
```

- **Out**：日志输出的Writer对象，可以是os.Stderr,也可以是文件；
- **Hooks**：用于打印日志时的hooks操作；
- **Formatter**：日志内容格式化对象，用于格式化日志内容；
- **ReportCaller**：是否打印调用日志打印的函数的信息；
- **Level**：日志等级；
- **mu**：并发锁，控制并发日志记录；
- **entryPool**：日志内容对象的sync.Pool；
- **ExitFunc**：打印日志退出时执行的方法，通常在Fatal函数中被调用。

### 2.2 日志等级

先来看看logrus支持的日志等级，共有七类，分别是`Panic, Fatal, Error, Warn, Info, Debug, Trace`，日志等级有高到低，`Panic`级别的日志是最高等级的日志，当打印日志等级为`Panic`的时候，会显示地调用panic函数，`Fatal`级别的日志会在调用后显示地调用`logger.Exit(1)`函数，然后退出当前程序。

```go
const (
	// PanicLevel level, highest level of severity. Logs and then calls panic with the
	// message passed to Debug, Info, ...
	PanicLevel Level = iota
	// FatalLevel level. Logs and then calls `logger.Exit(1)`. It will exit even if the
	// logging level is set to Panic.
	FatalLevel
	// ErrorLevel level. Logs. Used for errors that should definitely be noted.
	// Commonly used for hooks to send errors to an error tracking service.
	ErrorLevel
	// WarnLevel level. Non-critical entries that deserve eyes.
	WarnLevel
	// InfoLevel level. General operational entries about what's going on inside the
	// application.
	InfoLevel
	// DebugLevel level. Usually only enabled when debugging. Very verbose logging.
	DebugLevel
	// TraceLevel level. Designates finer-grained informational events than the Debug.
	TraceLevel
)
```

### 2.3 日志内容对象Entry

entry是logrus的核心模块，每个entry相当于一行日志，主要用于控制日志内容的写入。

```go
type Entry struct {
	Logger *Logger

	// Contains all the fields set by the user.
	Data Fields

	// Time at which the log entry was created
	Time time.Time

	// Level the log entry was logged at: Trace, Debug, Info, Warn, Error, Fatal or Panic
	// This field will be set on entry firing and the value will be equal to the one in Logger struct field.
	Level Level

	// Calling method, with package name
	Caller *runtime.Frame

	// Message passed to Trace, Debug, Info, Warn, Error, Fatal or Panic
	Message string

	// When formatter is called in entry.log(), a Buffer may be set to entry
	Buffer *bytes.Buffer

	// Contains the context set by the user. Useful for hook processing etc.
	Context context.Context

	// err may contain a field formatting error
	err string
}
```

- **Logger**：所属的Logger对象，拥有上述Logger的内容，如io.Writer对象和Formatter对象等；
- **Data**：存储的是日志的结构化数据，其类型是`type Fields map[string]interface{}` 一个map，key是字符串型，value是任意类型，存储的是`log.WithFileds()`中的数据内容；
- **Time**：存储内容创建的时间；
- **Level**：和Logger对象一致，保存日志等级；
- **Caller**：记录调用方的信息；
- **Message**：字符串类型，记录日志内容，通常是`log.Info("msg")`中msg的内容；
- **Buffer**：当formatter调用entry.log()方法的时候设置；
- **Context**：用于记录上下文信息，主要用于hook操；
- **err**：存储错误信息，可能在日志内容格式化的时候出错；

## 3 核心功能解读

### 3.1 日志对象创建

日志对象的创建可以分为2部分，一是logger的创建，二是entry部分的创建；

**logger创建**

```go
func New() *Logger {
	return &Logger{
		Out:          os.Stderr, // 默认输出为stderr
		Formatter:    new(TextFormatter), // 默认格式为TextFormatter
		Hooks:        make(LevelHooks), // 创建hook，每个日志一个级别对应一个hook数组
		Level:        InfoLevel, // 默认日志级别为Info
		ExitFunc:     os.Exit, // 默认退出方法为os.Exit
		ReportCaller: false, // 默认不打印输出调用日志打印的函数的信息
	}
}
```

**entry创建**

```go
func (logger *Logger) newEntry() *Entry {
	entry, ok := logger.entryPool.Get().(*Entry)
	if ok {
		return entry
	}
	return NewEntry(logger)
}

func (logger *Logger) releaseEntry(entry *Entry) {
	entry.Data = map[string]interface{}{}
	logger.entryPool.Put(entry)
}

func NewEntry(logger *Logger) *Entry {
	return &Entry{
		Logger: logger,
		// Default is three fields, plus one optional.  Give a little extra room.
		Data: make(Fields, 6),
	}
}
```

在创建一个新的Entry对象时，使用`sync.Pool`对象 entryPool，主要保存Entry指针空对象，使用完后再放回sync.Pool中，防止在记录日志的时候大量开辟内存空间，触发GC操作，从而提高日志记录的速度。

### 3.2 记录日志

以info方法为例，其主要调用逻辑是Logger.Infoln -> Logger.LogIn -> Entry.LogIn -> Entry.log，核心逻辑位于Entry.log方法中。

```go
func (logger *Logger) Infoln(args ...interface{}) {
	logger.Logln(InfoLevel, args...)
}
func (logger *Logger) Logln(level Level, args ...interface{}) {
	if logger.IsLevelEnabled(level) { // 当前日志是否满足日志级别打印输出要求
		entry := logger.newEntry() // 获取entry对象
		entry.Logln(level, args...) // 调用entry的LogIn方法进行日志记录
		logger.releaseEntry(entry)
	}
}

func (entry *Entry) Logln(level Level, args ...interface{}) {
	if entry.Logger.IsLevelEnabled(level) { // 二次判断，感觉有点多余
		entry.Log(level, entry.sprintlnn(args...))
	}
}

// 主要核心方法
func (entry Entry) log(level Level, msg string) {
	var buffer *bytes.Buffer

	// Default to now, but allow users to override if they want.
	//
	// We don't have to worry about polluting future calls to Entry#log()
	// with this assignment because this function is declared with a
	// non-pointer receiver.
	if entry.Time.IsZero() {
		entry.Time = time.Now() // 默认日志记录中的时间为当前时间
	}

	entry.Level = level
	entry.Message = msg
	entry.Logger.mu.Lock() // 加锁
	if entry.Logger.ReportCaller {
		entry.Caller = getCaller() // 获取调用者信息
	}
	entry.Logger.mu.Unlock()
	// 执行hook逻辑
	entry.fireHooks()
	// 获取buffer，其实也是从sync.Pool中获取得到
	buffer = getBuffer()
	defer func() {
		entry.Buffer = nil
		putBuffer(buffer)
	}()
    // 重置buffer中的内容
	buffer.Reset()
	entry.Buffer = buffer
	// 执行formatter，记录日志
	entry.write()

	entry.Buffer = nil

	// To avoid Entry#log() returning a value that only would make sense for
	// panic() to use in Entry#Panic(), we avoid the allocation by checking
	// directly here.
	if level <= PanicLevel {
		panic(&entry)
	}
}
func (entry *Entry) write() {
    // 加锁，保证并发
	entry.Logger.mu.Lock()
	defer entry.Logger.mu.Unlock()
    // 对entry进行格式化
	serialized, err := entry.Logger.Formatter.Format(entry)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Failed to obtain reader, %v\n", err)
		return
	}
    // 将格式化后的内容记录到outer中
	if _, err = entry.Logger.Out.Write(serialized); err != nil {
		fmt.Fprintf(os.Stderr, "Failed to write to log, %v\n", err)
	}
}
```

### 3.3 Hook

从3.2可以看到，在执行entry.log时会调用entry.fireHooks()，其实这就是在执行hook。我们可以先看一下Hook的interface定义，Levels函数主要定义的是那些日志级别需要执行特殊操作，而具体执行的逻辑则由Fire函数进行处理。

```go
type Hook interface {
	Levels() []Level
	Fire(*Entry) error
}

// Internal type for storing the hooks on a logger instance.
type LevelHooks map[Level][]Hook

// Add a hook to an instance of logger. This is called with
// `log.Hooks.Add(new(MyHook))` where `MyHook` implements the `Hook` interface.
func (hooks LevelHooks) Add(hook Hook) {
	for _, level := range hook.Levels() {
		hooks[level] = append(hooks[level], hook)
	}
}

// Fire all the hooks for the passed level. Used by `entry.log` to fire
// appropriate hooks for a log entry.
func (hooks LevelHooks) Fire(level Level, entry *Entry) error {
	for _, hook := range hooks[level] {
		if err := hook.Fire(entry); err != nil {
			return err
		}
	}

	return nil
}
```

**hook的添加**

在Logger结构体中定义一个Hooks字段，类型为LevelHooks，用户可以通过log.Hooks.Add(new(MyHook))来添加自定义的hook，默认logrus日志库内置提供了3种hook实现，分别为syslog，test和writer。

**调用hook**

entry.log中会调用fireHooks方法，该方法会执行用户自定义的hook逻辑

```go
func (entry *Entry) fireHooks() {
	entry.Logger.mu.Lock()
	defer entry.Logger.mu.Unlock()
	err := entry.Logger.Hooks.Fire(entry.Level, entry)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Failed to fire hook: %v\n", err)
	}
}
```

社区其实也有很多的logrus hook，比如：

```text
elogrus: Logrus Hook for ElasticSearch,
logrus-graylog-hook: Graylog hook for logrus
logrus-logstash-hook:  Logstash hook for logrus
logrus_sentry: sentry hook for logrus
... ...
```

### 3.4 日志格式化

logrus自带两种格式输出，一种是`TextFormatter`，另外一种是`JsonFormatter`。

先来看看Formatter的interface的定义，就只有一个Format方法，就是将Entry中的数据内容，如Data字段存储的有结构的数据键值对，Message中存储的无结构数据，以及Time存储的日志记录时间等等，格式化成我们易读的数据格式

```go
type Formatter interface {
	Format(*Entry) ([]byte, error)
}
```

它有一个自定义的字段Key，主要就是上述提到的那些日志中的内容对应的key。

```go
const (
	defaultTimestampFormat = time.RFC3339
	FieldKeyMsg            = "msg"
	FieldKeyLevel          = "level"
	FieldKeyTime           = "time"
	FieldKeyLogrusError    = "logrus_error"
	FieldKeyFunc           = "func"
	FieldKeyFile           = "file"
)
```

以及一个基础的方法`prefixFieldClashes`，这个可以将日志中的一些字段进行进行重写，防止在进行数据格式化的时候覆盖了需要填充的字段内容。

```go
func prefixFieldClashes(data Fields, fieldMap FieldMap, reportCaller bool) {
	timeKey := fieldMap.resolve(FieldKeyTime)
	if t, ok := data[timeKey]; ok {
		data["fields."+timeKey] = t
		delete(data, timeKey)
	}
	...//省略了其他key的复写
}
```

以JsonFormatter为例，看下其Formatter核心逻辑：

```go
func (f *JSONFormatter) Format(entry *Entry) ([]byte, error) {
	data := make(Fields, len(entry.Data)+4)
	for k, v := range entry.Data {
		switch v := v.(type) {
		case error:
			// Otherwise errors are ignored by `encoding/json`
			// https://github.com/sirupsen/logrus/issues/137
			data[k] = v.Error()
		default:
			data[k] = v
		}
	}

	if f.DataKey != "" {
		newData := make(Fields, 4)
		newData[f.DataKey] = data
		data = newData
	}

	prefixFieldClashes(data, f.FieldMap, entry.HasCaller())

	timestampFormat := f.TimestampFormat
	if timestampFormat == "" {
		timestampFormat = defaultTimestampFormat
	}

	if entry.err != "" {
		data[f.FieldMap.resolve(FieldKeyLogrusError)] = entry.err
	}
	if !f.DisableTimestamp {
		data[f.FieldMap.resolve(FieldKeyTime)] = entry.Time.Format(timestampFormat)
	}
	data[f.FieldMap.resolve(FieldKeyMsg)] = entry.Message
	data[f.FieldMap.resolve(FieldKeyLevel)] = entry.Level.String()
	if entry.HasCaller() {
		funcVal := entry.Caller.Function
		fileVal := fmt.Sprintf("%s:%d", entry.Caller.File, entry.Caller.Line)
		if f.CallerPrettyfier != nil {
			funcVal, fileVal = f.CallerPrettyfier(entry.Caller)
		}
		if funcVal != "" {
			data[f.FieldMap.resolve(FieldKeyFunc)] = funcVal
		}
		if fileVal != "" {
			data[f.FieldMap.resolve(FieldKeyFile)] = fileVal
		}
	}

	var b *bytes.Buffer
	if entry.Buffer != nil {
		b = entry.Buffer
	} else {
		b = &bytes.Buffer{}
	}
	// 调用go sdk原生的json进行序列化
	encoder := json.NewEncoder(b)
	encoder.SetEscapeHTML(!f.DisableHTMLEscape)
	if f.PrettyPrint {
		encoder.SetIndent("", "  ")
	}
	if err := encoder.Encode(data); err != nil {
		return nil, fmt.Errorf("failed to marshal fields to JSON, %v", err)
	}

	return b.Bytes(), nil
}
```

除了内置的`TextFormatter`和`JSONFormatter`，还有不少第三方格式支持，比如：

```text
FluentdFormatter: Formats entries that can be parsed by Kubernetes and Google Container Engine.
GELF: Formats entries so they comply to Graylog's GELF 1.1 specification.
logstash: Logs fields as Logstash Events.
prefixed: Displays log entry source along with alternative layout.
zalgo: Invoking the Power of Zalgo.
nested-logrus-formatter: Converts logrus fields to a nested structure.
powerful-logrus-formatter: get fileName, log's line number and the latest function's name when print log; Sava log to files.
caption-json-formatter: logrus's message json formatter with human-readable caption added.
```

## 4 总结

logrus是go语言中使用较多的一个日志库，其使用简单，代码量也不大，可读性强。其利用sync.Pool增加了对象复用能力，降低GC，从而提供程序性能，通过mutex保证了线程安全性，并且默认提供一个Std的实现。

但是其jsonFormatter时，采用的是标准的json，性能相对不是很高，可以使用json-iterator进行替换。

## 5 参考

[1] https://github.com/sirupsen/logrus