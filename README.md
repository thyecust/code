# README.md

运行：`cd src_full && bun unresolved-0005.js`

## 开放与民主的Agent

LLM-based Agent是一类非常重要的软件，所有的设备上都在运行它，它可以运行任何程序。

Anthropic 编写了如此优秀的代码，今天我们的生活和工作都离不开它。

## 如何阅读这份代码

这是朴素的JavaScript代码，并无特殊的语法或特性。

可以参考 https://deepwiki.com/thyecust/code .

也可参考 https://code.claude.com/docs/en/whats-new 和 [Whats-new-impl.md](./WHATS-NEW-IMPL.md) .

## 如何在二进制中提取文件

```javascript
// USE: BUN_OPTIONS="--preload ./preload.js" claude

import fs from 'fs';
import path from 'path';

const OUT = 'out';
const ROOTS = ['/$bunfs/root'];

const root = ROOTS.find((r) => {
    try {
        fs.readdirSync(r);
        return true;
    } catch {
        return false;
    }
})

if (!root) {
    throw new Error(`Could not find a valid root directory in ${ROOTS}`);
}

const manifest = [];
let files = 0, dirs = 0, bytes = 0, fail = 0;

function walk(vfsPath, rel) {
  let names;
  try {
    names = fs.readdirSync(vfsPath);
  } catch (e) {
    fail++;
    manifest.push({ path: rel, error: `readdir: ${e.message}` });
    return;
  }
  for (const name of names) {
    const childVfs = vfsPath + "/" + name;
    const childRel = rel ? rel + "/" + name : name;
    let st;
    try {
      st = fs.statSync(childVfs);
    } catch (e) {
      fail++;
      manifest.push({ path: childRel, error: `stat: ${e.message}` });
      continue;
    }
    if (st.isDirectory()) {
      dirs++;
      manifest.push({ path: childRel, kind: "dir" });
      fs.mkdirSync(path.join(OUT, childRel), { recursive: true });
      walk(childVfs, childRel);
      continue;
    }
    try {
      const buf = fs.readFileSync(childVfs);
      const dst = path.join(OUT, childRel);
      fs.mkdirSync(path.dirname(dst), { recursive: true });
      fs.writeFileSync(dst, buf);
      files++;
      bytes += buf.length;
      manifest.push({ path: childRel, size: buf.length });
    } catch (e) {
      fail++;
      manifest.push({ path: childRel, error: `read: ${e.message}` });
    }
  }
}

walk(root, "");

fs.writeFileSync(path.join(OUT, "manifest.json"), JSON.stringify(manifest, null, 2));

```
