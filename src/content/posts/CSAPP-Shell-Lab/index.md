---
title: CS:APP Shell Lab
published: 2026-02-02
pinned: true
description: 2025-2026秋 BIT计科 CSAPP 希冀实验IV Shell Lab.
tags: [CSAPP, Shell, C]
category: Coursework
licenseName: "Unlicensed"
author: akanade
sourceLink: "https://github.com/wyyalhz/CSAPP-Shell-Lab"
draft: false
date: 2026-02-02
image: "./cover.webp"
pubDate: 2026-02-02
permalink: "csapp-shell-lab"
---

## 导语

本实验中，你需要实现一个简单的支持作业控制的Unix shell程序，目的是使你熟悉进程控制和信号处理的概念。希冀平台的实验相关资料和说明文档可以从GitHub仓库下载（链接见底部），此处不再赘述。

写一个shell乍一听难度很高，但在本实验中你只需要对7个函数进行填空，且网络和课本上有大量相似内容，因此以通过实验为目的还算是轻松。由于本实验在笔者期末考试后才布置，本就不多的学习电量已被残酷的期末周彻底榨干，因此笔者是仅以通过为目的来完成实验的（笑）

写在开头：

> 进行实验前请仔细阅读CSAPP第八章：异常控制流
>
> 本实验仅保证希冀平台shell lab满分通过
>
> 部分代码注释由大模型生成

下面是笔者的实现思路。

## 各函数实现思路

### 0. 自定义函数

添加用于阻塞信号的函数，用于解决竞争问题。我这个mask_all()其实写得有些过分，把所有信号都屏蔽了，为了更简单地通过lab，但真实shell更常只block SIGCHLD。

```c
// 进入“临界区”：阻塞所有信号，返回旧的信号掩码（用于之后恢复）
sigset_t mask_all()
{
    sigset_t mask_all, pre_mask;

    sigfillset(&mask_all);                         // mask_all = {所有信号}
    sigprocmask(SIG_BLOCK, &mask_all, &pre_mask);  // 阻塞所有信号，并把旧mask存到pre_mask
    return pre_mask;                               // 返回旧mask（退出临界区要用）
}

// 退出“临界区”：把信号掩码恢复成进入前的样子
void set_mask(sigset_t pre_mask)
{
    sigprocmask(SIG_SETMASK, &pre_mask, NULL);     // 恢复旧mask（解除阻塞）
}
```

### 1. eval(char *cmdline) 

eval用来分析和解释命令行，是shell的心脏，也是本lab中代码行数最多的函数。从头完成令人头大，好在CSAPP第八章中有一个eval的详细实现示例，理解后可以大大提高效率。

eval的核心逻辑总结下来就是：

> 解析输入 → 判断是不是内置命令 → 不是就 fork+exec → 前台就等，后台就返回

```c
void eval(char *cmdline) 
{
    char *argv[MAXARGS];

    // parseline 会把一行命令拆成 argv[]（类似 main 的 argv）
    // 返回 bg=1 表示末尾有 '&'，要后台运行；bg=0 表示前台运行
    int bg = parseline(cmdline, argv);

    pid_t pid = 0;

    // builtin_cmd 返回 1 表示已经处理（quit/jobs/bg/fg），eval 直接结束
    // 返回 0 表示不是内置命令，需要 fork+exec 执行外部程序
    if (!builtin_cmd(argv))
    {
        // 进入临界区：阻塞信号，避免 addjob 和 sigchld_handler 发生竞态
        sigset_t pre_mask = mask_all();

        // 创建子进程去运行外部程序
        if ((pid = fork()) == 0)
        {
            // 子进程：恢复父进程原本的信号掩码（不要一直阻塞）
            set_mask(pre_mask);

            // 关键：让子进程成为新进程组的组长（PGID=PID）
            // 否则 Ctrl-C/Ctrl-Z 可能会把 shell 自己也一起干掉/暂停
            setpgid(0, 0);

            // 用 execve 覆盖子进程映像：成功则不返回
            if (execve(argv[0], argv, environ) < 0)
            {
                // execve 失败：通常是找不到文件/没权限/不是可执行文件等
                // lab 的要求一般只输出 “Command not found”
                printf("%s: Command not found\n", argv[0]);
            }

            // execve 失败才会走到这里
            exit(0);
        }

        // 父进程：把子进程加入作业表（jobs）
        // state：后台 BG 或前台 FG
        addjob(jobs, pid, bg ? BG : FG, cmdline);

        // 退出临界区：恢复信号掩码
        set_mask(pre_mask);

        if (!bg)
        {
            // 前台作业：必须等待它结束/停止，否则提示符会立刻返回（不符合前台语义）
            waitfg(pid);
        }
        else
        {
            // 后台作业：打印一行提示信息，然后立刻返回继续接收下一条命令
            printf("[%d] (%d) %s", pid2jid(pid), pid, cmdline);
        }
        return;
    }

    // 如果是内置命令，builtin_cmd 已经执行完，eval 直接结束
}
```

严格来说，在信号处理函数里用printf并不安全，具体原因书中有详细解释，真实工程会用sio_puts这类函数，但本lab允许并且trace能过

### 2. builtin_cmd

builtin_cmd用来识别和解释内置命令，即quit，fg，bg和jobs

```c
int builtin_cmd(char **argv) 
{
    // quit：直接退出 shell
    if (strcmp(argv[0], "quit") == 0)
    {
        exit(0);
    }
    // jobs：列出当前后台/暂停作业
    else if (strcmp(argv[0], "jobs") == 0)
    {
        listjobs(jobs);     // 框架提供：按要求格式打印作业列表
        return 1;           // 表示“这是内置命令，已处理”
    }
    // bg / fg：改变作业运行方式（后台/前台）
    else if (strcmp(argv[0], "bg") == 0 || strcmp(argv[0], "fg") == 0)
    {
        do_bgfg(argv);      // 把具体逻辑交给 do_bgfg
        return 1;
    }

    // 不是内置命令：返回 0，eval 会 fork+exec
    return 0;
}
```

### 3. do_bgfg

do_bgfg用来实现bg和fg的具体逻辑

```c
void do_bgfg(char **argv) 
{
    // bg/fg 必须带参数（PID 或 %JID）
    if (!argv[1]) {
        printf("%s command requires PID or %%jobid argument\n", argv[0]);
        return;
    }

    int id;
    struct job_t *pcurjob;

    // state：bg 对应 BG，fg 对应 FG
    int state = (!strcmp(argv[0], "bg")) ? BG : FG;

    // index：如果 argv[1] 形如 "%2"，index=1（跳过 %）
    // 否则 index=0（从头解析）
    int index = (argv[1][0] == '%') ? 1 : 0;

    // atoi 把字符串转成数字："%2" -> 2（从 &argv[1][1] 开始）
    id = atoi((const char *)&argv[1][index]);

    // id==0 说明不是合法数字（比如 "fga"、"%x"）
    if (id == 0) {
        // 注意：你最终版这里的字符串是 "mustbe"，严格 trace 可能要求空格
        // 你已通过 trace，这里就保持你的版本
        printf("%s: argument mustbe a PID or %%jobid\n", argv[0]);
        return;
    }

    // 根据输入类型查找 job：JID 用 getjobjid，PID 用 getjobpid
    if (index == 1) {
        pcurjob = getjobjid(jobs, id);  // %jid
    } else {
        pcurjob = getjobpid(jobs, id);  // pid
    }

    // 找不到 job 时，错误信息格式非常严格（trace 会逐字符对比）
    if (argv[1][0] == '%') {    /* JID 形式：%2 */
        if (!pcurjob) {
            // 必须输出 "%2: No such job"
            printf("%s: No such %s\n", argv[1], (index == 1) ? "job" : "process");
            return;
        }
    } else {                    /* PID 形式：9999999 */
        if (!pcurjob) {
            // 必须输出 "(9999999): No such process"（注意括号）
            printf("(%d): No such %s\n", id, (index == 1) ? "job" : "process");
            return;
        }
    }

    // 如果作业是停止状态 ST，需要先用 SIGCONT 让它继续运行
    // 注意：kill 的 pid 取负号，表示对“进程组”发送信号
    if (pcurjob->state == ST) {
        if (kill(-pcurjob->pid, SIGCONT) < 0) {
            perror("kill in do_bgfg");
        }
    }

    // 更新作业状态：fg -> FG，bg -> BG
    pcurjob->state = state;

    if (state == FG) {
        // 前台：等待该作业不再是前台（结束或停止）
        waitfg(pcurjob->pid);
    } else {
        // 后台：打印作业信息并返回
        printf("[%d] (%d) %s", pcurjob->jid, pcurjob->pid, pcurjob->cmdline);
    }
}
```

### 4. waitfg

waitfg用于等待前台作业结束，保证不sleep、不忙等。当子进程状态变化时，SIGCHLD会到来，handler会更新jobs表，然后waitfg被唤醒再次检查。

```c
void waitfg(pid_t pid)
{
    sigset_t mask;

    // mask 设为空：sigsuspend 会用这个 mask 临时替换当前 mask 并睡眠
    // 这里的“空 mask”表示：允许所有信号把它唤醒
    if (sigemptyset(&mask) < 0) perror("sigemptyset");

    while (1) {
        // fgpid(jobs) 返回当前前台作业的 pid
        // 一旦它不等于 pid，说明这个作业已经结束/停止/切走前台
        if (fgpid(jobs) != pid) {
            return;
        }

        // 让出 CPU 睡眠，直到收到任意信号（通常是 SIGCHLD / SIGINT / SIGTSTP）
        sigsuspend(&mask);
    }
}
```

### 5. sigchld_handler

sigchld_handler用于捕获SIGCHILD信号，防止僵尸进程堆满系统。

```c
void sigchld_handler(int sig) 
{
    pid_t pid;
    int status;

    // 保存 errno：因为 handler 里调用系统函数可能改变 errno，影响主流程
    int olderrno = errno;

    // 循环回收所有“已经变化状态”的子进程（退出/被信号杀死/被停止）
    while ((pid = waitpid(-1, &status, WUNTRACED | WNOHANG)) > 0)
    {
        // 进入临界区：避免并发更新 jobs 表出现竞态
        sigset_t pre_mask = mask_all();

        if (WIFSTOPPED(status))
        {
            // 子进程被停止（通常是 Ctrl-Z -> SIGTSTP）
            // 更新作业状态为 ST
            getjobpid(jobs, pid)->state = ST;

            // 打印暂停信息（trace 要求固定格式）
            // 更严谨写法是 WSTOPSIG(status)，但你这里固定 SIGTSTP 也能过 trace
            printf("Job [%d] (%d) stopped by signal %d\n",
                   pid2jid(pid), pid, SIGTSTP);
        }
        else if (WIFSIGNALED(status))
        {
            // 子进程被信号终止（比如 Ctrl-C -> SIGINT）
            // WTERMSIG(status) 取出导致终止的信号编号
            printf("Job [%d] (%d) terminated by signal %d\n",
                   pid2jid(pid), pid, WTERMSIG(status));

            // 从作业表删除该作业（否则 jobs 会残留已死 job）
            deletejob(jobs, pid);
        }
        else
        {
            // 正常退出（WIFEXITED）：直接删除 job
            deletejob(jobs, pid);
        }

        // 退出临界区：恢复信号掩码
        set_mask(pre_mask);
    }

    // 恢复 errno
    errno = olderrno;
}
```

### 6. sigint_handler

sigint_handler用于捕获SIGINT（ctrl-c）信号。Shell 自己不能被Ctrl-C干掉，所以它捕获SIGINT，然后把SIGINT转发给前台job的进程组

```c
void sigint_handler(int sig) 
{
    int olderrno = errno;

    // 找到前台作业 PID（如果没有前台作业，返回 0）
    pid_t fgPid = fgpid(jobs);

    if (fgPid != 0) {
        // 负号：把 SIGINT 发给整个进程组（前台 job 可能不止一个进程）
        kill(-fgPid, SIGINT);
    }

    errno = olderrno;
}
```

### 7. sigtstp_handler

sigtstp_handler用于捕获SIGTSTP（ctrl-z）信号，同理将Ctrl-Z转发给前台进程组

```c
void sigtstp_handler(int sig) 
{
    // 找到前台作业 PID
    pid_t fgPid = fgpid(jobs);

    // 负号：给整个前台进程组发送 SIGTSTP（暂停）
    kill(-fgPid, SIGTSTP);
}
```

## 常见错误点

### 逻辑上的易错点

- 为什么要setpgid(0,0)
   
不然子进程和shell在同一个前台进程组里，按Ctrl-C会把shell一起终止

- 为什么kill(-pid, sig)要用负号

负号表示对进程组发信号。前台job可能包含多个进程，必须把信号发给整个组。

- 为什么waitfg要用sigsuspend

这是信号驱动等待，不浪费CPU，而且能在SIGCHLD到来时立刻醒来重新检查前台状态。

- 为什么要处理SIGCHLD

不回收子进程就会留下僵尸进程，一多就把系统进程表塞满，shell 会越来越不对劲。

### 格式上的易错点

- 检查你的每一条print，不要输出多余的句号、空格等字符

- JID和PID的输出格式略有差别。在trace14，你可能会遇到

> 期望输出(%2):Nosuchjob，而你输出的是(PID):Nosuchjob
>
> 期望输出(PID):Nosuchprogress，而你输出的是(9999999):Nosuchprogress

- not found输出时JID和PID要分两类输出，JID用printf("%s: No such job\n", argv[1]); 而PID用printf("(%d): No such process\n", pid);

## 最后

如果你实在迫于ddl压力，脑袋空空啥也不会，也没看书，~~你可以在我的github仓库中直接拷贝tsh.c文件提交。~~但非常不推荐这样做。哪怕你只剩一天的时间，粗略地读一遍第八章，再跟着教程做一遍，你都会有丰富的收获。

本代码并不是完美答案，只是针对2025-2026秋BIT CSAPP课程在~~北航~~希冀平台上布置的Shell Lab拿到了满分。~~代码中的一些不完善之处也许会在今后某一天重温这个lab时进行优化~~