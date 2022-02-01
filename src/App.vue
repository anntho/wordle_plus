<template>
  <div>
    <div class="header flex">
      <span :style="{ color: '#d7dadc' }">WORDLE</span>
      <span :style="{ color: '#63e2b7' }">+</span>
    </div>

    <div class="flex">
      <n-config-provider :theme="darkTheme">
        <n-select 
          v-model:value="grid" 
          :options="options" 
          @update:value="update" />
      </n-config-provider>
    </div>

    <div class="flex container">
      <div class="grid" v-bind:style="styleObject">
        <div 
          class="flex col"
          :class="{ correct: el.correct ? true : false, hint: el.hint ? true: false, wrong: el.wrong ? true : false }"
          v-for="el in elements"
          :key="el.id"
          v-bind:value="el.value"
        >{{ el.value }}</div>
      </div>
    </div>

    <div class="flex keyboard">
      <div class="flex keyboard-wrapper">
        <n-config-provider :theme="darkTheme">
          <n-button 
            @click="click(key)" 
            v-for="key of keys" 
            :key="key.id" 
            :type="key.disabled ? 'tertiary' : 'primary'" 
            secondary class="key"
            :class="{ 'wide': key.wide ? true : false, 'margin-left': key.left ? true : false, 'margin-right': key.right ? true : false }"
          >{{ key.value }}</n-button>
        </n-config-provider>
      </div>
    </div>

    <div class="flex" :style="{ marginTop: '40px' }">
      <n-config-provider :theme="darkTheme">
        <n-button 
          id="restartButton" 
          type="error" 
          :keyboard="false" 
          secondary 
          @click.stop="load"
          class="wide"
        >↻</n-button>
      </n-config-provider>
    </div>
  </div>
</template>

<script>
import jsonData from './assets/words_dictionary.json'
import commonData from './assets/common.json'

import { darkTheme } from 'naive-ui'
import { NConfigProvider, NButton, NSelect } from 'naive-ui'

export default {
  components: {
    NButton,
    NSelect,
    NConfigProvider,
  },
  setup() {
    return {
      darkTheme,
    }
  },
  mounted() {
    document.title = 'Wordle+'
  },
  created() {
    this.load()

    window.addEventListener('keydown', (e) => {
      let ref = e.key.toLowerCase() === 'backspace' ? '<' : e.key.toLowerCase()
      let key = this.keys.find(key => key.value.toLowerCase() === ref)
      if (key) {
        this.click(key)
      }
    })
  },
  methods: {
    load() {
      this.word = ''
      this.index = 0
      this.elements = []
      this.guesses = []
      this.complete = false
      this.length = this.grid

      for (const key of this.keys) {
        key.disabled = false
      }

      this.elements = Array.from(Array(this.length * this.rows).keys()).map(el => { return { id: el, value: '' }})

      if (document.body.offsetWidth < 600) {
        this.styleObject.gridTemplateColumns = `repeat(${this.length}, 35px)`
      }
      else {
        this.styleObject.gridTemplateColumns = `repeat(${this.length}, 50px)`
      }

      const words = this.words.commonWords.filter(w => w.length === this.length).sort(() => .5 - Math.random())
      this.word = words[0].toUpperCase()
      console.log(this.word)
    },
    update(a) {
      this.grid = a
      this.load()
    },
    compare(guess) {
      let correctLetters = Array.from(this.word).map(item => { return { letter: item, correct: false } })

      guess.forEach((item, index) => {
        let element = this.elements.find(element => element.id === item.id)
        if (item.value === correctLetters[index].letter) {
          element.correct = true
          item.correct = true
          correctLetters[index].correct = true
        }
      })

      guess.forEach(item => {
        let element = this.elements.find(element => element.id === item.id)

        if (element.correct) {
          return
        }

        let occurrences = correctLetters.filter(correctLetter => correctLetter.letter === item.value)

        let correctLettersAlreadyCounted = 0
        let lettersThatMayneedHint = guess.filter(g => g.value === item.value)

        let needsHint = true

        for (const letter of lettersThatMayneedHint) {
          if (letter.hint) {
            needsHint = false
            break
          }
        }

        for (const occurrence of occurrences) {
          if ((occurrence.letter === item.value) && occurrence.correct) {
            correctLettersAlreadyCounted++
          }
        }

        if ((occurrences.length > correctLettersAlreadyCounted) && needsHint) {
          element.hint = true
          item.hint = true
        }
        else {
          element.wrong = true

          if (occurrences.length === 0) {
            let key = this.keys.find(key => key.value === item.value)
            key.disabled = true
          }
        }
      })

      if (this.word === guess.map(g => g.value).join('')) {
        this.complete = true
      }
    },
    enter() {
      let { currentRowMaximum } = this.getCurrentRow()

      if (this.index === currentRowMaximum) {
        const ref = Array.from(this.elements)
        const guess = ref.splice(this.index-this.length, this.length)
        const str = guess.map(g => g.value).join('').toLowerCase()

        if (!this.dictionary[str]) {
          return false
        }

        this.guesses.push(...guess)
        this.compare(guess)
      }
    },
    backspace() {
      let { element } = this.getElement(true)
      let { currentRowMinimum } = this.getCurrentRow()

      if (this.index > currentRowMinimum) {
        element.value = ''
        this.index--
      }
    },
    add(key) {
      if (this.complete) {
        return false
      }

      let { element } = this.getElement()
      let { currentRowMaximum } = this.getCurrentRow()

      if (this.index < currentRowMaximum) {
        element.value = key.value
        if (this.index < this.elements.length) {
          this.index++
        }
      }
    },
    click(key) {
      switch (key.value){
        case 'Enter':
          this.enter()
          break
        case '<':
          this.backspace()
          break
        default:
          this.add(key)
          break
      }
    },
    getElement(previous = false) {
      let index = previous ? this.index - 1 : this.index
      let element = this.elements.find(element => element.id === index)
      let elementIndex = this.elements.indexOf(element)
      return { element, elementIndex }
    },
    getCurrentRow() {
      let currentRow = this.guesses.length / this.length
      let currentRowMaximum = (currentRow + 1) * this.length
      let currentRowMinimum = ((currentRow + 1) * this.length ) - this.length
      return { currentRow, currentRowMaximum, currentRowMinimum }
    }
  },
  computed: {
    globalMaximum() {
      return this.guesses.length && ((this.guesses.length + 1) / this.rows) === this.length
    },
  },
  data() {
    return {
      word: '',
      index: 0,
      length: 5,
      rows: 6,
      elements: [],
      guesses: [],
      grid: 5,
      options: [
        { label: '2 x 6', value: 2 },
        { label: '3 x 6', value: 3 },
        { label: '4 x 6', value: 4 },
        { label: '5 x 6', value: 5 },
        { label: '6 x 6', value: 6 },
        { label: '7 x 6', value: 7 },
        { label: '8 x 6', value: 8 },
      ],
      dictionary: jsonData,
      words: commonData,
      styleObject: {
        gridTemplateColumns: ''
      },
      swalObjects: {
        notValid: {
          text: 'Not a valid word!',
          icon: 'error',
          allowOutsideClick: false,
          allowEnterKey: false,
        },
      },
      keys: [
        { id: 1, value: 'Q' },
        { id: 2, value: 'W' },
        { id: 3, value: 'E' },
        { id: 4, value: 'R' },
        { id: 5, value: 'T' },
        { id: 6, value: 'Y' },
        { id: 7, value: 'U' },
        { id: 8, value: 'I' },
        { id: 9, value: 'O' },
        { id: 10, value: 'P' },
        { id: 11, value: 'A', left: true },
        { id: 12, value: 'S' },
        { id: 13, value: 'D' },
        { id: 14, value: 'F' },
        { id: 15, value: 'G' },
        { id: 16, value: 'H' },
        { id: 17, value: 'J' },
        { id: 18, value: 'K' },
        { id: 19, value: 'L', right: true },
        { id: 20, value: 'Enter', wide: true },
        { id: 21, value: 'Z' },
        { id: 22, value: 'X' },
        { id: 23, value: 'C' },
        { id: 24, value: 'V' },
        { id: 25, value: 'B' },
        { id: 26, value: 'N' },
        { id: 27, value: 'M' },
        { id: 28, value: '<', wide: true },
      ],
    }
  },
}
</script>

<style>
html, body {
  width: 100%;
  background-color: #121213;
}

#app {
  height: 100%;
}

.header {
  font-size: 2rem;
  font-weight: 700;
  width: 100%;
  margin-top: 10px;
  margin-bottom: 10px;
}

.flex {
  display: flex;
  justify-content: center;
  align-items: center;
}

.container {
  width: 100%;
  margin-top: 60px;
}

.grid {
  display: grid;
  grid-gap: 10px;
  justify-content: center;
}

.col {
  border: 2px solid #3a3a3c;
  height: 35px;
  width: 35px;
  color: #fff;
  font-weight: 700;
  font-size: inherit;
}

.keyboard {
  margin-top: 80px;
  font-weight: 700;
  color: #fff;
}

.keyboard-wrapper {
  width: 340px;
  flex-wrap: wrap;
}

.key {
  cursor: pointer;
  font-weight: 700;
  text-transform: capitalize;
}

.correct {
  color: #63e2b7;
  background-color: #63e2b729;
  border: 2px solid #63e2b729;
}

.hint {
  color: #f2c97d;
  background-color: #f2c97d29;
  border: 2px solid #f2c97d29;
}

.wrong {
  color: #ffffffd1;
  background-color: #ffffff0f;
  border: 2px solid #ffffff0f;
}

.n-select {
  min-width: 100px;
}

.wide {
  width: 45px !important;
}

.margin-left {
  margin-left: 18px !important;
}

.margin-right {
  margin-right: 10px !important;
}

.n-button {
  margin: 2px;
  height: 50px;
  width: 30px;
}

@media only screen and (min-width: 600px) {
  .col {
    height: 50px;
    width: 50px;
    font-size: 1.4rem;
  }

  .keyboard-wrapper {
    width: 450px;
  }

  .wide {
    width: 60px !important;
  }

  .margin-left {
    margin-left: 20px !important;
  }

  .margin-right {
    margin-right: 10px !important;
  }

  .n-button {
    margin: 2px;
    height: 40px;
    width: 40px;
  }
}
</style>
