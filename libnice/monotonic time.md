

[`CLOCK_REALTIME` vs `CLOCK_MONOTONIC`](https://stackoverflow.com/posts/3527632/timeline)

`CLOCK_REALTIME` represents the machine's best-guess as to the current wall-clock, time-of-day time. As [Ignacio](https://stackoverflow.com/questions/3523442/difference-between-clock-realtime-and-clock-monotonic/3523482#3523482) and [MarkR](https://stackoverflow.com/questions/3523442/difference-between-clock-realtime-and-clock-monotonic/3523857#3523857) say, this means that `CLOCK_REALTIME` can jump forwards and backwards as the system time-of-day clock is changed, including by NTP.

`CLOCK_MONOTONIC` represents the absolute elapsed wall-clock time since some arbitrary, fixed point in the past. It isn't affected by changes in the system time-of-day clock.

If you want to compute the elapsed time between two events observed on the one machine without an intervening reboot, `CLOCK_MONOTONIC` is the best option.
