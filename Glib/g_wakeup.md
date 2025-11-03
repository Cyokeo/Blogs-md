使用一个pipe进行；
且pipe的两个管道均设置为nonblocking
mainContext监听管道的一侧fd？
另一侧通过写管道，唤醒maincontext？

void g_wakeup_signal (GWakeup *wakeup)

## 两种工作模式
```c
void
g_wakeup_signal (GWakeup *wakeup)
{
int res;
if (wakeup->fds[1] == -1)
{

uint64_t one = 1;
/* eventfd() case. It requires a 64-bit counter increment value to be
* written. */
do
res = write (wakeup->fds[0], &one, sizeof one);
while (G_UNLIKELY (res == -1 && errno == EINTR));
}
else
{
uint8_t one = 1;
/* Non-eventfd() case. Only a single byte needs to be written, and it can

* have an arbitrary value. */
do
res = write (wakeup->fds[1], &one, sizeof one);
while (G_UNLIKELY (res == -1 && errno == EINTR));
}
}
```

```c
void
g_wakeup_acknowledge (GWakeup *wakeup)
{
int res;
if (wakeup->fds[1] == -1)
{
uint64_t value;
/* eventfd() read resets counter */
do
res = read (wakeup->fds[0], &value, sizeof (value));
while (G_UNLIKELY (res == -1 && errno == EINTR));
}
else
{
uint8_t value;
/* read until it is empty */
do
res = read (wakeup->fds[0], &value, sizeof (value));
while (res == sizeof (value) || G_UNLIKELY (res == -1 && errno == EINTR));
}
}
```

## how to define a function which is safe to call from a UNIX signal handler ???

