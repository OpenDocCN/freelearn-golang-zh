# 3

# 处理日期和时间

在任何编程语言中处理日期和时间都可能很困难。Go 的标准库提供了易于使用的工具来处理日期和时间结构。这些可能与许多人习惯的不同。例如，不同语言中的库会在 `time` 类型与 `date` 类型之间做出区分。Go 的标准库只包含 `time.Time` 类型。这可能会让你在处理 Go 的时间时感到困惑。

我希望认为 Go 对日期/时间的处理减少了创建微妙错误的机会。你看，当你谈论时间时，你必须非常小心和明确你所说的意思：你是谈论一个时间点还是一个时间段？实际上，日期是一个时间段（例如，08/01/2024 从 08/01/2024T00:00:00 开始，一直持续到 08/01/2024T23:59:59），尽管通常这不是意图。特定的日期/时间值也取决于你测量时间的位置。在科罗拉多州的丹佛，2023-11-05T08:00 与在德国柏林的 2023-11-05T08:00 是不同的。时间总是向前移动，但日期/时间可能会跳过或倒退：在科罗拉多州的丹佛，2023-11-05T02:59 之后，时间会倒退到 2023-11-05T02:00，因为那是科罗拉多州夏令时结束的时候。因此，实际上对于 2023-11-05T02:10:10 有两个时间实例，一个在山地夏令时，另一个在山地标准时。

目前生产中的许多软件错误都处理时间不正确。例如，如果你在计算客户订阅何时结束，你必须考虑该客户的位置和订阅结束的时间，否则，他们的订阅可能在最后一天提前（或延迟）结束。

本章包含以下关于正确处理日期/时间的食谱：

+   处理 Unix 时间

+   日期/时间组件

+   日期/时间算术

+   日期/时间的格式化和解析

+   处理时区

+   计时器

+   计时器

+   存储时间信息

# 处理 Unix 时间

Unix 时间是从 1970 年 1 月 1 日 UTC（纪元）开始经过的秒数（或毫秒、微秒或纳秒）。Go 使用 `int64` 来表示这些值，因此 Unix 时间以秒为单位可以表示过去或未来数十亿年的时间。Unix 时间以纳秒为单位可以表示 1678 年至 2262 年之间的日期值。Unix 时间是自纪元以来（或直到纪元）的绝对时间实例度量。它是独立于位置的，因此如果有两个 Unix 时间，`s` 和 `t`，如果 `s<t`，则 `s` 发生在 `t` 之前，无论位置如何。由于这些属性，Unix 时间通常用作标记事件发生时间（日志写入时、记录插入时等）的时间戳。

## 如何做到...

+   获取当前 Unix 时间，请使用以下方法：

    +   `time.Now().Unix() int64`：Unix 时间以秒为单位

    +   `time.Now().UnixMilli() int64`：Unix 时间以毫秒为单位

    +   `time.Now().UnixMicro() int64`：Unix 时间以微秒为单位

    +   `time.Now().UnixNano() int64`: 以纳秒为单位的 Unix 时间

+   给定一个 Unix 时间，使用以下方法将其转换为`time.Time`类型：

    +   `time.Unix(sec, nanosec int64) time.Time`: 将秒和/或纳秒级的 Unix 时间转换为`time.Time`

    +   `time.UnixMilli(int64) time.Time`: 将毫秒级的 Unix 时间转换为`time.Time`

    +   `time.UnixMicro(int64) time.Time`: 将微秒级的 Unix 时间转换为`time.Time`

+   要将 Unix 时间转换为本地时间，请使用`localTime := time.Unix(unixTimeSeconds,0).In(location)`，其中`location`是要解释 Unix 时间的位置的`*time.Location`

# 日期/时间组成部分

当处理日期值时，你通常需要从其组成部分组合一个日期/时间，或者需要访问日期/时间的组成部分。这个菜谱展示了如何做到这一点。

## 如何操作...

+   要从各个部分构建日期/时间值，请使用`time.Date`函数

+   要获取日期/时间值的各个部分，请使用`time.Time`方法：

    +   `time.Day() int`

    +   `time.Month() time.Month`

    +   `time.Year() int`

    +   `time.Date() (year, month, day int)`

    +   `time.Hour() int`

    +   `time.Minute() int`

    +   `time.Second() int`

    +   `time.Nanosecond() int`

    +   `time.Zone() (name string,offset int)`

    +   `time.Location() *time.Location`

`time.Date`将从其组成部分创建一个时间值：

```go
d := time.Date(2020, 3, 31, 15, 30, 0, 0, time.UTC)
fmt.Println(d)
// 2020-03-31 15:30:00 +0000 UTC
```

输出将被标准化，如下所示：

```go
d := time.Date(2020, 3, 0, 15, 30, 0, 0, time.UTC)
fmt.Println(d)
// 2020-02-29 15:30:00 +0000 UTC
```

由于月份的日期从 1 日开始，使用`0`天创建的日期将导致上一个月的最后一天。

一旦你有一个`time.Time`值，你可以获取其组成部分：

```go
d := time.Date(2020, 3, 0, 15, 30, 0, 0, time.UTC)
fmt.Println(d.Day())
// 29
```

再次强调，`time.Date`会标准化日期值，所以`d.Day()`将返回`29`。

# 日期/时间算术

日期/时间算术对于回答以下问题等是必要的：

+   完成一次操作需要多长时间？

+   5 分钟后将会是什么时间？

+   下个月还有多少天？

这个菜谱展示了如何使用`time`包来回答这些问题。

## 如何操作...

+   要找出两个时间实例之间经过的时间，请使用`Time.Sub`方法来减去它们。

+   要找出从现在到较晚时间的时间间隔，请使用`time.Until(laterTime)`。

+   要找出从给定时间以来经过的时间，请使用`time.Since(beforeTime)`。

+   要找出经过一定时间后将会是什么时间，请使用`Time.Add`方法。使用负持续时间来查找在某个时间之前的时间。

+   要在时间上添加/减去年、月或日，请使用`Time.AddDate`方法。

+   要比较两个`time.Time`值，请使用以下方法：

    +   使用`Time.Equal`来检查两个时间值是否表示相同的实例

    +   使用`Time.Before`或`Time.After`来检查时间值是否在给定时间值之前或之后

## 它是如何工作的...

`time.Duration`类型表示两个实例之间的时间间隔（以纳秒为单位）作为一个`int64`值。换句话说，如果你从一个`time.Time`值减去另一个值，你得到一个`time.Duration`：

```go
dur := tm1.Sub(tm2)
```

由于`Duration`是一个表示纳秒的`int64`，你可以进行持续时间算术：

```go
// Add 1 day to duration
dur+=time.Hour*24
```

注意前面提到的最后一个操作也涉及到乘法，因为 `time.Hour` 本身就是 `time.Duration` 类型。

你可以将持续时间值添加到 `time.Time` 值中：

```go
now := time.Now()
then := now.Add(dur)
```

小贴士

持续时间是一个 `int64` 类型意味着 `time.Duration` 值限制在大约 290 年左右。这对于大多数实际案例应该足够了。然而，如果你不满足这种情况，你需要为自己构建解决方案或寻找第三方库。

你可以通过添加一个负持续时间值从 `time.Time` 值中减去持续时间：

```go
fmt.Println( then.Add(-dur).Equal(now) )
```

注意 `Time.Equal` 方法的使用。它比较两个时间实例，考虑到它们的时间区可能不同。例如，`Time.Equal` 将对 `2024-01-09 09:00 MST` 和 `2024-01-09 08:00 PST` 返回 `true`。

使用 `Time.Before` 和 `Time.After` 来比较时间值。例如，你可以通过以下方式检查一个具有到期日期的对象是否已过期：

```go
if object.Expiration.After(time.Now()) {
   // Object expired
}
```

你也可以给一个给定的日期添加年/月/日：

```go
t:=time.Now()
// Subtract 1 year from now to get this moment in last year
lastYear := t.AddDate(-1,0,0)
// Add 1 day to get same time tomorrow
tomorrow := t.AddDate(0,0,1)
// Add 1 day to get the next month
nextMonth := t.AddDate(0,1,0)
```

这些操作的结果将被标准化。例如，如果你从 `2020-02-29` 减去一年，你会得到 `2019-03-01`。这在你处理月底的日期并需要加减月份值时会引起问题。将月份加到 `2020-03-31` 两次将得到 `2020-06-01`，但加两个月将得到 `2020-05-31`：

```go
d := time.Date(2020, 3, 31, 0, 0, 0, 0, time.UTC)
fmt.Println(d.AddDate(0, 1, 0).AddDate(0, 1, 0))
// 2020-06-01 00:00:00 +0000 UTC
fmt.Println(d.AddDate(0, 2, 0))
// 2020-05-31 00:00:00 +0000 UTC
```

# 日期/时间的格式化和解析

Go 使用了一个有趣且有些有争议的日期/时间格式化方案。日期/时间格式使用一个特定的时间点来表示，调整后使得日期/时间的每个组成部分都是唯一的数字：

+   1 是月份：“Jan” “January” “01” “1”

+   2 是月份中的天：“2” “_2” “02”

+   3 是 12 小时制中的小时：“15” “3” “03”

+   15 是 24 小时制中的小时，

+   4 是分钟：“4” “04”

+   5 是秒：“5” “05”

+   6 是年份：“2006” “06”

+   MST 是时区：“-0700” “-07:00” “-07” “-070000” “-07:00:00” “MST”

+   0 是填充了 0 的毫秒：“0” “000”

+   9 是未填充的毫秒：“9” “999”

## 如何做到这一点...

+   使用 `time.Parse` 和适当的格式来解析日期/时间。在格式中未指定的日期/时间部分将被初始化为其零值，月份为 1 月，年份为 1，月份中的天为 1，其余部分为 0。如果缺少时区信息，解析的日期/时间将使用 UTC。

+   使用 `time.ParseInLocation` 在指定位置解析日期/时间。时区将根据日期值和位置确定。

+   使用 `Format()` 方法来格式化日期/时间值。

```go
func main() {
  t := time.Date(2024, 3, 8, 18, 2, 13, 500, time.UTC)
  fmt.Println("Date in yyyy/mm/dd format", t.Format("2006/01/02"))
  // Date in yyyy/mm/dd format 2024/03/08
  fmt.Println("Date in yyyy/m/d format", t.Format("2006/1/2"))
  // Date in yyyy/m/d format 2024/3/8
  fmt.Println("Date in yy/m/d format", t.Format("06/1/2"))
  // Date in yy/m/d format 24/3/8
  fmt.Println("Time in hh:mm format (12 hr)", t.Format("03:04"))
  // Time in hh:mm format (12 hr) 06:02
  fmt.Println("Time in hh:m format (24 hr)", t.Format("15:4"))
  // Time in hh:m format (24 hr) 18:2
  fmt.Println("Date-time with time zone", t.Format("2006-01-02 
  13:04:05 -07:00"))
  // Date-time with time zone 2024-03-08 36:02:13 +00:00
}
```

时区根据位置和日期而变化。在以下示例中，即使使用相同的位置来解析日期，时区也会变化，因为 7 月 9 日是山地夏令时，而 1 月 9 日是山地标准时：

```go
loc, _ := time.LoadLocation("America/Denver")
const format = "Jan 2, 2006 at 3:04pm"
str, _ := time.ParseInLocation(format, "Jul 9, 2012 at 5:02am", loc)
fmt.Println(str)
// 2012-07-09 05:02:00 -0600 MDT
str, _ = time.ParseInLocation(format, "Jan 9, 2012 at 5:02am", loc)
fmt.Println(str)
// 2012-01-09 05:02:00 -0700 MST
```

# 处理时区

Go 的 `time.Time` 值包括 `time.Location`，这可以是两种情况之一：

+   一个真实的位置，例如 `America/Denver`。如果是这种情况，实际时区将取决于时间值。对于 `Denver`，时区将是 `MDT`（山地夏令时）或 `MST`（山地标准时），具体取决于实际的时间值。

+   一个提供偏移量的固定时区。

一些应用程序使用 **本地时间**。这是在特定位置捕获的日期/时间值，并在任何地方解释为相同的值，而不是解释为相同的时间点。生日（因此，年龄）通常使用本地时间来解释。也就是说，如果你在 2005-07-14 出生，你将在 2007-07-14 00:00（东部时区）在纽约被认为是 2 岁，但在同一点时间在洛杉矶，即 2007-07-13 21:00（太平洋时区），你仍然是 1 岁。

## 如何做到这一点...

如果你正在处理时间点，始终使用相关位置捕获日期/时间值。这些日期/时间值可以轻松转换为其他时区。

如果你正在处理多个时区的本地时间，在新的位置或时区中重新创建 `time.Time` 以进行转换。

## 它是如何工作的...

当你创建一个 `time.Time` 时，它总是与一个位置相关联：

```go
// Create a new time using the local time zone
t := time.Date(2021,12,31,15,0,0,0, time.Local)
// 2021-12-31 15:00:00 -0700 MST
```

一旦你有一个 `time.Time`，你就可以在不同的时区中获取同一时间点：

```go
utcTime := t.In(time.UTC)
fmt.Println(utcTime)
// 2021-12-31 22:00:00 +0000 UTC
ny,err:=time.LoadLocation("America/New_York")
if err!=nil {
  panic(err)
}
nyTime := t.In(ny)
fmt.Println(nyTime)
// 2021-12-31 17:00:00 -0500 EST
```

这些是在不同时区中同一时间点的不同表示。

你也可以创建一个自定义时区：

```go
zone30 := time.FixedZone("30min", 30)
fmt.Println(t.In(zone30))
// 2021-12-31 22:00:30 +0000 30min
```

当你处理本地时间时，你会丢弃位置和时间区域信息：

```go
// Create a local time, UTC zone
t := time.Date(2021,12,31,15,0,0,0, time.UTC)
// 2021-12-31 15:00:00 +0000 UTC
```

要在纽约获取相同的时间值，请使用以下方法：

```go
ny,err:=time.LoadLocation("America/New_York")
if err!=nil {
  panic(err)
}
nyTime := time.Date(t.Year(), t.Month(), t.Day(), t.Hour(), t.Minute(), t.Second(), t.Nanosecond(), ny)
fmt.Println(nyTime)
// 2021-12-31 15:00:00 -0500 EST
```

# 存储时间信息

一个常见问题是将日期/时间信息以可移植的方式存储在数据库、文件中等，以便可以正确解释。

## 如何做到这一点...

你应该首先确定确切的需求：你需要存储一个时间点还是一天中的时间？

+   要存储一个时间点，执行以下操作之一：

    +   在所需粒度上存储 Unix 时间（即 `time.Unix` 用于秒，`time.UnixMilli` 用于毫秒等）。

    +   存储 UTC 时间 (`time.UTC()`)

+   要存储一天中的时间，存储表示一天中瞬间的 `time.Duration` 值。以下函数计算该天内的瞬间作为 `time.Duration`：

    ```go
    func GetTimeOfDay(t time.Time) time.Duration {
      beginningOfDay:=time.Date(t.Year(),t.Month(),t.
      Day(),0,0,0,0,t.Location())
      return t.Sub(beginningOfDay)
    }
    ```

+   要存储日期值，你可以清除 `time.Time` 的时间部分：

    ```go
    date:=time.Date(t.Year(), t.Month(), t.Day(), 0,0,0,0,t.Location())
    ```

    注意，以这种方式存储的日期比较可能会出现问题，因为每个时区都会将每天解释为不同的瞬间。

# 定时器

使用 `time.Timer` 来安排将来要执行的一些工作。当定时器到期时，你将从一个通道接收到一个信号。你可以使用定时器在以后运行一个函数或取消运行时间过长的进程。

## 如何做到这一点...

你可以通过两种方式之一创建一个定时器：

+   使用 `time.NewTimer` 或 `time.After`。定时器将在到期时通过通道发送一个信号。使用 `select` 语句或从通道读取以接收定时器到期信号。

+   使用 `time.AfterFunc` 在计时器到期时调用一个函数。

## 它是如何工作的...

使用 `time.Duration` 创建 `time.Timer` 计时器：

```go
// Create a 10-second timer
timer := time.NewTimer(time.Second*10)
```

计时器包含一个通道，在 10 秒后将会接收到当前的时间戳。计时器创建时通道容量为 `1`，因此计时器运行时总能向该通道写入并停止计时器。换句话说，如果你未能从计时器中读取，它不会泄漏；它最终会被垃圾回收。

计时器可以用来停止一个长时间运行的过程：

```go
func longProcess() {
  timer := time.NewTimer(time.Second*10)
  for {
     processData()
     select {
       case <-timer.C:
          // 2 seconds passed
          return
       default:
     }
  }
}
```

以下示例展示了如何使用计时器来限制函数返回所需的时间。如果计算在一秒内完成，则返回响应。如果计算时间更长，则函数返回一个调用者可以使用以接收结果的通道。此函数还演示了如何停止计时器：

```go
func longComputation() (concurrent chan Result, result Result) {
  timer:=time.NewTimer(time.Second)
  concurrent=make(chan Result)
  // Start the concurrent computation. Its result will be received 
  // from the channel
  go func() {
     concurrent <- processData()
  }()
  // Wait until result is available, or timer expires
  select {
     case result:=<-concurrent:
        // Result became available quickly. Stop the timer and return 
        //the result.
        timer.Stop()
        return nil,result
     case <-timer.C:
        // Timer expired before result is computed. Return the channel
        return concurrent,Result{}
  }
}
```

注意，计时器可能在调用 `timer.Stop()` 之前就到期了。这是可以的。计时器最终都会到期并被垃圾回收。调用 `timer.Stop()` 只是为了防止计时器比必要的持续时间更长。

提示

当另一个 goroutine 正在监听计时器时，你不能并发地调用 `Timer.Stop`。所以，如果你必须调用 `Timer.Stop`，请从监听计时器通道的同一个 goroutine 中调用它。

同样可以使用 `time.After` 实现：

```go
  concurrent=make(chan Result)
  // Start the concurrent computation. Its result will be received 
  // from the channel
  go func() {
     concurrent <- processData()
  }()
  select {
     case result:=<-concurrent:
        return nil,result
     case <-time.After(time.Second):
        return concurrent,Result{}
  }
```

# 计时器

使用 `time.Ticker` 定期执行任务。你将通过通道定期接收到信号。与 `time.Timer` 不同，你必须小心处理计时器。如果你忘记停止计时器，一旦超出作用域，它就不会被垃圾回收，并且会发生泄漏。

## 如何做到这一点...

1.  使用 `time.Ticker` 创建一个新的计时器。

1.  从计时器的通道读取以接收周期性的滴答声。

1.  当你完成对计时器的使用后，停止它。你不需要排空计时器的通道。

## 它是如何工作的...

使用计时器进行周期性事件。一个常见的模式如下：

```go
func poorMansClock(done chan struct{}) {
  // Create a new ticker with a 1 second period
  ticker:=time.NewTicker(time.Second)
  // Stop the ticker once we're done
  defer ticker.Stop()
  for {
    select {
      case <-done:
         return
      case <-ticker.C:
         fmt.Println(time.Now())
    }
  }
}
```

如果你错过了滴答声会发生什么？如果你运行了一个长时间的过程，阻止你监听计时器通道，那么当你再次开始监听时，计时器会发送大量的滴答声吗？

与 `time.Timer` 类似，`time.Ticker` 也使用一个容量为 `1` 的通道。因此，如果你不从这个通道读取，它最多只能存储一个滴答声。当你再次从通道开始监听时，你会立即接收到你错过的滴答声，以及在其周期到期时的下一个滴答声。例如，考虑以下每秒调用给定函数的程序：

```go
func everySecond(f func(), done chan struct{}) {
  // Create a new ticker with a 1 second period
  ticker:=time.NewTicker(time.Second)
  start:=time.Now()
  // Stop the ticker once we're done
  defer ticker.Stop()
  for {
    select {
      case <-done:
         return
      case <-ticker.C:
         fmt.Println(time.Since(start).Milliseconds())
         // Call the function
         f()
    }
  }
}
```

假设第一次调用 `f()` 运行时间为 10 毫秒，但第二次调用运行时间为 1.5 秒。在 `f()` 运行期间，没有人从计时器的通道读取，因此会错过一个滴答声。一旦 `f()` 返回，`select` 语句将立即读取这个错过的滴答声，并在 500 毫秒后接收到下一个滴答声。输出看起来像这样：

```go
1000
2000
3500
4000
5000
...
```

提示

与`time.Timer`不同，你可以在从其通道读取的同时并发地停止一个计时器。
