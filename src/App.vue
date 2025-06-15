<template>
  <div class="field">
    <label>Voice</label>
    <select v-model="voice">
      <option v-for="v in voices" :value="v.name">{{ v.name }}</option>
    </select>
  </div>
  <div class="field">
    <label>Question</label>
    <textarea v-model="message" />
  </div>
  <div class="field">
    <button @click="ask(message)" :disabled="processing">Speak</button>
  </div>
  <h2 v-if="ai.reasoning">Thoughts</h2>
  <div v-if="ai.reasoning" class="markdown-body" v-html="parse(ai.reasoning)" />
  <h2 v-if="ai.reasoning">Response</h2>
  <div class="markdown-body" v-html="parse(ai.content)" />
  <h2 v-if="sentences.length > 0">Sentences to speak</h2>
  <div v-if="sentences.length > 0">{{ sentences.map(s => s.replaceAll(END_SENTENCE_TOKEN, '')) }}</div>
  <h2 v-if="spoken.length > 0">Sentences spoken</h2>
  <div v-if="spoken.length > 0">{{ spoken.map(s => s.replaceAll(END_SENTENCE_TOKEN, '')) }}</div>
</template>

<script setup>
import { ref, onMounted, shallowRef } from 'vue'
import OpenAI from 'openai'
import * as marked from 'marked'
import strip from 'remove-markdown'

const message = ref('gdzie leżą gliwice?')
const voices = shallowRef([])
const voice = ref('')
const processing = ref(false)
const ai = ref({ role: 'assistant', content: '', reasoning: '' })

const END_SENTENCE_TOKEN = '<|end_of_sentence|>'

function parse(content) {
  return content ? marked.parse(content.replaceAll(END_SENTENCE_TOKEN, '.')) : ''
}

const sentences = ref([])
const spoken = ref([])

/**
 * @param {string[]} text
 */
function speak() {
  const line = sentences.value.shift()
  if (!line) return

  const sentence = new SpeechSynthesisUtterance(strip(line.replaceAll(END_SENTENCE_TOKEN, '')))
  sentence.voice = voices.value.find(v => v.name === voice.value)
  sentence.onend = () => {
    if (sentences.value.length > 0) {
      speak()
    } else {
      while (spoken.value.length > 0) spoken.value.pop()
    }
  }

  window.speechSynthesis.speak(sentence)
  spoken.value.push(line)
}

const sleep = ms => new Promise(resolve => setTimeout(resolve, ms))

async function ask(text) {
  processing.value = true
  try {
    ai.value = { role: 'assistant', content: '', reasoning: '' }

    const client = new OpenAI({
      apiKey: import.meta.env.VITE_APP_OPENAI_API_KEY,
      baseURL: import.meta.env.VITE_APP_OPENAI_BASE_URL,
      dangerouslyAllowBrowser: true,
    })
    const response = await client.chat.completions.create({
      model: import.meta.env.VITE_APP_MODEL,
      messages: [
        { role: 'system', content: `Jesteś polskim asystentem. Odpowiadasz tylko po polsku. Zakończ zdania symbolem "${END_SENTENCE_TOKEN}" zamiast kropki.` },
        { role: 'user', content: text }
      ],
      stream: true,
    })

    let thinking = false

    for await (const chunk of response) {
      if (chunk.choices[0].delta.content === '<think>') {
        thinking = true
      } else if (chunk.choices[0].delta.content === '</think>') {
        thinking = false
      } else if (thinking) {
        ai.value.reasoning += chunk.choices[0].delta.content || ''
      } else {
        ai.value.content += chunk.choices[0].delta.content || ''
      }

      // This code starts speaking the sentences before the full response
      // is retrieved. Actually it does so after the first sentence is retrieved.
      const currentLength = (spoken.value.join('') + sentences.value.join('')).length
      const idx = ai.value.content.indexOf(END_SENTENCE_TOKEN, currentLength)
      if (idx > 0) {
        sentences.value.push(ai.value.content.slice(currentLength, currentLength + idx + END_SENTENCE_TOKEN.length))
        if (!window.speechSynthesis.speaking) speak()
      }
    }

    // this checks if the model issued END_SENTENCE_TOKEN and if not it treats the whole thing
    // as the content to say. Works only up to 32k characters.
    if (ai.value.content.indexOf(END_SENTENCE_TOKEN) === -1) {
      sentences.value.push(ai.value.content)
      speak()
    }

    while (window.speechSynthesis.speaking) {
      await sleep(1)
    }
  } finally {
    processing.value = false
  }
}

function getVoices() {
  return new Promise(resolve => {
    const synth = window.speechSynthesis;
    const id = setInterval(() => {
      if (synth.getVoices().length !== 0) {
        resolve(synth.getVoices());
        clearInterval(id);
      }
    }, 10);
  })
}

onMounted(async() => {
  voices.value = await getVoices()
  voices.value = voices.value.filter(v => v.lang.startsWith('pl'))
  voice.value = voices.value[0]?.name

  window.speechSynthesis.cancel()
})

</script>

<style lang="postcss" scoped>
label {
  display: block;
}
select, textarea {
  width: 30rem;
}

textarea {
  height: 3rem;
}

.field {
  margin-bottom: 8px;
}

.markdown-body {
  background-color: white;
  color: #666;

}
</style>
