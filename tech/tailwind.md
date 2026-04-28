# Tailwind CSS — AI Development Reference
> Attach when working on a project that uses Tailwind CSS. Covers configuration, design tokens, class organization, and patterns for maintainable AI-generated styles. If no base file (`fresh.md`, `new.md`, or `legacy.md`) is attached alongside this one, run a standalone audit: treat each section's guidelines as a checklist, assess the current state (✅ pass / ⚠️ partial / ❌ missing), and save a report as `tailwind-audit-YYYY-MM-DD.md`.

---

## 1. Configuration

```typescript
// tailwind.config.ts
import type { Config } from 'tailwindcss'

const config: Config = {
  // Tell Tailwind where to scan for class names (critical for tree-shaking)
  content: [
    './src/**/*.{ts,tsx}',
    './app/**/*.{ts,tsx}',
    // Add other paths as needed
  ],
  
  theme: {
    extend: {
      // Extend, don't replace — keeps all default Tailwind utilities
      colors: {
        brand: {
          50:  '#f0f9ff',
          500: '#0ea5e9',
          900: '#0c4a6e',
        },
        // Map to CSS variables for theming support
        background: 'hsl(var(--background))',
        foreground: 'hsl(var(--foreground))',
        primary: {
          DEFAULT: 'hsl(var(--primary))',
          foreground: 'hsl(var(--primary-foreground))',
        },
      },
      borderRadius: {
        lg: 'var(--radius)',
        md: 'calc(var(--radius) - 2px)',
        sm: 'calc(var(--radius) - 4px)',
      },
    },
  },
  
  plugins: [
    require('@tailwindcss/typography'),    // .prose class for markdown
    require('@tailwindcss/forms'),         // Consistent form styling
    require('tailwindcss-animate'),        // Animation utilities
  ],
}

export default config
```

---

## 2. CSS Variables for Theming (Dark Mode)

Use CSS variables so Tailwind classes automatically adapt to themes:

```css
/* globals.css */
@layer base {
  :root {
    --background: 0 0% 100%;      /* hsl values (no hsl() wrapper) */
    --foreground: 222.2 84% 4.9%;
    --primary: 222.2 47.4% 11.2%;
    --primary-foreground: 210 40% 98%;
    --radius: 0.5rem;
  }
  
  .dark {
    --background: 222.2 84% 4.9%;
    --foreground: 210 40% 98%;
    --primary: 210 40% 98%;
    --primary-foreground: 222.2 47.4% 11.2%;
  }
}
```

This is the shadcn/ui approach. Classes like `bg-background` and `text-foreground` automatically switch between light and dark values.

---

## 3. The `cn()` Utility (Required)

`cn()` merges Tailwind classes intelligently — handles conflicts and conditional classes:

```bash
npm install clsx tailwind-merge
```

```typescript
// src/lib/utils.ts
import { clsx, type ClassValue } from 'clsx'
import { twMerge } from 'tailwind-merge'

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs))
}
```

```typescript
// Usage
import { cn } from '@/lib/utils'

// Conditional classes
<div className={cn(
  'rounded-lg p-4',
  isActive && 'bg-blue-500 text-white',
  isDisabled && 'opacity-50 cursor-not-allowed',
  className  // Allow parent to override
)} />

// Without cn(): classes conflict — 'bg-red-500 bg-blue-500' → unpredictable
// With cn(): last wins correctly — 'bg-blue-500'
```

**Always use `cn()` for conditional classes.** String concatenation breaks on conflicts.

---

## 4. Component Variants with `cva`

For components with multiple variants, use `class-variance-authority`:

```bash
npm install class-variance-authority
```

```typescript
// src/components/ui/Button.tsx
import { cva, type VariantProps } from 'class-variance-authority'
import { cn } from '@/lib/utils'

const buttonVariants = cva(
  // Base classes (always applied)
  'inline-flex items-center justify-center rounded-md text-sm font-medium transition-colors focus-visible:outline-none disabled:opacity-50',
  {
    variants: {
      variant: {
        primary:   'bg-primary text-primary-foreground hover:bg-primary/90',
        secondary: 'bg-secondary text-secondary-foreground hover:bg-secondary/80',
        danger:    'bg-destructive text-destructive-foreground hover:bg-destructive/90',
        ghost:     'hover:bg-accent hover:text-accent-foreground',
        link:      'text-primary underline-offset-4 hover:underline',
      },
      size: {
        sm: 'h-9 px-3',
        md: 'h-10 px-4 py-2',
        lg: 'h-11 px-8',
        icon: 'h-10 w-10',
      },
    },
    defaultVariants: {
      variant: 'primary',
      size: 'md',
    },
  }
)

interface ButtonProps
  extends React.ButtonHTMLAttributes<HTMLButtonElement>,
    VariantProps<typeof buttonVariants> {
  asChild?: boolean
}

export function Button({ className, variant, size, ...props }: ButtonProps) {
  return (
    <button
      className={cn(buttonVariants({ variant, size }), className)}
      {...props}
    />
  )
}

export { buttonVariants }
```

```typescript
// Usage
<Button variant="danger" size="lg">Delete</Button>
<Button variant="ghost" size="icon"><TrashIcon /></Button>
```

---

## 5. Class Organization Convention

Long class strings become unreadable. Use this ordering convention:

```typescript
// Order: layout → spacing → sizing → visual → state
<div className={cn(
  // 1. Layout (display, position)
  'flex items-center justify-between',
  // 2. Spacing (margin, padding)
  'px-4 py-2 gap-2',
  // 3. Sizing (width, height)
  'w-full max-w-sm',
  // 4. Visual (color, border, shadow)
  'rounded-lg border bg-white shadow-sm',
  // 5. Typography
  'text-sm font-medium text-gray-900',
  // 6. State (hover, focus, disabled)
  'hover:bg-gray-50 focus:ring-2 disabled:opacity-50',
  // 7. Responsive (breakpoint prefixes last)
  'md:flex-row md:items-start',
  // 8. Dynamic classes (always last)
  isActive && 'border-blue-500',
  className
)} />
```

---

## 6. Responsive Design

Tailwind is mobile-first — unprefixed classes apply to all sizes, prefixed classes apply at that breakpoint and above:

```typescript
// ✓ Mobile-first
<div className="flex flex-col gap-4 md:flex-row md:gap-8 lg:gap-12">

// Breakpoints: sm(640px) md(768px) lg(1024px) xl(1280px) 2xl(1536px)
```

---

## 7. Dark Mode

```typescript
// globals.css: add 'dark' class to <html> for dark mode

// In components: prefix with dark:
<div className="bg-white text-gray-900 dark:bg-gray-900 dark:text-white" />

// Or use CSS variable approach (preferred — see Section 2)
<div className="bg-background text-foreground" />
```

```typescript
// tailwind.config.ts — class-based dark mode (not media query)
const config: Config = {
  darkMode: 'class',  // Toggle dark mode by adding/removing 'dark' class on <html>
  // ...
}
```

---

## 8. shadcn/ui Integration

If using shadcn/ui (common with Next.js + Tailwind):

```bash
npx shadcn@latest init        # Set up shadcn in your project
npx shadcn@latest add button  # Add individual components
```

Components are copied into `src/components/ui/` — they are your code, not a dependency. Modify them freely.

**AI rule:** When adding a new shadcn component, always run `npx shadcn@latest add [component]` rather than having AI write it from scratch. The generated components are production-quality and integrate with your existing config.

---

## 9. Tailwind AI Instructions for CLAUDE.md

```markdown
## Tailwind CSS Conventions

### Setup
- Tailwind version: [version]
- CSS variables for theming: src/app/globals.css (follows shadcn/ui convention)
- cn() utility: src/lib/utils.ts — always use for conditional/merged classes

### Component variants
- Use cva (class-variance-authority) for components with multiple variants
- Base pattern: see src/components/ui/Button.tsx

### Class ordering
1. Layout (flex, grid, position)
2. Spacing (padding, margin, gap)
3. Sizing (width, height)
4. Visual (color, border, rounded, shadow)
5. Typography (text-*, font-*)
6. State (hover:, focus:, disabled:)
7. Responsive (md:, lg:)
8. Dynamic/conditional (always last)

### Rules
- Mobile-first: write base styles for mobile, override for larger screens
- Dark mode: class-based, use dark: prefix or CSS variables
- Never hardcode pixel values — use Tailwind scale (p-4 not p-[16px])
- Arbitrary values ([value]) only when design requires a non-standard value
```
