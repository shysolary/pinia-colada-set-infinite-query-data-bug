<script setup lang="ts">
import {
    setInfiniteQueryData,
    useInfiniteQuery,
    useQueryCache,
} from "@pinia/colada";
import { computed, ref } from "vue";

type PageParam = { cursor?: string };
type PageData = {
    items: string[];
    nextCursor: string | null;
    prevCursor: string | null;
};

const KEYS = [
    ["tracks", "queue-A"],
    ["tracks", "queue-B"],
    ["tracks", "queue-C"],
] as const;

const activeKeyIndex = ref(0);
const activeKey = computed(() => KEYS[activeKeyIndex.value]);

function makePage(id: string): PageData {
    return {
        items: [`${id}-1`, `${id}-2`, `${id}-3`],
        nextCursor: `${id}-next-cursor`,
        prevCursor: `${id}-prev-cursor`,
    };
}

const queryCache = useQueryCache();

// Pre-populate cache synchronously in setup, BEFORE useInfiniteQuery
KEYS.forEach((key) => {
    const id = key[1] as string;
    setInfiniteQueryData<PageData, Error, PageParam>(queryCache, key, {
        pageParams: [{}],
        pages: [makePage(id)],
    });
});

const { data, hasNextPage, hasPreviousPage, loadNextPage, loadPreviousPage } =
    useInfiniteQuery<PageData, Error, PageParam>({
        key: () => activeKey.value,
        initialPageParam: {} as PageParam,
        // staleTime: 5000,
        staleTime: Infinity,
        query: async ({ pageParam }) => {
            const id = activeKey.value[1] as string;
            return makePage(id);
        },
        getNextPageParam: (lastPage) =>
            lastPage.nextCursor ? { cursor: lastPage.nextCursor } : null,
        getPreviousPageParam: (firstPage) =>
            firstPage.prevCursor ? { cursor: firstPage.prevCursor } : null,
    });

function switchKey(idx: number) {
    activeKeyIndex.value = idx;
}
</script>

<template>
    <div
        style="
            font-family: sans-serif;
            max-width: 800px;
            margin: 0 auto;
            padding: 20px;
        "
    >
        <h1>
            setInfiniteQueryData: hasNextPage / hasPreviousPage not recomputed
            on key change
        </h1>

        <div style="margin-bottom: 20px">
            <p><strong>Active key:</strong> {{ activeKey }}</p>
            <button
                v-for="(key, idx) in KEYS"
                :key="idx"
                @click="switchKey(idx)"
                :style="{
                    background: activeKeyIndex === idx ? '#4CAF50' : '#ccc',
                    color: 'white',
                    padding: '8px 16px',
                    marginRight: '8px',
                    border: 'none',
                    cursor: 'pointer',
                }"
            >
                Switch to {{ key[1] }}
            </button>
        </div>

        <div style="margin-bottom: 20px; padding: 12px; background: #f5f5f5">
            <h3 style="margin: 0 0 10px 0">Flags (from useInfiniteQuery):</h3>
            <p style="margin: 4px 0">
                <strong>hasNextPage:</strong> {{ hasNextPage }}
            </p>
            <p style="margin: 4px 0">
                <strong>hasPreviousPage:</strong> {{ hasPreviousPage }}
            </p>
        </div>

        <div style="margin-bottom: 20px; padding: 12px; background: #f9f9f9">
            <h3 style="margin: 0 0 10px 0">Data (from cache):</h3>
            <pre>{{ JSON.stringify(data, null, 2) }}</pre>
        </div>

        <div style="margin-bottom: 20px">
            <button
                @click="loadPreviousPage()"
                :disabled="!hasPreviousPage"
                style="padding: 8px 16px; margin-right: 8px"
            >
                Load Previous ({{ hasPreviousPage ? "enabled" : "disabled" }})
            </button>
            <button
                @click="loadNextPage()"
                :disabled="!hasNextPage"
                style="padding: 8px 16px"
            >
                Load Next ({{ hasNextPage ? "enabled" : "disabled" }})
            </button>
        </div>

        <div
            style="
                margin-top: 20px;
                padding: 12px;
                background: #fff3cd;
                font-size: 14px;
            "
        >
            <strong>Expected:</strong> For every key, both flags should be
            <code>true</code> because <code>makePage()</code> returns non-null
            <code>nextCursor</code> and <code>prevCursor</code>.
            <br />
            <strong>Actual:</strong> Only the initially active key shows
            <code>true</code>. After switching to another key, the flags stay
            <code>false</code> (with <code>staleTime: Infinity</code>) or only
            become <code>true</code> after the stale time elapses and a refetch
            is triggered.
        </div>
    </div>
</template>
