---
name: md-image-embed
description: >-
  将 Markdown 文件中的本地图片转为 base64 内嵌，使单个 MD 文件即可独立显示所有图片。
  当用户要发送带图片的 MD 给别人、只想发一个文件、或提到"内嵌图片"、"图片打包成一个md"时使用。
  自动跳过已经是 base64 的图片和远程 URL 图片。
triggers:
  - 内嵌图片
  - base64图片
  - 只发一个md
  - 图片打包
  - 图片内嵌
  - embed images
---

# MD Image Embed

将 Markdown 中的本地图片引用转为 base64 data URI 内嵌，使 MD 文件自包含。

## 使用方式

用户提供一个或多个 MD 文件路径，skill 自动处理。

## 执行流程

### 第一步：确认目标文件

- 如果用户已指定文件路径，直接使用
- 如果用户只说了当前目录下的某个 MD，用 `find` 或 `ls` 确认文件存在
- 如果用户没指定文件，用 `AskUserQuestion` 询问

### 第二步：执行内嵌

运行以下 Python 脚本完成转换（在 Bash 中执行）：

```python
import base64, re, os, sys, mimetypes

md_path = sys.argv[1]
md_dir = os.path.dirname(os.path.abspath(md_path))

with open(md_path, 'r', encoding='utf-8') as f:
    content = f.read()

IMG_EXTS = {'.png', '.jpg', '.jpeg', '.gif', '.svg', '.webp', '.bmp', '.ico', '.tiff', '.tif'}

def guess_mime(path):
    ext = os.path.splitext(path)[1].lower()
    mime_map = {
        '.png': 'image/png', '.jpg': 'image/jpeg', '.jpeg': 'image/jpeg',
        '.gif': 'image/gif', '.svg': 'image/svg+xml', '.webp': 'image/webp',
        '.bmp': 'image/bmp', '.ico': 'image/x-icon', '.tiff': 'image/tiff',
        '.tif': 'image/tiff',
    }
    return mime_map.get(ext, 'application/octet-stream')

count = 0
total_bytes = 0

def replace_image(match):
    global count, total_bytes
    alt = match.group(1)
    path = match.group(2).strip()

    # Skip already embedded or remote
    if path.startswith('data:'):
        return match.group(0)
    if path.startswith('http://') or path.startswith('https://'):
        return match.group(0)

    # URL decode the path (e.g. %20 -> space)
    from urllib.parse import unquote
    path_decoded = unquote(path)

    # Resolve relative paths against MD file directory
    if not os.path.isabs(path_decoded):
        abs_path = os.path.join(md_dir, path_decoded)
    else:
        abs_path = path_decoded

    if not os.path.isfile(abs_path):
        print(f"  SKIP (file not found): {abs_path}")
        return match.group(0)

    ext = os.path.splitext(abs_path)[1].lower()
    if ext not in IMG_EXTS:
        print(f"  SKIP (not image): {abs_path}")
        return match.group(0)

    with open(abs_path, 'rb') as f:
        data = f.read()
    b64 = base64.b64encode(data).decode('utf-8')
    mime = guess_mime(abs_path)

    count += 1
    total_bytes += len(data)
    print(f"  Embedded: {os.path.basename(abs_path)} ({len(data):,} bytes)")

    return f'![{alt}](data:{mime};base64,{b64})'

# Match ![alt](path) patterns
pattern = r'!\[([^\]]*)\]\(([^)]+)\)'
result = re.sub(pattern, replace_image, content)

with open(md_path, 'w', encoding='utf-8') as f:
    f.write(result)

final_size = os.path.getsize(md_path)
print(f"\nDone: {count} images embedded, final file size: {final_size:,} bytes ({final_size/1024/1024:.1f} MB)")
```

在 Bash 中调用：

```bash
python3 -c '<上面的脚本>' "<md文件路径>"
```

### 第三步：报告结果

向用户报告：
- 处理了几张图片
- 跳过了几张（已是 base64 / 远程链接 / 文件不存在）
- 最终文件大小

## 注意事项

- 不修改 http/https 远程图片链接
- 不修改已经是 data: URI 的图片
- 路径中的 `%20` 等 URL 编码会自动解码
- 支持 png/jpg/jpeg/gif/svg/webp/bmp/ico/tiff 格式
- 相对路径基于 MD 文件所在目录解析
