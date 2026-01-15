# Custom Markdown Components Architecture Guide

## Overview

This codebase uses **Nuxt Content** which automatically registers Vue components from the `components/content/` directory as markdown directives.

## How It Works

### 1. Auto-Registration

**Nuxt Content automatically registers components** in `components/content/` directory:
- `components/content/Alert.vue` → `::alert::`
- `components/content/Collapse.vue` → `::collapse::`
- `components/content/ChildCard.vue` → `::ChildCard::`
- `components/content/Toc.vue` → `::toc::`

**No configuration needed!** Just place your component in `components/content/` and it's available.

### 2. Component Types

#### Type A: Content-Rendering Components (Standard)

These components render visible content when used in markdown.

**Example: `Alert.vue`**
```vue
<template>
    <Alert :type="type">
        <slot />
    </Alert>
</template>

<script lang="ts" setup>
    import Alert from "@kestra-io/ui-libs/src/components/content/Alert.vue";
    
    defineProps<{
        type: InstanceType<typeof Alert>["$props"]["type"];
    }>();
</script>
```

**Usage in markdown:**
```markdown
:::alert{type="info"}
This is an info alert.
:::
```

**How it works:**
1. Nuxt Content parses `::alert::` and creates a component node in `body.children`
2. `ContentRenderer` automatically renders it with the component
3. The component receives props and slot content

#### Type B: Marker Components (Special Case)

These components serve as markers but don't render visible content. They're detected separately.

**Example: `Toc.vue`**
```vue
<template>
    <!-- This component is a marker for the TopToc component -->
    <!-- It doesn't render anything itself, but its presence indicates the TOC should be shown -->
    <div style="display: none;"></div>
</template>

<script setup>
// This component serves as a marker in markdown
// When ::toc:: is used in markdown, it will be detected by the page component
</script>
```

**Usage in markdown:**
```markdown
::toc::
```

**How it works:**
1. `::toc::` is parsed and included in `body.children`
2. The page component (`pages/docs/[...slug].vue`) detects its presence
3. A separate component (`TopToc.vue`) is conditionally rendered based on detection

### 3. Detection Methods

#### For Content Components (Standard)
No detection needed - `ContentRenderer` handles everything automatically.

#### For Marker Components (Like Toc)

**Method 1: Check raw markdown** (Current implementation)
```javascript
const showTopToc = computed(() => {
    if (!page.value) return false;
    
    // Check the raw markdown content if available
    if (page.value._raw?.body) {
        return page.value._raw.body.includes('::toc::');
    }
    
    return false;
});
```

**Method 2: Check body.children** (More robust)
```javascript
const showTopToc = computed(() => {
    if (!page.value?.body?.children) return false;
    
    // Check if Toc component exists in the body children
    const hasTocComponent = page.value.body.children.some((child: any) => {
        return child.tag === 'Toc' || 
               child.type === 'element' && child.tag === 'toc' ||
               child.component === 'Toc';
    });
    
    return hasTocComponent;
});
```

**Method 3: Check stringified body** (Fallback)
```javascript
if (page.value.body) {
    const bodyText = JSON.stringify(page.value.body);
    return bodyText.includes('"component":"Toc"') || 
           bodyText.includes('"tag":"Toc"');
}
```

### 4. Component Structure

#### Standard Component Pattern

```vue
<template>
    <!-- Your component UI -->
    <div class="my-component">
        <slot />
    </div>
</template>

<script setup lang="ts">
    // Props if needed
    defineProps<{
        title?: string;
        type?: string;
    }>();
</script>

<style lang="scss" scoped>
    @import "../../assets/styles/variable";
    
    .my-component {
        // Your styles
    }
</style>
```

#### Marker Component Pattern

```vue
<template>
    <!-- Hidden marker - doesn't render visible content -->
    <div style="display: none;"></div>
</template>

<script setup>
// Marker component - detected separately by page component
</script>
```

### 5. Rendering Flow

```
Markdown File
    ↓
::ComponentName::
    ↓
Nuxt Content Parser
    ↓
body.children[] (AST structure)
    ↓
ContentRenderer Component
    ↓
Auto-renders component from components/content/
```

### 6. Examples from Codebase

#### Alert Component
- **File:** `components/content/Alert.vue`
- **Usage:** `:::alert{type="info"}...:::`
- **Renders:** Visible alert box with content

#### Collapse Component
- **File:** `components/content/Collapse.vue`
- **Usage:** `:::collapse{title="Details"}...:::`
- **Renders:** Collapsible section with title

#### ChildCard Component
- **File:** `components/content/ChildCard.vue`
- **Usage:** `:::ChildCard:::`
- **Renders:** Grid of child page cards

#### Toc Component (Marker)
- **File:** `components/content/Toc.vue`
- **Usage:** `::toc::`
- **Renders:** Nothing (hidden marker)
- **Effect:** Triggers `TopToc` component to render separately

## Creating Your Own Component

### Step 1: Create Component File

Create `components/content/MyComponent.vue`:

```vue
<template>
    <div class="my-custom-component">
        <h3 v-if="title">{{ title }}</h3>
        <slot />
    </div>
</template>

<script setup lang="ts">
    defineProps<{
        title?: string;
        variant?: 'primary' | 'secondary';
    }>();
</script>

<style lang="scss" scoped>
    @import "../../assets/styles/variable";
    
    .my-custom-component {
        padding: 1rem;
        background: $black-2;
        border: 1px solid $black-6;
    }
</style>
```

### Step 2: Use in Markdown

```markdown
:::MyComponent{title="My Title" variant="primary"}
This is the content inside the component.
:::
```

### Step 3: (Optional) Detection for Marker Components

If you need to detect the component's presence (like `Toc`):

```javascript
// In your page component
const hasMyComponent = computed(() => {
    if (!page.value?._raw?.body) return false;
    return page.value._raw.body.includes('::MyComponent::');
});
```

## Key Points

1. **Auto-registration:** Components in `components/content/` are automatically available
2. **Naming:** Component name in markdown matches the file name (case-sensitive)
3. **Props:** Use `{prop="value"}` syntax in markdown
4. **Slots:** Content between `:::` tags becomes slot content
5. **Detection:** For marker components, check `_raw.body` or `body.children`
6. **Rendering:** `ContentRenderer` handles standard components automatically

## Current Implementation: Toc Component

The `Toc` component is a **marker component** that:
1. Doesn't render visible content (hidden div)
2. Is detected by checking `_raw.body` for `::toc::`
3. Triggers the `TopToc` component to render separately
4. Allows docs writers to control TOC visibility with `::toc::`

This pattern is useful when you need:
- Conditional rendering based on markdown presence
- Components that affect page layout but don't render inline
- Features that need separate detection logic
