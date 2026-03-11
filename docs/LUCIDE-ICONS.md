# Lucide Icons Installation & Usage Guide

This project uses [Lucide](https://lucide.dev/) — an open-source icon library with 1000+ consistently designed SVG icons.

## Installation

✅ **Already installed** in both plugin and shell:

```json
// Main plugin (package.json - devDependencies)
"lucide": "^0.575.0"

// Shell (shell/package.json - dependencies)  
"lucide": "^0.575.0"
```

Installed via:
```bash
# Main plugin
npm install lucide --save-dev

# Shell
cd shell && npm install lucide
```

## Current Implementation

The project currently uses **manually copied SVG path data** in [`shell/src/renderer/shims/icons.ts`](../shell/src/renderer/shims/icons.ts):

```typescript
const ICON_PATHS: Record<string, string> = {
  'message-square': '<path d="M21 15a2 2 0 0 1-2 2H7l-4 4V5a2 2 0 0 1 2-2h14a2 2 0 0 1 2 2z"/>',
  'settings': '<path d="M12.22 2h-.44a2 2 0 0 0-2 2v.18..."/>',
  // ... more icons
};
```

## How to Use Lucide Package

### Option 1: Import SVG String (Recommended for Dynamic Icons)

```typescript
import { icons } from 'lucide';

// Get SVG markup for any icon
const sendIconSvg = icons['send'];
// Returns: '<svg>...</svg>' (full SVG element)

// Use with existing setIcon pattern
function setIcon(element: HTMLElement, iconName: string) {
  const svg = icons[iconName];
  if (svg) {
    element.innerHTML = svg;
  }
}
```

### Option 2: Import Individual Icons

```typescript
import { Send } from 'lucide';

// Each icon is exported as PascalCase (e.g., MessageSquare, AlertTriangle)
console.log(Send); // Full SVG markup
```

### Option 3: Tree-Shakable Imports (Best for Bundle Size)

```typescript
import { createIcons, Send, MessageSquare, Settings } from 'lucide';

// Initialize icons in the DOM
createIcons({
  icons: { Send, MessageSquare, Settings }
});

// Then use with data attributes in HTML:
// <i data-lucide="send"></i>
```

### Option 4: Dynamic Creation

```typescript
import { createElement } from 'lucide';

// Create an icon element programmatically
const sendIcon = createElement(Send);
document.body.appendChild(sendIcon);

// With custom attributes
const customIcon = createElement(MessageSquare, {
  size: 24,
  color: 'blue',
  strokeWidth: 2
});
```

## Migrating Current Implementation

### Before (Manual Path Data)

```typescript
const ICON_PATHS: Record<string, string> = {
  'send': '<path d="m22 2-7 20-4-9-9-4Z"/><path d="M22 2 11 13"/>',
  'message-square': '<path d="M21 15a2 2 0 0 1-2 2H7l-4 4V5..."/>',
};

export function setIcon(el: HTMLElement, name: string) {
  const paths = ICON_PATHS[name];
  if (!paths) return;
  
  el.innerHTML = `<svg xmlns="http://www.w3.org/2000/svg" 
                       width="24" height="24" 
                       viewBox="0 0 24 24" 
                       fill="none" 
                       stroke="currentColor" 
                       stroke-width="2" 
                       stroke-linecap="round" 
                       stroke-linejoin="round">
                    ${paths}
                  </svg>`;
}
```

### After (Using Lucide Package)

```typescript
import { icons } from 'lucide';

export function setIcon(el: HTMLElement, name: string) {
  const svg = icons[name];
  if (!svg) {
    console.debug(`Icon not found: ${name}`);
    return;
  }
  
  el.innerHTML = svg;
  
  // Optional: customize the SVG element
  const svgEl = el.querySelector('svg');
  if (svgEl) {
    svgEl.setAttribute('width', '24');
    svgEl.setAttribute('height', '24');
    svgEl.setAttribute('stroke-width', '2');
  }
}
```

## Benefits of Using the Package

### 1. **Access to Full Library**
- 1000+ icons available immediately
- No need to manually copy/maintain path data
- Always stay up-to-date with latest icon designs

### 2. **Type Safety**
```typescript
import { icons } from 'lucide';

// TypeScript knows all available icon names
type IconName = keyof typeof icons;

function setIcon(el: HTMLElement, name: IconName) {
  // Type-safe icon usage
}
```

### 3. **Tree-Shaking Support**
Only bundle the icons you actually use:

```typescript
// Only these icons will be included in the final bundle
import { Send, MessageSquare } from 'lucide';
```

### 4. **Consistent Updates**
```bash
# Update to latest Lucide release
npm update lucide
```

## Common Use Cases

### Adding a New Icon

**Old way (manual):**
1. Go to lucide.dev
2. Find icon → copy SVG path data
3. Manually add to `ICON_PATHS` object

**New way (with package):**
```typescript
import { icons } from 'lucide';

// All 1000+ icons are already available
const myIcon = icons['new-icon-name'];
```

### Ribbon Button with Icon

```typescript
import { MessageSquare } from 'lucide';

const button = document.createElement('button');
button.innerHTML = MessageSquare;
button.classList.add('ribbon-button');
```

### Settings Tab Icon

```typescript
import { Settings } from 'lucide';

this.addSettingTab(new MySettingTab(this.app, this));
// Icon is automatically available via setIcon('settings')
```

### Custom Icon in Modal

```typescript
import { AlertTriangle } from 'lucide';

export class WarningModal extends Modal {
  onOpen() {
    const { contentEl } = this;
    const iconContainer = contentEl.createDiv('warning-icon');
    iconContainer.innerHTML = AlertTriangle;
  }
}
```

## Icon Name Reference

All icons use **kebab-case** names. Common examples:

| Icon | Name | Import |
|------|------|--------|
| 📤 | `send` | `Send` |
| 💬 | `message-square` | `MessageSquare` |
| ⚙️ | `settings` | `Settings` |
| ➕ | `plus` | `Plus` |
| ❌ | `x` | `X` |
| 📎 | `paperclip` | `Paperclip` |
| ✏️ | `pencil` | `Pencil` |
| 🔍 | `search` | `Search` |
| 🗑️ | `trash` | `Trash` |
| 🎤 | `mic` | `Mic` |
| ⚠️ | `alert-triangle` | `AlertTriangle` |
| ℹ️ | `info` | `Info` |
| 🔄 | `refresh-cw` | `RefreshCw` |

**Full icon browser:** https://lucide.dev/icons/

## TypeScript Configuration

Already configured in `tsconfig.json`:

```json
{
  "compilerOptions": {
    "moduleResolution": "node",
    "esModuleInterop": true
  }
}
```

## esbuild Configuration

Icons are automatically bundled by esbuild. No additional configuration needed.

```javascript
// esbuild.config.mjs
external: [],  // lucide is bundled, not external
```

## API Reference

### Core Functions

```typescript
import { 
  icons,           // Record<string, string> - all icon SVG strings
  createElement,   // Create icon DOM element
  createIcons,     // Initialize data-lucide attributes
  Icon,            // Icon component type
  IconNode         // Icon data structure
} from 'lucide';
```

### Icon Properties

```typescript
interface IconProps {
  size?: number | string;
  color?: string;
  strokeWidth?: number | string;
  absoluteStrokeWidth?: boolean;
  fill?: string;
  class?: string;
}
```

### Example: Custom Icon Helper

```typescript
import { createElement } from 'lucide';
import * as icons from 'lucide';

export function createIcon(
  name: string, 
  options?: { size?: number; color?: string; strokeWidth?: number }
): SVGElement {
  const IconComponent = icons[name];
  if (!IconComponent) {
    throw new Error(`Icon "${name}" not found`);
  }
  
  return createElement(IconComponent, {
    size: options?.size ?? 24,
    color: options?.color ?? 'currentColor',
    strokeWidth: options?.strokeWidth ?? 2
  });
}

// Usage
const icon = createIcon('send', { size: 32, color: 'blue' });
document.body.appendChild(icon);
```

## Resources

- **Official Website**: https://lucide.dev/
- **Icon Browser**: https://lucide.dev/icons/
- **Installation Guide**: https://lucide.dev/guide/installation
- **Package Documentation**: https://lucide.dev/guide/packages/lucide
- **GitHub Repository**: https://github.com/lucide-icons/lucide
- **NPM Package**: https://www.npmjs.com/package/lucide

## Version Information

- **Current Version**: 0.575.0
- **Update Command**: `npm update lucide` (run in both root and shell/)
- **Changelog**: https://github.com/lucide-icons/lucide/releases

## Support

For issues with Lucide icons:
- Report bugs: https://github.com/lucide-icons/lucide/issues
- Request icons: https://github.com/lucide-icons/lucide/issues/new?template=icon_request.md

---

**Last Updated**: February 27, 2026  
**Lucide Version**: 0.575.0
