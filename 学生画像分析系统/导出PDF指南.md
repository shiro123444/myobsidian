# Obsidian导出PDF完整指南

##  文件位置
您的4个Markdown文档已复制到：
```
E:\myobsidian\学生画像分析系统\
 design_doc.md           (设计文档)
 implementation_plan.md  (实施计划)
 code_examples.md        (代码示例)
 walkthrough.md          (项目指南)
```

##  推荐方案（3种方法）

### 方法1：Pandoc插件（最佳，支持Mermaid）

#### 安装Pandoc
1. 下载：https://pandoc.org/installing.html
2. 安装后验证：```pandoc --version```

#### 安装Obsidian插件
1. 设置 (Ctrl+,)  第三方插件  关闭安全模式
2. 浏览  搜索 **Pandoc Plugin**  安装并启用

#### 导出PDF
1. 打开任意.md文件
2. Ctrl+P  输入 ```Pandoc`  选择 **Export as PDF**

### 方法2：Better Export PDF插件（最简单）

1. 设置  第三方插件   浏览
2. 搜索 **Better Export PDF**  安装
3. 打开.md文件  右上角更多选项  Export to PDF

### 方法3：浏览器打印（最快）

1. 打开.md文件
2. Ctrl+P  选择 Microsoft Print to PDF

>  此方法不渲染Mermaid图表

##  Mermaid图表处理

### 选项A：安装mermaid-filter
```bash
npm install -g mermaid-filter
```
在Pandoc插件设置添加：```--filter mermaid-filter```

### 选项B：手动截图
1. 在Obsidian中查看渲染的Mermaid图
2. 截图并保存
3. 替换为图片链接

##  快速开始

**新手推荐**：安装 Better Export PDF 插件  一键导出
**专业级**：Pandoc + mermaid-filter  完美渲染

现在打开Obsidian，开始导出吧！
