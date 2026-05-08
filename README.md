# md-image-embed

> A Claude Code skill that converts local image references in Markdown files to base64 data URIs, making your MD files fully self-contained.

## The Problem

You have a Markdown file with images:

```markdown
![架构图](./images/architecture.png)
![结果图](E:\comet\result.png)
```

When you send this `.md` file to someone else — they see **broken images**. The images are separate files on *your* machine.

## The Solution

`md-image-embed` converts all local image paths to inline base64:

```markdown
![架构图](data:image/png;base64,iVBORw0KGgoAAAANSUhEUg...)
![结果图](data:image/png;base64,R0lGODlhAQABAIAAAAAAAP...)
```

**One file. All images included. Send it anywhere.**

## Demo

### Before — images break when shared

```
论文阅读/
├── 笔记.md              ← 发这个给别人？
├── images/
│   ├── figure1.png      ← 别人没有这些图
│   └── figure2.png
```

### After — single self-contained file

```
笔记.md                  ← 直接发这一个，图片全在里面
```

### Example images from real usage

These images are embedded directly in the demo markdown files:

| Figure | Description |
|--------|-------------|
| ![Figure 1](demo/figure1.png) | Research framework overview |
| ![Figure 2](demo/figure2.png) | Model architecture diagram |
| ![Figure 3](demo/figure3.png) | Experimental results |
| ![Figure 4](demo/figure4.png) | Overall pipeline |

## Features

- **Automatic detection** — finds all `![alt](path)` image references
- **Smart skipping** — ignores already-embedded base64 and remote URLs (http/https)
- **URL decoding** — handles paths with `%20` and other encoded characters
- **Relative path support** — resolves paths relative to the MD file location
- **Wide format support** — PNG, JPG, GIF, SVG, WebP, BMP, TIFF

## Usage

### As a Claude Code Skill

This is a [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skill. Install it and just ask:

```
把这个 md 的图片内嵌进去，我只发一个文件
```

Or use the slash command:

```
/md-image-embed
```

### Manual Installation

Copy the `SKILL.md` file to your Claude Code skills directory:

```bash
# Project-level
mkdir -p .claude/skills/md-image-embed
cp SKILL.md .claude/skills/md-image-embed/

# Or user-level
mkdir -p ~/.claude/skills/md-image-embed
cp SKILL.md ~/.claude/skills/md-image-embed/
```

### Standalone Usage

The core logic is a Python script. You can run it directly:

```python
import base64, re, os
from urllib.parse import unquote

def embed_images(md_path):
    md_dir = os.path.dirname(os.path.abspath(md_path))
    with open(md_path, 'r', encoding='utf-8') as f:
        content = f.read()

    def replace(match):
        alt, path = match.group(1), unquote(match.group(2).strip())
        if path.startswith('data:') or path.startswith(('http://', 'https://')):
            return match.group(0)
        abs_path = os.path.join(md_dir, path) if not os.path.isabs(path) else path
        if not os.path.isfile(abs_path):
            return match.group(0)
        with open(abs_path, 'rb') as f:
            b64 = base64.b64encode(f.read()).decode()
        ext = os.path.splitext(abs_path)[1].lower()
        mime = {'.png':'image/png','.jpg':'image/jpeg','.jpeg':'image/jpeg',
                '.gif':'image/gif','.svg':'image/svg+xml','.webp':'image/webp'}.get(ext,'image/png')
        return f'![{alt}](data:{mime};base64,{b64})'

    result = re.sub(r'!\[([^\]]*)\]\(([^)]+)\)', replace, content)
    with open(md_path, 'w', encoding='utf-8') as f:
        f.write(result)

# Usage
embed_images('your-notes.md')
```

## When to Use

| Scenario | Use it? |
|----------|---------|
| Sending a single MD file to someone | Yes |
| MD with local/absolute image paths | Yes |
| Images already hosted on the web | No (skipped automatically) |
| Images already base64 embedded | No (skipped automatically) |
| Large images (>5MB each) | Consider resizing first |

## File Size Note

Base64 encoding increases data size by ~33%. A 1MB image becomes ~1.3MB in the MD file. For files with many large images, consider:

- Resizing images before embedding
- Using image compression tools
- The tradeoff: slightly larger file vs. zero broken images

## License

[MIT](LICENSE)
