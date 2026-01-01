# GoMusic → 椒盐音乐（Salt Player）歌单转换器

将 **GoMusic**（https://music.unmeta.cn/）导出的纯文本歌单（每行形如 `曲名 - 作曲家/歌手`）自动匹配到你电脑上的本地曲库音频文件，并生成 **椒盐音乐（Salt Player）** 可导入的歌单文本（每行是手机上的绝对路径）：

```text
/storage/emulated/0/Music/曲库/<音乐文件全名含扩展名>
```

核心能力：

- 扫描本地曲库（递归）
- 使用模糊匹配（RapidFuzz）尽可能将每一行文本映射到 **唯一** 的音频文件
- 输出椒盐音乐可识别的歌单（关键点：**LF 换行**，并避免行尾多余字符导致导入失败）
- 生成未匹配/歧义项报告，便于人工修正

---

## 背景与目标

同样是“纯文本歌单”：

- **GoMusic**：每行只是 `曲名 - 歌手/作曲家`
- **椒盐音乐（Salt Player）**：每行必须是指向手机目录中的**具体文件路径**，包含目录位置和音乐文件全名（含扩展名）

因此需要一个转换步骤，把 GoMusic 的文本条目映射到真实文件名，并生成椒盐音乐可导入的路径歌单。

---

## 完整流程（以 QQ 音乐为例，可类推其他平台）

### Step 1：从平台下载音乐文件（电脑端）
在 QQ 音乐中选中歌单（示例：红心之王）并下载全部音乐。常见的下载命名格式为：

- `歌名 - 歌手名.<扩展名>`

将所有下载得到的音频文件放入电脑某个曲库目录，例如：

- `D:\Workshop\音乐工作目录\曲库\`

> 提示：不同歌单的音乐文件可以混放在同一曲库目录中，这不影响椒盐音乐的扫描；我们真正要解决的是“歌单条目 → 文件名”的映射。

---

### Step 2：用 GoMusic 导出歌单纯文本
打开 GoMusic（项目仓库：https://github.com/Bistutu/GoMusic）：https://music.unmeta.cn/ ，输入歌单链接，得到纯文本，例如：

```text
Soviet March - 群星
Night Crusing(夜间巡航) - 牛尾憲輔
Gilded Runner 流金疾驰 - HOYO-MiX
MASURAO - 川井宪次
神々が恋した幻想郷 - 上海爱莉丝幻乐团
```

---

### Step 3：保存 GoMusic 歌单文件（电脑端）
将上述纯文本保存为 `<歌单名>.txt`，放到歌单目录，例如：

- `D:\Workshop\音乐工作目录\歌单\红心之王.txt`

每行一个条目。

---

### Step 4：运行脚本生成“椒盐音乐歌单”（电脑端）
椒盐音乐原生创建歌单后导出的文本形如：（参考：椒盐音乐开源仓库 https://github.com/Moriafly/SaltPlayerSource）

```text
/storage/emulated/0/Music/曲库/Akasha Pulses, the Kalpa Flame Rises (Nahida Theme) (Nahida Version) - tnbee.wav
/storage/emulated/0/Music/曲库/After the Wind - DJ OKAWARI.flac
/storage/emulated/0/Music/曲库/A Lannister Always Pays His Debts - Ramin Djawadi.flac
/storage/emulated/0/Music/曲库/00 Gundam - 川井宪次 (かわい けんじ).flac
```

本项目脚本的目的就是完成这个【填空】：

> **把 GoMusic 的 `<歌名 - 歌手>` 列表，最大概率匹配到本地曲库中的唯一文件，并输出为椒盐音乐可导入的“手机绝对路径歌单”。**

脚本输出位置（另一个【填空】）：

- `D:\Workshop\音乐工作目录\椒盐歌单_output\`

其中每个输出歌单与输入歌单同名。

---

### Step 5：把曲库同步到手机
将电脑端 `曲库/` 复制到手机目录，例如：

- `/storage/emulated/0/Music/曲库/`

推荐方式：先压缩电脑端 `曲库` 目录 → 传到手机 → 解压缩，减少传输过程中文件名被改写的概率。

---

### Step 6：把输出歌单导入椒盐音乐
将 `椒盐歌单_output/` 下生成的歌单 `.txt` 文件传到手机，通过椒盐音乐的“导入歌单/从文件导入”等入口导入即可。

---

## 安装与运行

### 1）准备目录结构

建议结构如下（可在脚本 CONFIG 里改）：

```text
D:\Workshop\音乐工作目录\
  曲库\                # 音频文件（递归扫描）
  歌单\                # GoMusic 导出的纯文本歌单（每行：曲名 - 歌手）
  convert_playlists.py # 本脚本
```

---

### 2）安装依赖

本项目依赖 **RapidFuzz**（用于模糊匹配）：

```bash
pip install rapidfuzz
```

---

### 3）运行脚本

在音乐工作目录运行：

```bash
python convert_playlists.py
```

输出目录：

- `椒盐歌单_output/`：椒盐音乐可导入歌单（每行是手机绝对路径）
- `椒盐歌单_output/_report/`：每个歌单对应一个 `*.report.json`，记录未匹配/歧义项候选
- `椒盐歌单_output/SUMMARY.json`：汇总统计

---

## 关键实现细节（为什么之前会“导入 0 首”）

椒盐音乐对“歌单文件每行内容”是按路径逐行解析的。实践中发现：

- **换行符非常关键**：原生歌单是 **LF (`\n`)** 风格  
- 若输出为 Windows 默认的 **CRLF (`\r\n`)**，部分情况下会导致每行末尾残留 `\r`，进而路径不等于真实文件路径，最终“导入 0 首”

因此本脚本写输出歌单时强制使用：

- `newline="\n"`
- 且不在末尾额外追加空行

---

## 相关项目 / 致谢

本项目的输入/输出格式与使用体验，离不开以下开源项目（在此致谢与引用）：

- 椒盐音乐（Salt Player）源码仓库：<https://github.com/Moriafly/SaltPlayerSource>  
  本项目输出的歌单为椒盐音乐可导入的“手机绝对路径歌单”。

- GoMusic 仓库：<https://github.com/Bistutu/GoMusic>  
  本项目的输入歌单文本来自 GoMusic 导出的 `<曲名 - 歌手>` 纯文本格式。

---

## 常见问题（FAQ）

### Q1：为什么有些条目匹配不到？
可能原因：

- 歌单文本与文件名差异过大（别名/翻译名/版本信息不同）
- 曲库存在同名多版本，脚本无法判定唯一

请查看 `_report/*.report.json` 中的 `not_found` 与 `ambiguous`，按候选项手动修正歌单文本或文件名后再运行一次。

---

### Q2：文件名里有全角空格（U+3000）或 NBSP（U+00A0）会怎样？
路径匹配是**逐字符**的。只要歌单里的字符与手机端文件名不完全一致，就会判定文件不存在。

建议：

- 尽量保证“电脑曲库文件名”和“手机端最终文件名”完全一致（复制/压缩解压通常较稳）
- 不要在脚本里盲目把全角空格替换成半角空格（可能适得其反）

## AI 生成说明

本项目代码与文档由 AI 辅助生成并经本地测试迭代。除 GoMusic 与 椒盐音乐（Salt Player）相关的仓库/网站/产品行为外，无其他定向参考依据。

---

## License

MIT (建议你在仓库中添加 `LICENSE` 文件；如需我也可以给你一份 MIT License 模板)。