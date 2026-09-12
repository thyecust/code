# README.md

运行：`cd src_full && bun unresolved-0005.js`

## 开放与民主的Agent

LLM-based Agent是一类非常重要的软件，所有的设备上都在运行它，它可以运行任何程序。

Anthropic 编写了如此优秀的代码，今天我们的生活和工作都离不开它。

## 如何阅读这份代码

这是朴素的JavaScript代码，并无特殊的语法或特性。

可以参考 https://deepwiki.com/thyecust/code .

## 如何在二进制中提取文件

静态提取，尝试常见的编码，如 `UTF-16LE`.

动态提取，对bun可以使用这种方法

```bash
BUN_OPTIONS="--preload .../vfs-dump-all.js" \
  .venv/lib/python3.14/site-packages/claude_agent_sdk/_bundled/claude \
  --version < /dev/null
```

```javascript
// 全量导出 bun 编译产物进程内的 /$bunfs/root/。
//
// 用法（WSL）：
//   BUN_OPTIONS="--preload /mnt/c/.../vfs-dump-all.js" <claude> --version < /dev/null
//
// 这一步之后就不需要再对二进制做字节切分了：ke = import.meta.require，
// 而 VFS 里的条目名 == 代码里 require 的路径，一一对应。

const fs = require("fs");
const path = require("path");

const ROOT = "/$bunfs/root";
const OUT = "bunfs-dump/all";

let names = [];
try {
  names = fs.readdirSync(ROOT);
} catch (e) {
  console.error(`[hook] 枚举失败: ${e.message}`);
  process.exit(0);
}

fs.mkdirSync(OUT, { recursive: true });

let ok = 0,
  fail = 0,
  bytes = 0;
const manifest = [];
for (const n of names) {
  const src = path.join(ROOT, n);
  const dst = path.join(OUT, n);
  try {
    const st = fs.statSync(src);
    if (st.isDirectory()) {
      manifest.push({ name: n, kind: "dir" });
      continue;
    }
    const buf = fs.readFileSync(src);
    fs.writeFileSync(dst, buf);
    ok++;
    bytes += buf.length;
    manifest.push({ name: n, size: buf.length });
  } catch (e) {
    fail++;
    manifest.push({ name: n, error: e.message });
  }
}

fs.writeFileSync(
  path.join(OUT, "_manifest.json"),
  JSON.stringify({ count: names.length, ok, fail, bytes, entries: manifest }, null, 1),
);
console.error(`[hook] ${names.length} 项：成功 ${ok}，失败 ${fail}，共 ${(bytes / 1048576).toFixed(1)} MiB`);
console.error(`[hook] -> ${OUT}`);
```
