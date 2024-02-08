<template>
    <div class="words-list">
        <div
            v-for="word in words"
            :key="word"
            class="words-list__item"
        >
            <span
                class="words-list__value"
                :class="{found: isFound(word)}"
            >
                {{ word }}
            </span>
        </div>
    </div>
</template>

<script setup>
const props = defineProps({
    words: {
        type: Array,
        required: true,
    },
    foundWords: {
        type: Array,
        required: true,
    }
})

const isFound = word => props.foundWords.find(w => w === word)
</script>
<style lang="scss" scoped>
.words-list {
    margin-bottom: 20px;
    &__item {
        display: inline-block;
        margin-bottom: 5px;
        margin-right: 10px;
        .found {
            text-decoration: line-through;
        }
    }
}
</style>
