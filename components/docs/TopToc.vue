<template>
    <div v-if="generated.length > 0" class="top-toc-wrapper">
        <div class="top-toc">
            <div class="top-toc-header">
                <strong class="h6 top-toc-title">Table of Contents</strong>
            </div>
            <nav class="top-toc-nav">
                <template v-for="(item, index) in flattenedToc" :key="index">
                    <a
                        v-if="item.id && item.text"
                        :href="`#${item.id}`"
                        :class="['top-toc-link', `depth-${item.depth}`]"
                        @click.prevent="menuNavigate"
                    >
                        {{ item.text }}
                    </a>
                </template>
            </nav>
        </div>
    </div>
</template>

<script setup lang="ts">
import { computed } from "vue";
import { useRoute } from "vue-router";

type TocLink = {
    id?: string;
    text?: string;
    depth?: number;
    children?: TocLink[];
};

const props = defineProps<{
    page: any;
}>();

const route = useRoute();

const generated = computed<TocLink[]>(() => props.page?.body?.toc?.links ?? []);

// Flatten the TOC structure for horizontal display
const flattenedToc = computed<TocLink[]>(() => {
    const flatten = (items: TocLink[]): TocLink[] => {
        const result: TocLink[] = [];
        items.forEach((item) => {
            // Only include h2 and h3 headings for the top TOC
            if (item.depth && item.depth >= 2 && item.depth <= 3 && item.id && item.text) {
                result.push({
                    id: item.id,
                    text: item.text,
                    depth: item.depth,
                });
            }
            // Recursively add children
            if (item.children && item.children.length > 0) {
                result.push(...flatten(item.children));
            }
        });
        return result;
    };
    return flatten(generated.value);
});

const getFixedHeaderOffset = (): number => {
    const selectors = ["header", ".site-header", ".navbar", ".topbar", ".bd-title", ".page-header"];
    for (const sel of selectors) {
        const el = document.querySelector(sel) as HTMLElement | null;
        if (el) {
            const style = window.getComputedStyle(el);
            const isFixed =
                style.position === "fixed" ||
                style.position === "sticky" ||
                Math.round(el.getBoundingClientRect().top) === 0;
            if (isFixed) return el.getBoundingClientRect().height;
        }
    }
    return 160;
};

const scrollToElement = (id: string): void => {
    const element = document.getElementById(id);
    if (!element) return;

    const offset = getFixedHeaderOffset();
    const rectTop = element.getBoundingClientRect().top + window.pageYOffset;
    const top = Math.max(0, rectTop - offset - 8);
    window.scrollTo({ top, behavior: "smooth" });
};

const menuNavigate = (e: Event): void => {
    const anchor = (e.target as HTMLElement)?.closest("a") as HTMLAnchorElement | null;
    const href = anchor?.getAttribute("href");
    const id = href?.startsWith("#") ? href.substring(1) : anchor?.name || "";

    if (!id) return;

    scrollToElement(id);

    try {
        history.pushState(null, "", `#${id}`);
        window.dispatchEvent(new Event("hashchange"));
    } catch {
        window.location.hash = id;
    }
};
</script>

<style lang="scss" scoped>
@import "../../assets/styles/variable";

.top-toc-wrapper {
    margin: 2rem 0 3rem 0;
    padding: 1.5rem;
    background-color: $black-2;
    border: 1px solid $black-6;
    border-radius: 8px;

    @include media-breakpoint-down(lg) {
        margin: 1.5rem 0 2rem 0;
        padding: 1rem;
    }
}

.top-toc {
    max-width: 100%;
}

.top-toc-header {
    margin-bottom: 1rem;
}

.h6 {
    color: $white-1;
    font-size: $font-size-sm;
    line-height: 1.875rem;
    font-weight: 600;
    padding-top: 0;
}

.top-toc-nav {
    display: flex;
    flex-direction: column;
    gap: 0.5rem 0.75rem;
    align-items: left;
    @include font-size(0.875rem);
    padding-bottom: 1.5rem;
    border-bottom: 1px solid $black-6;

    @include media-breakpoint-down(lg) {
        gap: 0.5rem 0.75rem;
        padding-bottom: 1rem;
    }
}

.top-toc-link {
    display: inline-block;
    padding: 0 0.75rem;
    color: var(--ks-content-secondary);
    text-decoration: none;
    font-size: 12px;
    font-weight: 500;
    cursor: pointer;
    white-space: nowrap;
    border-left: 1px solid transparent;
    transition: all 0.2s ease;

    code {
        font: inherit;
    }

    @for $i from 2 through 6 {
        &.depth-#{$i} {
            padding-left: calc(0.5rem + 0.75rem);
        }
    }

    &:hover {
        color: $purple;
        border-left: 1px solid $purple-36 !important;
    }

    &.depth-2 {
        font-weight: 600;
    }

    &.depth-3 {
        font-weight: 500;
    }

    @include media-breakpoint-down(lg) {
        font-size: 11px;
        padding: 0 0.5rem;
    }
}
</style>