<template>
    <div class="word-search-game">
        <word-list
            :words="words"
            :found-words="foundWords"
        />
        <div class="matrix word-search-game__matrix">
            <template
                v-for="(row, row_key) in matrix"
                :key="row_key-1"
            >
                <div
                    v-for="(letter, col_key) in row"
                    :key="`${row_key}_${col_key}`"
                    :class="letterTileClasses(col_key, row_key)"
                    class="matrix-cell"
                >
                    <div
                        :data-x="col_key"
                        :data-y="row_key"
                        class="cell"
                        @mousedown.prevent="wordSelectStart"
                        @mouseup="wordSelectStop"
                        @mouseenter="debounceActiveCell = `${col_key}_${row_key}`"
                        @touchstart.prevent="wordSelectStart"
                        @touchend="wordSelectStop"
                        @touchmove="wordSelectUpdate"
                    >
                        <svg
                            style="border:1px solid black"
                            width="100%"
                            height="100%"
                            viewBox="0 0 18 18"
                        >
                            <text
                                x="50%"
                                y="13"
                                text-anchor="middle"
                            >
                                {{ letter }}
                            </text>
                        </svg>
                    </div>
                    <div
                        v-for="(wordLineData, i) in wordLinesForTile(col_key, row_key)"
                        :key="`${row_key}_${col_key}_${i}`"
                        :class="wordLineClasses(wordLineData)"
                    />
                </div>
            </template>
        </div>
    </div>
</template>

<script setup>
import {computed, onMounted, ref, watch} from 'vue'
import WordList from './WordList.vue'

const props = defineProps({
    matrix: {
        type: Array,
        required: true,
    },
    words: {
        type: Array,
        required: true,
    },
})

const debounceActiveCell = ref('')
const selectedFrom = ref(null)
const selectedTo = ref(null)
const dragging = ref(false)
const foundWords = ref([])
const foundCells = ref([])

const done = computed(() => foundWords.value.length === props.words.length)

const selectedCells = computed(() => {
    let cells = []
    if (selectedFrom.value && selectedTo.value) {
        if (selectedFrom.value.x === selectedTo.value.x) {
            // horizontal direction (-)
            let from = Math.min(selectedFrom.value.y, selectedTo.value.y)
            let to = Math.max(selectedFrom.value.y, selectedTo.value.y)
            for (let i = from; i <= to; i++) {
                cells.push({x: selectedFrom.value.x, y: i})
            }
        } else if (selectedFrom.value.y === selectedTo.value.y) {
            // vertical direction (|)
            let from = Math.min(selectedFrom.value.x, selectedTo.value.x)
            let to = Math.max(selectedFrom.value.x, selectedTo.value.x)
            for (let i = from; i <= to; i++) {
                cells.push({x: i, y: selectedFrom.value.y})
            }
        } else if (selectedFrom.value.x - selectedTo.value.x === selectedFrom.value.y - selectedTo.value.y) {
            // right-down direction (\)
            let x_from = Math.min(selectedFrom.value.x, selectedTo.value.x)
            let x_to = Math.max(selectedFrom.value.x, selectedTo.value.x)
            let y_from = Math.min(selectedFrom.value.y, selectedTo.value.y)
            for (let i = x_from; i <= x_to; i++) {
                cells.push({x: i, y: y_from})
                y_from++
            }
        } else if (selectedFrom.value.x - selectedTo.value.x === selectedTo.value.y - selectedFrom.value.y) {
            // right-up direction (/)
            let x_from = Math.min(selectedFrom.value.x, selectedTo.value.x)
            let x_to = Math.max(selectedFrom.value.x, selectedTo.value.x)
            let y_to = Math.max(selectedFrom.value.y, selectedTo.value.y)
            for (let i = x_from; i <= x_to; i++) {
                cells.push({x: i, y: y_to})
                y_to--
            }
        }
    }
    return cells
})

let debounceSetActiveCell

onMounted(() => {
    debounceSetActiveCell = debounce(wordSelectUpdate, 100)
})

watch(debounceActiveCell, () => {
    debounceSetActiveCell()
})


function letterTileClasses(x, y) {
    const foundCell = selectedCells.value.find(cell => cell.x === x && cell.y === y)
    if (foundCell) {
        return 'selected'
    }
    if (done.value && ! foundCell) {
        return 'done'
    }
}

function wordSelectStart(event) {
    dragging.value = true
    const touchedElement = event.target.closest('div.cell')
    if (touchedElement && touchedElement.dataset && touchedElement.dataset.x) {
        const {x, y} = touchedElement.dataset
        selectedFrom.value = {
            x: parseInt(x, 10),
            y: parseInt(y, 10),
        }
        return true
    }
    return false
}

function wordSelectStop(event) {
    dragging.value = false
    let selected = []
    selectedCells.value.forEach(coordinate => {
        selected.push(props.matrix[coordinate.y][coordinate.x])
    })

    if (selectedCells.value.length < 2) {
        selectedFrom.value = null
        selectedTo.value = null
        return
    }

    let word = props.words.find(item => item.value === selected.join(''));
    let x_start = selectedCells.value[0]?.x
    let y_start = selectedCells.value[0]?.y
    let x_end = selectedCells.value[selectedCells.value.length - 1].x
    let y_end = selectedCells.value[selectedCells.value.length - 1].y
    if (!word) {
        const selected_word = selected.reverse().join('')
        word = props.words.find(item => item.value === selected_word)
        x_end = selectedCells.value[0]?.x
        y_end = selectedCells.value[0]?.y
        x_start = selectedCells.value[selectedCells.value.length - 1].x
        y_start = selectedCells.value[selectedCells.value.length - 1].y
    }
    if (word) {
        let exists = false;
        foundWords.value.forEach(w => {
            if (w.value === word.value) {
                exists = true
            }
        })

        if (!exists) {
            selectedCells.value.forEach(coordinate => {
                foundCells.value.push({x: coordinate.x, y:coordinate.y})
            })

            foundWords.value.push({
                value: word.value,
                x_start: x_start,
                y_start: y_start,
                direction: wordDirection(x_start, y_start, x_end, y_end),
                length: word.value.length
            })
        }
    }
    selectedFrom.value = null
    selectedTo.value = null
}

function wordSelectUpdate(event = null) {
    if (! dragging.value) return

    let x, y

    if (event) {
        let touch = event
        if (event.type.indexOf('touch') === 0) {
            touch = event.changedTouches.item(0)
        }

        const touchedElement = document.elementFromPoint(touch.clientX, touch.clientY).closest('div.cell')

        if (touchedElement && touchedElement.dataset && touchedElement.dataset.x) {
            x = parseInt(touchedElement.dataset.x, 10)
            y = parseInt(touchedElement.dataset.y, 10)
        }
    } else {
        [x, y] = debounceActiveCell.value.split('_')
    }
    selectedTo.value = {x: parseInt(x), y: parseInt(y)}
}

function wordDirection(x_start, y_start, x_end, y_end) {
    if (x_start === x_end && y_start > y_end) {
        // up
        return 0
    } else if (x_start < x_end && y_start > y_end) {
        // up-right
        return 1
    } else if (x_start < x_end && y_start === y_end) {
        // right
        return 2
    } else if (x_start < x_end && y_start < y_end) {
        // down-right
        return 3
    } else if (x_start === x_end && y_start < y_end) {
        // down
        return 4
    } else if (x_start > x_end && y_start < y_end) {
        // down-left
        return 5
    } else if (x_start > x_end && y_start === y_end) {
        // left
        return 6
    } else if (x_start > x_end && y_start > y_end) {
        // up-left
        return 7
    }
}

function wordLinesForTile(x, y) {
    return foundWords.value.filter(w => (w.x_start === x) && (w.y_start === y))
}

function wordLineClasses(wordLine) {
    const classes = [
        'word-strike',
        'word-strike-direction-' + wordLine.direction,
        'word-strike-length-' + wordLine.length,
    ]
    // Odd directions are diagonal
    if (wordLine.direction % 2 === 1) {
        classes.push('word-strike-diagonal')
    }
    return classes
}

const debounce = (fn, delay) => {
    let timeout

    return (...args) => {
        if (timeout) {
            clearTimeout(timeout)
        }

        timeout = setTimeout(() => {
            fn(...args)
        }, delay)
    }
}
</script>

<style lang="scss" scoped>
$local-green: #32cd99;
.word-search-game {
    margin-top: 20px;
    max-width: 640px;

    &__matrix {
        width: 100%;
        display: grid;
        grid-template-columns: repeat(10, 1fr);
        grid-template-rows: repeat(10, 1fr);
    }

    .matrix-cell {
        text-align: center;
        aspect-ratio: 1/1;
        &.selected {
            background-color: #ccc;
        }
        &.done {
            transition: transform 3s;
            transform: rotate(360deg) scale(.01, .01);
        }
        .cell {
            display: flex;
            align-items: center;
            justify-content: center;
            width: 100%;
            height: 100%;
            user-select: none;
            position: relative;
            z-index: 2011;
            svg {
                position: absolute;
                left: 0;
                top: 0;
                line-height: 0;
                width: 100%;
                font-size: 0.7rem;
                fill: var(--color-text);
            }
        }
    }

    .word-strike {
        height: 100%;
        width: 100%;
        position: relative;
        z-index: 100;
        transform-origin: 0 0;
        opacity: .5;
        border-radius: 25%/150%;
        margin: -100% 0 0 0;
        padding: 0;
        background-color: $local-green;

        @for $i from 0 to 8 {
            &.word-strike-direction-#{$i} {
                transform: rotate(($i * 45deg) - 90deg);
            }
        }
        @for $i from 0 to 20 {
            &:not(.word-strike-diagonal).word-strike-length-#{$i} { width: calc($i * 100%); transform-origin: calc(1 / $i / 2 * 100%) 50%; }
            &.word-strike-diagonal.word-strike-length-#{$i} { width: $i * 100% * 1.32; transform-origin: calc(1 / $i / 2 / 1.38 * 100%) 50%; } // 1.414214 is a hypotenuse multiplier
        }
    }
}
</style>
