## 消灭 Lost Wakeup：确保安全的 Sleep 与 Wakeup 机制

为了消除 `lost wakeup` 问题，我们需要确保在释放锁和将进程状态设置为 `SLEEPING` 之间没有时间窗口。下面详细解释如何通过稍微复杂的 `sleep` 函数设计，避免这个问题。

### 1. 问题分析：窗口时间的消除

在原始代码中，存在一个关键的时间窗口：

```c
release(&uart_tx_lock);
// 这里可能会发生中断
broken_sleep(&tx_chan);
acquire(&uart_tx_lock);
```

**问题描述**：

- 我们必须释放 `uart_tx_lock`，因为中断处理程序需要获取这个锁。
- 然而，释放锁之后、线程将自身标记为 `SLEEPING` 之前，如果发生中断，可能导致 `wakeup` 在没有任何线程处于 `SLEEPING` 状态时被调用，进而导致 `lost wakeup`。

**解决方案**：

- 通过让 `sleep` 函数在原子操作中同时释放锁并将进程设置为 `SLEEPING`，我们可以消除这个时间窗口。

### 2. Sleep 函数的改进

为了实现上述目标，我们需要让 `sleep` 函数接受一个锁作为参数，并在内部原子性地处理两个操作：
1. 将进程状态设置为 `SLEEPING`。
2. 释放锁。

这样可以确保 `wakeup` 在中断处理程序中检查进程状态时，不会看到锁已经被释放但进程尚未进入 `SLEEPING` 状态的情况。

**接口层面的承诺**：

- `sleep` 函数承诺在释放锁的同时，将进程状态设置为 `SLEEPING`，这是一个原子操作。这样可以确保在调用 `wakeup` 时，进程已经处于可被唤醒的状态。

### 3. Wakeup 函数的实现

`wakeup` 函数在操作系统的进程表中查找所有处于 `SLEEPING` 状态并匹配指定 `channel` 的进程，并将这些进程的状态设置为 `RUNNABLE`。在这个过程中，确保对每个进程的操作都被锁保护，以避免竞态条件。

```c
// 唤醒所有在 chan 上睡眠的进程。
// 调用时必须未持有任何 p->lock。
void wakeup(void *chan)
{
  struct proc *p;

  for(p = proc; p < &proc[NPROC]; p++) {
    if(p != myproc()){
      acquire(&p->lock); // 获取进程的锁
      if(p->state == SLEEPING && p->chan == chan) {
        p->state = RUNNABLE; // 将进程状态设置为 RUNNABLE
      }
      release(&p->lock); // 释放锁
    }
  }
}
```

**关键点**：
- 在操作进程状态前，`wakeup` 函数会获取进程的锁，确保在修改进程状态时没有其他线程能够干扰。
- 只有在进程的状态为 `SLEEPING` 且其 `channel` 与 `wakeup` 的参数匹配时，进程的状态才会被设置为 `RUNNABLE`，这样该进程就可以被调度执行。

### 4. 遵守规则避免 Lost Wakeup

**规则总结**：
- **Sleep 函数**：当调用 `sleep` 函数时，必须传入保护条件的锁。`sleep` 函数负责在原子操作中释放锁并将进程状态设置为 `SLEEPING`。
- **Wakeup 函数**：调用 `wakeup` 函数时，必须持有与之关联的锁，以确保唤醒操作的一致性。

通过这些规则，我们确保在 `sleep` 与 `wakeup` 的执行过程中，锁始终保护着关键操作，避免了竞态条件和 `lost wakeup` 问题的发生。



## 带锁的 `sleep` 函数：解决 Lost Wakeup 问题的关键

在我们深入了解了 `sleep` 函数的实现后，可以清楚地看到它是如何通过一系列原子操作和锁机制来避免 `lost wakeup` 问题的。以下是对 `sleep` 函数及其在 `UART` 代码中的应用的详细分析。

### 1. `sleep` 函数的实现与关键步骤

```c
// Atomically release lock and sleep on chan.
// Reacquires lock when awakened.
void sleep(void *chan, struct spinlock *lk)
{
  struct proc *p = myproc();
  
  // Must acquire p->lock in order to
  // change p->state and then call sched.
  // Once we hold p->lock, we can be
  // guaranteed that we won't miss any wakeup
  // (wakeup locks p->lock),
  // so it's okay to release lk.

  acquire(&p->lock);  // 获取进程锁，确保接下来的操作是安全的
  release(lk);  // 释放传入的条件锁，允许其他线程获取该锁

  // 进入睡眠状态
  p->chan = chan;
  p->state = SLEEPING;

  sched();  // 调用 sched 函数，放弃 CPU，切换到调度器线程

  // 线程被唤醒后，清理工作
  p->chan = 0;

  // 重新获取传入的条件锁
  release(&p->lock);
  acquire(lk);
}
```

**关键步骤**：

1. **获取进程锁 `p->lock`**：
   - 在修改进程状态（如设置为 `SLEEPING`）之前，必须先获取进程的锁 `p->lock`。
   - 这一步确保在 `sleep` 函数执行的过程中，`wakeup` 函数无法在没有获取进程锁的情况下唤醒进程。

2. **释放条件锁 `lk`**：
   - 释放传入的条件锁 `lk`，使得其他线程（如中断处理程序）能够获取该锁进行操作。
   - 这一操作是在持有进程锁 `p->lock` 的前提下进行的，从而确保在锁释放后，如果 `wakeup` 被调用，它无法在进程未进入 `SLEEPING` 状态之前唤醒该进程。

3. **设置进程状态为 `SLEEPING` 并记录 `sleep channel`**：
   - 在持有进程锁的情况下，安全地将进程状态设置为 `SLEEPING`，并记录 `chan`，标识该进程在等待的事件。

4. **调用 `sched` 切换进程**：
   - 调用 `sched` 函数，进行上下文切换，当前进程进入睡眠，等待被唤醒。
   - 在调用 `sched` 函数期间，进程锁依然被持有，直到切换到调度器线程后才释放。

5. **清理工作与重新获取锁**：
   - 当进程被唤醒后，`sched` 返回，线程继续执行。此时，清理 `chan`，并重新获取原来的条件锁 `lk`。

### 2. `UART` 代码中 `sleep` 的应用

```c
// transmit buf[].
void uartwrite(char buf[], int n)
{
  acquire(&uart_tx_lock);  // 获取条件锁

  int i = 0;
  while(i < n){
    while(tx_done == 0){  // 检查条件
      sleep(&tx_chan, &uart_tx_lock);  // 如果条件不满足，调用 sleep
    }
    WriteReg(THR, buf[i]);
    i += 1;
    tx_done = 0;
  }

  release(&uart_tx_lock);  // 释放条件锁
}
```

**`UART` 代码分析**：
- 在 `uartwrite` 函数中，条件锁 `uart_tx_lock` 在一开始就被获取，并在循环的每一轮中保持，直到线程需要进入睡眠状态。
- 当 `tx_done` 为 0 时，意味着当前传输尚未完成，此时线程调用 `sleep` 进入睡眠，等待下一个传输中断信号。
- `sleep` 函数在 `uart_tx_lock` 被释放之前获取进程锁 `p->lock`，确保不会在中途被错误地唤醒。
- 通过上述操作，`sleep` 和 `wakeup` 之间的时序问题得到解决，不会出现 `lost wakeup` 的情况。

### 3. 避免 `Lost Wakeup` 的机制总结

- **持有锁的时机**：
  - `sleep` 函数的核心在于确保在释放条件锁 `lk` 和设置进程状态之间，持有进程锁 `p->lock`。这样，`wakeup` 函数必须等到进程锁被释放后，才能操作进程的状态，从而避免 `lost wakeup`。

- **规则与流程**：
  - 调用 `sleep` 时，必须持有条件锁 `lk`，以便在安全的环境下修改条件。
  - `sleep` 函数通过持有进程锁并原子地执行释放条件锁和设置进程状态这两步，避免了中断处理程序在不合适的时间唤醒线程。
  - `wakeup` 函数在检查和修改进程状态时，必须持有进程锁 `p->lock`，确保状态的改变不会发生竞态条件。

通过这些规则和流程，`lost wakeup` 问题被彻底解决，从而确保线程在并发环境下的正确协调和唤醒。
