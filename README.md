# 🏸 ShuttleScope Docs

ShuttleScope 羽毛球数据分析平台 — 文档与使用说明。

- [📖 用户手册](USAGE.md) — 功能讲解与操作指南
- [📊 参数计算方法](PARAMETERS.md) — 五维/七维分析算法详细说明
- [📋 更新日志](CHANGELOG.md) — 版本历史
- [🧪 可视化原型](visualization-prototype.html) — 数据墙、赛事点阵、战术排名、球员画像、H2H 与逐分节奏图

在线体验：[shuttlescope.org](https://shuttlescope.org)

## 可视化原型预览

`visualization-prototype.html` 是一个可独立打开的单页原型，沿用 ShuttleScope 的深色数据工具风格，并直接读取线上公开数据：

- `https://shuttlescope.org/data/matches.json`
- `https://shuttlescope.org/data/players.json`

本地预览：

```bash
python -m http.server 8787
```

然后访问 `http://127.0.0.1:8787/visualization-prototype.html`。
