# how_to_localization_in_inZOI_ui_mod
如何汉化inZOI中的UI类模组
# inZOI LifeController 模组汉化与字体修复踩坑记录

本文档记录了在汉化 [inZOI Life Controller](https://legacy.curseforge.com/inzoi/overlay-ui/lifecontroller-854e2a05) 模组时遇到的各种技术壁垒，以及最终的解决思路。希望能为其他想要汉化 inZOI 模组的开发者提供参考。

## 第一关：绕过文件加密与防篡改机制

**现象：**
当我们尝试用文本编辑器打开模组的核心 UI 文件（`index.html` 和 `app.js`）时，发现里面全是乱码，只有文件头包含 `IZUI` 字样。并且一旦强行修改文件，模组在游戏中就会自动被禁用。

**破解加密 (AES-256-ECB)：**
经过逆向分析，我们发现 inZOI 的 UI 模组文件使用了 **AES-256-ECB** 加密算法，其加解密的密钥是硬编码在引擎中的字符串：`InZOIUIModEncppppgo0Key202&Stu5i`。
文件结构如下：
- `IZUI` 魔数 (4 bytes)
- 版本号 (2 bytes: `\x01\x00`)
- 明文长度 (4 bytes, Little Endian)
- AES-256-ECB 加密后的密文数据 (带 16 字节 PKCS7 padding)

通过编写 Python 解密脚本，我们将文件还原为明文，从而提取了所有文本进行汉化，最后再重新加密写回。

**破解防作弊校验 (FileChecksums)：**
解密并修改文件后，必须更新校验码。
在 `mod_manifest.json` 中有一个 `FileChecksums` 字段。经过测试，游戏引擎计算 MD5 的逻辑非常“奇特”：**它校验的并不是文件的真实内容，而是“文件名+文件大小”拼接成的字符串！**
算法公式为：`MD5("filename|filesize")` （例如 `MD5("index.html|181210")`）。
掌握了这个规律后，我们编写了自动打包脚本，在每次修改后自动计算伪造的 MD5 写入 Manifest，成功骗过了引擎。

## 第二关：解决“全屏方框”与底层字体的博弈

成功加载汉化文件后，迎来了最大的噩梦：**所有中文字符全部变成了 `▯▯▯▯` 方框。**
inZOI 使用了 Cohtml 作为其 UI 渲染引擎。这是一个极其封闭的沙盒环境，我们在这上面踩了无数的坑：

1. **无法读取系统字体：** 尝试在 CSS 中硬编码 `font-family: "Microsoft YaHei"`。无效，Cohtml 被限制无法读取 Windows 系统本地字体库。
2. **硬编码字体导致回退机制损坏：** 最初我们错误地沿用了原模组的 `font-family: Arial`。这导致了致命问题：由于 Arial 不包含中文字符，Cohtml 没有正确回退到游戏引擎自带的全局中文 UI 字体，而是强制降级到了不支持中文的等宽方块字。
3. **加载外部字体文件失败：** 我们将 `simhei.ttf` 放入模组目录并用 `@font-face` 加载。结果按 F10 直接导致界面崩溃无法呼出。
   - *原因A (内存溢出)*：原始中文字体体积接近 10MB，超长的二进制数据或者庞大的字体文件直接让轻量级的 UI 解析器崩溃了。
   - *原因B (白名单机制)*：**这是最隐蔽的暗坑！** 虚幻引擎的虚拟文件系统 (`coui://`) 有极严格的白名单机制。如果你只把字体放到文件夹里，引擎根本不会加载它。**任何想要在 UI 中加载的本地文件，都必须手动注册到 `mod_manifest.json` 的 `NonAssets` 数组中！**

**最终完美解决方案：**
1. **字体极限压缩 (Subsetting)**：我们使用 Python 的 `fontTools`，将 10MB 的黑体扫描提取，只保留了模组中实际用到的 1000 多个汉字。最终将字体打包成了体积仅 **144KB 的 WOFF 格式** (`simhei.woff`)。
2. **注册资产白名单**：在 `mod_manifest.json` 中添加 `/ui/main_menu/simhei.woff` 到 `NonAssets` 数组，赋予其加载权限。
3. **精准 CSS 覆盖**：在 `index.html` 中通过 `@font-face` 引入字体。**特别注意**：网页中的 `<button>`、`<input>` 等控件默认是不继承全局字体的。必须在 CSS 中显式声明 `button, input, select, textarea { font-family: 'SimHei', sans-serif; }`，才能彻底消灭所有按钮上的方框。

## 第三关：UI 面板的等比例无损放大

汉化成功后，因为中文字符的视觉密度较大，导致原模组的界面在 2K/4K 屏幕下显得极为吃力。
由于界面的 HTML 采用了复杂的 Flex 相对布局，逐一修改内部各个元素的像素宽高很容易导致整体排版错位。
**解决方案**：
我们直接在 CSS 的最外层容器 `#life-controller` 中加入：
```css
transform: scale(1.25);
transform-origin: center;
```
得益于 Cohtml 矢量的渲染特性，这样不仅将整个 UI 完美放大了 125%，且不会产生任何模糊或错位，一行代码解决了适配问题。

---

**总结：**
这次汉化修复之旅涵盖了：AES 解密、特殊的 MD5 签名伪造、Web 字体子集化 (Subsetting)、虚幻引擎 UI 沙盒资产白名单穿透、以及 Web 前端 CSS 深度排错。希望能帮到大家！
