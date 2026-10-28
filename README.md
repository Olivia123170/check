# CSV Check 技术文档

CSV Check 是一个只读的 CSV 列名校验工具。本仓库是它的中文技术文档站点。

## 在线阅读

**https://htmlpreview.github.io/?https://github.com/Olivia123170/check/blob/main/standalone.html**

这个链接直接渲染 `standalone.html`（九张截图以 data URI 内嵌，不依赖同目录资源）。

本地打开：直接双击 `index.html`，或在仓库目录运行 `python3 -m http.server 8000`。

## 仓库结构

| 文件 | 说明 |
|---|---|
| `index.html` | 文档主文件，图片走 `assets/` 相对路径 |
| `assets/*.png` | 9 张终端截图 |
| `standalone.html` | 单文件版，图片全部内嵌，适合任何无法访问同级资源的托管环境 |
| `CSV_Check技术文档.md` | Markdown 源文件 |

## 文档内容

8 章 + 9 张实测截图：

| 章节 | 内容 |
|---|---|
| 01 功能定位 | 它做什么 / 它不做什么，PASS 的准确边界 |
| 02 环境依赖 | Python ≥ 3.8、零第三方包、文件结构 |
| 03 准备与校验 | 环境自检命令与预期结果 |
| 04 操作步骤 | 参数表、位置陷阱、A–E 五个场景 |
| 05 预期输出 | 七行输出逐行解释、退出码 0/1/2 |
| 06 故障排查 | 10 条可折叠条目，报错原文 → 原因 → 处理 |
| 07 版本差异 | 1.0.0 vs 1.1.0，`--delimiter` 与两个入口的默认值差异 |
| 08 已知限制 | 7 条限制 + 输入要求汇总表 |

## 事实来源

文档中所有命令输出与截图均来自本机实际运行，不是按源码推测，也不是手绘示意：

- 1.0.0 与 1.1.0 分别运行 4 个官方样例与 8 个自建边界样例
- 退出码由 `$?` 逐一读取确认
- 截图流程：真实命令捕获 stdout 与退出码 → 套入终端样式模板 → 无头 Chrome 渲染

边界样例覆盖：字段数不齐、表头重复、表头空值、仅表头无数据、带 BOM、非 UTF-8 字节、纯空行、表头带空格、引号内逗号、传入目录、参数位置错误。

## 更新方式

改完 `index.html` 后需要同步重建单文件版，否则两个版本会不一致：

```sh
python3 - <<'EOF'
import base64, pathlib, re
root = pathlib.Path(".")
html = (root / "index.html").read_text(encoding="utf-8")
def inline(m):
    data = base64.b64encode((root / m.group(1)).read_bytes()).decode()
    return 'src="data:image/png;base64,' + data + '"'
out, n = re.subn(r'src="(assets/[^"]+\.png)"', inline, html)
(root / "standalone.html").write_text(out, encoding="utf-8")
print(f"inlined {n} images")
EOF
```

然后 `git push`，在线阅读链接会自动反映新内容。

## 关于 GitHub Pages

本仓库的 GitHub Pages 尚未启用。开启需要仓库管理员权限，而自动化令牌无权创建 Pages 站点（`Resource not accessible by integration`），所以当前用 htmlpreview 提供在线阅读。

若要在 GitHub 网页端手动开启：Settings → Pages → Source 选 `Deploy from a branch` → Branch 选 `main`、目录 `/ (root)` → Save。开启后的地址是 https://olivia123170.github.io/check/

## 许可

文档内容供课程教学使用。
