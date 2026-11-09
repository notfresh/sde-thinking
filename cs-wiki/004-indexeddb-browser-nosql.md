# IndexedDB：浏览器自带的 NoSQL 数据库

> cs-wiki·第 004 篇。一篇笔记，三句话：IndexedDB 是 Chrome 等浏览器内置的 NoSQL 数据库，观察它有两种最常用方法——浏览器开发者工具（推荐）和 JavaScript 代码。

## 先把名词固定

| 名字 | 是什么 |
|---|---|
| **IndexedDB** | 浏览器内置的 NoSQL 数据库，按源（origin）隔离，存的是结构化对象而不是字符串 |
| **源（origin）** | 协议 + 域名 + 端口的组合。同一个源下的页面共享同一个 IndexedDB |
| **Object Store** | IndexedDB 里的"表"，按 key 存放记录，可以有 keyPath 和索引 |
| **开发者工具（DevTools）** | 浏览器自带的调试面板，按 F12 / Ctrl+Shift+I（macOS 是 Cmd+Option+I）或右键"检查"打开 |

一句话关系：**前端想在浏览器里存结构化数据，IndexedDB 是浏览器自带的那个 NoSQL 库**——它不像 Cookie 和 LocalStorage 只能存字符串，也不像 localStorage 是同步阻塞的。

其他前端存储的定位：

| 存储 | 类型 | 容量量级 | 同步/异步 | 适合放什么 |
|---|---|---|---|---|
| Cookie | 字符串 | 几 KB | 同步 | 会话标识、服务端要读的小状态 |
| localStorage | 字符串 | 几 MB | 同步 | 偏好设置、轻量缓存 |
| sessionStorage | 字符串 | 几 MB | 同步 | 单标签页的临时状态 |
| **IndexedDB** | **结构化对象** | **上百 MB 起** | **异步** | 离线数据、大对象、图片/文件 |

---

## 方法一：使用浏览器开发者工具（最直观）

这是最常用、最可视化的方法，适用于绝大多数现代浏览器，如 Chrome、Edge 和 Firefox。

1. **打开开发者工具**

   在目标网页上，按下快捷键 `F12` 或 `Ctrl+Shift+I`（Windows/Linux）/ `Cmd+Option+I`（macOS），或者右键点击页面选择"检查"。

2. **进入应用面板**

   在开发者工具的顶部菜单栏中，找到并点击 **"Application"（应用）** 面板。

3. **查看 IndexedDB**

   在左侧的边栏中，找到 **"Storage"（存储）** 部分，展开后点击 **"IndexedDB"**。

   - 你会看到当前网站使用的所有 IndexedDB 数据库列表
   - 点击数据库名称可以查看其版本号和来源
   - 继续展开，点击具体的 **Object Store**（对象存储），右侧面板就会显示其中存储的所有键值对数据

**重要提示：**

IndexedDB 的数据在开发者工具中**不会实时更新**。如果你在网页上进行了操作导致数据变化，需要手动点击工具栏中的 **"Refresh"（刷新）** 按钮来更新视图。

---

## 方法二：使用 JavaScript 代码（更灵活）

如果你需要通过编程方式进行检测或调试，可以在开发者工具的 **"Console"（控制台）** 面板中运行 JavaScript 代码。

### 列出所有数据库

使用 `indexedDB.databases()` 方法可以获取当前源下所有 IndexedDB 数据库的列表。

```javascript
window.indexedDB.databases().then(dbs => {
  console.log("当前源下的数据库:", dbs);
}).catch(err => {
  console.log("此浏览器不支持 indexedDB.databases() 方法或未使用IndexedDB");
});
```

### 检查特定数据库是否存在

如果你知道数据库的名称，可以尝试打开它来判断其是否存在。

```javascript
let request = indexedDB.open("MyApp"); // 将 "MyApp" 替换为你的数据库名
request.onsuccess = function(event) {
  console.log("数据库 'MyApp' 存在");
  event.target.result.close(); // 记得关闭连接
};
request.onerror = function(event) {
  console.log("数据库 'MyApp' 不存在或无法访问");
};
```

---

## 两个容易踩的坑

**坑一：`indexedDB.databases()` 在旧浏览器上不存在。**

这个方法是 2024 年 5 月才成为 Baseline 的新特性，各家支持版本是：Chrome 72 / Edge 72 / Firefox 126 / Safari 14。也就是说 **Firefox 125 及以下、Safari 13 及以下**直接没有这个方法，写检测代码会走 `catch` 分支——这时候得改用 DevTools 看，或者用 `open()` + `onupgradeneeded` 判断。

另外它只列**当前源**下的数据库；而且在 opaque origin（沙箱 iframe、`file://`、`data:` URL）里调用会抛 `SecurityError`，用户禁用了存储也会抛。

**坑二：`indexedDB.open()` 判断不了"存不存在"，反而会把它创建出来。**

`open()` 的语义是"打开，没有就建一个（版本 1）"。所以上面那段代码跑完，即使这个数据库原本不存在，库里也会多出一个空的 `MyApp`。想真正判断存在与否，用 `indexedDB.databases()` 查列表；或者在 `onupgradeneeded` 里判定——只有"首次创建"才会触发这个回调。

```javascript
const request = indexedDB.open("MyApp");
request.onupgradeneeded = function(event) {
  // 只在数据库首次被创建时触发
  console.log("这是新建数据库");
  const db = request.result;
  db.createObjectStore("notes", { keyPath: "id" });
};
request.onsuccess = function(event) {
  event.target.result.close();
};
```

---

**一句话总结**：IndexedDB 是浏览器内置的 NoSQL 库，看它优先用 DevTools 的 Application → Storage → IndexedDB（注意手动 Refresh 才刷新），要脚本化就用 `indexedDB.databases()` 列库——而 `open()` 是"打开或新建"，别拿它当存在性检查。