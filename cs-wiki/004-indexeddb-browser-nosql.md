# IndexedDB：浏览器自带的 NoSQL 数据库

> cs-wiki·第 004 篇。一篇笔记，三句话：IndexedDB 是 Chrome 浏览器内置的 NoSQL 数据库，观察它主要用两种方法：浏览器开发者工具（推荐）和通过 JavaScript 代码。

IndexedDB 是 chrome 浏览器内置的 NoSQL 数据库。观察浏览器自带的 IndexedDB 主要有两种最常用且有效的方法：使用浏览器开发者工具（推荐）和通过 JavaScript 代码。

## 🛠️ 方法一：使用浏览器开发者工具（最直观）

这是最常用、最可视化的方法，适用于绝大多数现代浏览器，如 Chrome、Edge 和 Firefox。

1. **打开开发者工具**
   在目标网页上，按下快捷键 `F12` 或 `Ctrl+Shift+I` (Windows/Linux) / `Cmd+Option+I` (macOS)，或者右键点击页面选择"检查"。
2. **进入应用面板**
   在开发者工具的顶部菜单栏中，找到并点击 **"Application"**（应用）面板。
3. **查看 IndexedDB**
   在左侧的边栏中，找到 **"Storage"**（存储）部分，展开后点击 **"IndexedDB"**。
   * 你会看到当前网站使用的所有 IndexedDB 数据库列表。
   * 点击数据库名称可以查看其版本号和来源。
   * 继续展开，点击具体的 **Object Store**（对象存储），右侧面板就会显示其中存储的所有键值对数据。

**重要提示：**
IndexedDB 的数据在开发者工具中**不会实时更新**。如果你在网页上进行了操作导致数据变化，需要手动点击工具栏中的 **"Refresh"**（刷新）按钮来更新视图。

## 💻 方法二：使用 JavaScript 代码（更灵活）

如果你需要通过编程方式进行检测或调试，可以在开发者工具的 **"Console"**（控制台）面板中运行 JavaScript 代码。

* **列出所有数据库**
  使用 `indexedDB.databases()` 方法可以获取当前源下所有 IndexedDB 数据库的列表。

```javascript
window.indexedDB.databases().then(dbs => {
  console.log("当前源下的数据库:", dbs);
}).catch(err => {
  console.log("此浏览器不支持 indexedDB.databases() 方法或未使用IndexedDB");
});
```

* **检查特定数据库是否存在**
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
