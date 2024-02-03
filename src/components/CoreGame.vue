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

<script>
import WordList from '@/components/WordList.vue'
export default {
    components: {WordList},
    props: {
        matrix: {
            type: Array,
            required: true,
        },
        words: {
            type: Array,
            required: true,
        },
    },
    data() {
        return {
            debounceActiveCell: '',
            selectedFrom: null,
            selectedTo: null,
            dragging: false,
            foundWords: [],
            foundCells: [],
            showRobot: false,
        }
    },
    computed: {
        done() {
            return this.foundWords.length === this.words.length
        },
        selectedCells() {
            let cells = []
            if (this.selectedFrom && this.selectedTo) {
                if (this.selectedFrom.x === this.selectedTo.x) {
                    // horizontal direction (-)
                    let from = Math.min(this.selectedFrom.y, this.selectedTo.y)
                    let to = Math.max(this.selectedFrom.y, this.selectedTo.y)
                    for (let i = from; i <= to; i++) {
                        cells.push({x: this.selectedFrom.x, y: i})
                    }
                } else if (this.selectedFrom.y === this.selectedTo.y) {
                    // vertical direction (|)
                    let from = Math.min(this.selectedFrom.x, this.selectedTo.x)
                    let to = Math.max(this.selectedFrom.x, this.selectedTo.x)
                    for (let i = from; i <= to; i++) {
                        cells.push({x: i, y: this.selectedFrom.y})
                    }
                } else if (this.selectedFrom.x - this.selectedTo.x === this.selectedFrom.y - this.selectedTo.y) {
                    // right-down direction (\)
                    let x_from = Math.min(this.selectedFrom.x, this.selectedTo.x)
                    let x_to = Math.max(this.selectedFrom.x, this.selectedTo.x)
                    let y_from = Math.min(this.selectedFrom.y, this.selectedTo.y)
                    for (let i = x_from; i <= x_to; i++) {
                        cells.push({x: i, y: y_from})
                        y_from++
                    }
                } else if (this.selectedFrom.x - this.selectedTo.x === this.selectedTo.y - this.selectedFrom.y) {
                    // right-up direction (/)
                    let x_from = Math.min(this.selectedFrom.x, this.selectedTo.x)
                    let x_to = Math.max(this.selectedFrom.x, this.selectedTo.x)
                    let y_to = Math.max(this.selectedFrom.y, this.selectedTo.y)
                    for (let i = x_from; i <= x_to; i++) {
                        cells.push({x: i, y: y_to})
                        y_to--
                    }
                }
            }
            return cells
        }
    },
    watch: {
        debounceActiveCell: function () {
            this.debounceSetActiveCell()
        },
    },
    mounted() {
        this.debounceSetActiveCell = this.debounce(this.wordSelectUpdate, 100)
    },
    methods: {
        letterTileClasses(x, y) {
            const foundCell = this.selectedCells.find(cell => cell.x === x && cell.y === y)
            if (foundCell) {
                return 'selected'
            }
            if (this.done && ! foundCell) {
                return 'done'
            }
        },
        wordSelectStart(event) {
            this.dragging = true
            const touchedElement = event.target.closest('div.cell')
            if (touchedElement && touchedElement.dataset && touchedElement.dataset.x) {
                const {x, y} = touchedElement.dataset
                this.selectedFrom = {
                    x: parseInt(x, 10),
                    y: parseInt(y, 10),
                }
                return true
            }
            return false
        },
        wordSelectStop(event) {
            this.dragging = false
            let selected = []
            this.selectedCells.forEach(coordinate => {
                selected.push(this.matrix[coordinate.y][coordinate.x])
            })

            if (this.selectedCells.length < 2) {
                this.selectedFrom = null
                this.selectedTo = null
                return
            }

            let word = this.words.find(item => item.value === selected.join(''));
            let x_start = this.selectedCells[0]?.x
            let y_start = this.selectedCells[0]?.y
            let x_end = this.selectedCells[this.selectedCells.length - 1].x
            let y_end = this.selectedCells[this.selectedCells.length - 1].y
            if (!word) {
                const selected_word = selected.reverse().join('')
                word = this.words.find(item => item.value === selected_word)
                x_end = this.selectedCells[0]?.x
                y_end = this.selectedCells[0]?.y
                x_start = this.selectedCells[this.selectedCells.length - 1].x
                y_start = this.selectedCells[this.selectedCells.length - 1].y
            }
            if (word) {
                let exists = false;
                this.foundWords.forEach(w => {
                    if (w.value === word.value) {
                        exists = true
                    }
                })

                if (!exists) {
                    this.selectedCells.forEach(coordinate => {
                        this.foundCells.push({x: coordinate.x, y:coordinate.y})
                    })

                    this.foundWords.push({
                        value: word.value,
                        x_start: x_start,
                        y_start: y_start,
                        direction: this.wordDirection(x_start, y_start, x_end, y_end),
                        length: word.value.length
                    })
                }
            }
            this.selectedFrom = null
            this.selectedTo = null
        },
        wordSelectUpdate(event = null) {
            if (! this.dragging) return

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
                [x, y] = this.debounceActiveCell.split('_')
            }
            this.selectedTo = {x: parseInt(x), y: parseInt(y)}
        },

        wordDirection(x_start, y_start, x_end, y_end) {
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
        },
        wordLinesForTile(x, y) {
            return this.foundWords.filter(w => (w.x_start === x) && (w.y_start === y))
        },
        wordLineClasses(wordLine) {
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
        },
        debounce(f, t) {
            return function (args) {
                let previousCall = this.lastCall;
                this.lastCall = Date.now();
                if (previousCall && ((this.lastCall - previousCall) <= t)) {
                    clearTimeout(this.lastCallTimer);
                }
            this.lastCallTimer = setTimeout(() => f(args), t);
            }
        }
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
