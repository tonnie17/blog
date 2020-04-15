+++
title = "go database/sql使用误区"
date = 2020-04-15T00:00:00+08:00
tags = ["go"]
categories = [""]
draft = false
+++

- **在循环中defer**：在一个长生命周期的函数里进行`defer rows.Close()`，容易使连接数得不到释放，或defer堆栈过多，导致内存占用过多。
- **打开多个sql.DB对象**：使用一个全局的`sql.DB`对象，否则会频繁打开和关闭对db的tcp连接，导致延时，以及过多的`TIME-WAIT`。
- **结束后没有进行row.Close**：会导致连接得不到释放（回到连接池），导致占用内存增多，同理`db.QueryRow()`和`Scan()`应该在同一个链中使用。
- **单次使用的prepare statements**：会导致两次网络往返请求，增加不必要的耗时，如果可以考虑使用`fmt.Sprintf`来避免参数绑定和prepare statements。
- **在代码中使用strconv和cast来转换db type**：应该使用`Scan()`，它会在内部来帮你自动进行类型转换。
- **在代码中显示处理错误和重试**：应该借助`database/sql`的功能来处理连接池，重连和重试逻辑，而不是显式编写逻辑。
- **没有在rows.Next()后检查错误**：`rows.Next()`可能会非正常退出，如果没有在内部检查错误，需要在`rows.Next()`结束后用`rows.Err()`来捕获rows里存放的错误。
- **使用db.Query()来执行非SELECT查询**：应该使用`db.Exec()`。
- **假设两条连续查询使用了同一个连接**：使用db对象执行两次操作时使用的是两个不同的连接，如果分别进行了加锁和查询操作会导致一直阻塞，对于这类查询，应该使用`sql.Tx`或者同一个连接对象。
- **在使用事务时操作db对象**：因为`sql.Tx`对象绑定了一个事务，而db则不是，这时候可能不会得到想要的结果。
- **错误处理NULL对象**：不能把一个NULL对象scan到一个类型不是`sql.NullXXX`的变量中。
- **使用uint64作为参数**：如果使用一个uint64变量作为`Query()`，`Exec()`的参数，并且它的高位不为0，会抛出一个错误。

---

References:

- [learn-how-to-connect-golang-to-databases-with-database-sql](https://www.vividcortex.com/blog/learn-how-to-connect-golang-to-databases-with-database-sql)
