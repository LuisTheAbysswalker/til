---
type: how-to
tags: [react, ios, safari, css, mobile]
created: 2026-03-20
---

# iOS Safari 弹性滚动导致 textarea 光标溢出

## 问题

iOS Safari 中，`fixed` 定位的容器内有 `textarea`，当用户上下拖动页面到底部时，Safari 的弹性滚动（rubber banding）会拖动整个页面，导致 textarea 的闪烁光标（caret）溢出输入框容器，显示在容器外面。

`overflow: hidden` 无法阻止光标溢出，因为 iOS Safari 对 caret 有特殊处理。

## 解决方案

### 1. 禁止页面弹性滚动

在 `html` 和 `body` 上设置 `overscrollBehavior: none`：

```typescript
useEffect(() => {
  document.documentElement.style.overscrollBehavior = 'none';
  document.body.style.overscrollBehavior = 'none';
  document.documentElement.style.overflow = 'hidden';
  document.body.style.overflow = 'hidden';
  document.documentElement.style.height = '100%';
  document.body.style.height = '100%';
  return () => {
    document.documentElement.style.overscrollBehavior = '';
    document.body.style.overscrollBehavior = '';
    document.documentElement.style.overflow = '';
    document.body.style.overflow = '';
    document.documentElement.style.height = '';
    document.body.style.height = '';
  };
}, []);
```

### 2. 使用 `overflow-clip` 替代 `overflow-hidden`

```css
/* overflow-hidden 会创建滚动容器，光标可能溢出 */
.container { overflow: hidden; }

/* overflow-clip 硬裁剪，不创建滚动上下文 */
.container { overflow: clip; }
```

Tailwind class: `overflow-clip` 替代 `overflow-hidden`

### 3. textarea 高度重置

发送按钮和 textarea 不在同一个 focus 上下文，点击发送后 `document.activeElement` 是 button 不是 textarea。用 ref 直接操作：

```typescript
const textareaRef = useRef<HTMLTextAreaElement>(null);

const handleSend = () => {
  // ... 发送逻辑
  setMessageInput('');
  setIsMultiline(false);
  // 用 ref 重置，不依赖 activeElement
  if (textareaRef.current) {
    textareaRef.current.style.height = 'auto';
    textareaRef.current.blur();
  }
};
```

## 关键点

| 属性 | 作用 | 注意 |
|------|------|------|
| `overscrollBehavior: none` | 禁止弹性滚动 | 需要在 html 和 body 都设置 |
| `overflow: clip` | 硬裁剪内容含光标 | 不创建滚动容器，比 `hidden` 更强 |
| `position: fixed` + `overflow: hidden` on body | 锁定页面 | 单独用可能不够 |

## 注意事项

- 这些设置是页面级别的，组件卸载时需要恢复
- iframe 内部的滚动不受影响（有自己的滚动上下文）
- `overscrollBehavior` 在 iOS Safari 15+ 支持
