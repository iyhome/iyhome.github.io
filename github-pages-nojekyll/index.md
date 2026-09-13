# GitHub Pages 的 Jekyll 幽灵：一次 Liquid 报错的根治


本站是 Hugo 构建、推送到 `iyhome.github.io` 仓库再由 GitHub Pages 发布的。某天开始，Pages 构建反复失败，日志里蹦出一句 `Liquid syntax error: Unknown tag 'F'`。追下去发现，这不是内容写错，而是 **GitHub Pages 默认还在用 Jekyll 处理一个 Hugo 已经构建好的站点**。本文记录从应急到根治的完整过程。

## 1. 现象

Pages 的内置构建（`pages build and deployment`）失败：

```
Liquid Exception: Liquid syntax error (line 21): Unknown tag 'F' in archive/oh-my-zsh-agnoster/index.md
... jekyll-3.10.0 ... github-pages-232 ...
```

线上页面一度打不开。奇怪的是：这个文件早就存在，为什么突然炸？

## 2. 根因：Jekyll 把 `.md` 当模板渲染

本站发布到 `iyhome.github.io` 的产物里，既有一堆 `.html`，也有 Hugo 顺手输出的同名 `.md`。而 GitHub Pages **在没有 `.nojekyll` 时，会用 Jekyll 处理仓库**——它会拿 `.md` 文件跑 Liquid 模板。

于是问题文件里那段 zsh 提示符示例：

```
"%(!.%{%F{yellow}%}.)%n@%m"
```

其中的 `%{` 加 `%F` 拼出了字面量 `{%`，被 Liquid 当成「标签开始」，读到 `%}` 结束，得到未知标签 `F` → 整个构建中断。

关键认知：**这不是内容 bug，是构建器选错了**。只要 Jekyll 还开着，任何新内容里出现 `{%` / `{{` 都可能再次炸。

## 3. 应急：用 raw 包住

最快的止血是在 `.md` 里用 `{% raw %}...{% endraw %}` 把危险片段包起来，让 Liquid 原样输出：

```markdown
{% raw %}
"%(!.%{%F{yellow}%}.)%n@%m"
{% endraw %}
```

（注意 `{% raw %}` 要独占一行、放在代码围栏外侧。）这能让构建先过，但只是**绕开**报错点，没有关掉 Jekyll。

## 4. 根治：一个空的 `.nojekyll`

真正的解法是在部署仓库**根目录放一个空的 `.nojekyll` 文件**，告诉 GitHub Pages：别跑 Jekyll，直接按静态文件发布。

```bash
touch .nojekyll
git add .nojekyll
git commit -m "chore(pages): add .nojekyll to skip Jekyll"
git push origin main
```

之后 Pages 的构建步骤只剩 `Checkout` + `Upload artifact`，**不再有 `Build with Jekyll`**，`.md` 也就不再被 Liquid 解析。

## 5. 顺手把源头也修了

治标之外，还要保证「自动发布」这条链路不会再丢 `.nojekyll`。本站源仓库的 workflow 用 `peaceiris/actions-gh-pages`：

```yaml
- uses: peaceiris/actions-gh-pages@v3
  with:
    personal_token: ${{ secrets.XXX }}
    external_repository: iyhome/iyhome.github.io
    publish_dir: ./public
    publish_branch: main
    enable_jekyll: false   # 显式声明：生成 .nojekyll
```

两个额外的坑：

- **`enable_jekyll: false` 是默认值**，它会自动写入 `.nojekyll`；但一旦有人用「手动 rsync 兜底发布」，就容易把 `.nojekyll` 弄丢——所以显式写出来，并尽量走 workflow。
- 旧 workflow 写的是 `runs-on: ubuntu-20.04`，而该 runner 已退役，任务会一直排队/取消。换成 `ubuntu-latest` 后构建立刻恢复。

## 6. 小结

| 阶段 | 动作 |
|---|---|
| 现象 | Pages 构建 `Unknown tag 'F'` |
| 根因 | Jekyll 解析 Hugo 产物里的 `.md`，`{%` 被当 Liquid |
| 应急 | `{% raw %}` 包裹危险片段 |
| 根治 | 部署仓库根放空 `.nojekyll` |
| 防复发 | workflow 显式 `enable_jekyll: false` + 更新 runner |

一句话：**静态站交给 GitHub Pages 时，务必放 `.nojekyll`**。否则你永远不知道哪天，一段代码示例里的花括号就会把构建干掉。

