<script setup lang="ts">
import { onMounted, ref, computed, watch, nextTick } from 'vue'
import axios from 'axios'
import type { Story } from '../types/story'
import type { Message } from '../types/message'
import type { DisplayedMessage } from '@/types/displayedMessage'
import { stories } from '@/data/stories'

const props = defineProps({
  id: Number,
})
const selectedStory = ref<Story>({} as Story)
const errorWhileGenerating = ref<boolean>(false)
const messages = ref<Message[]>([])
const storyTitle = ref<string>('')
const selectedOptions = ref<boolean[]>([])
const displayedMessages = computed<DisplayedMessage[]>(() => {
  const filteredMessages = messages.value.filter(
    message => message.role === 'assistant',
  )
  const displayedMessages: DisplayedMessage[] = []
  filteredMessages.forEach(message => {
    const content = JSON.parse(message.content)
    displayedMessages.push({
      title: content.title,
      story: content.story,
      options: content.options,
    })
  })
  return displayedMessages
})
const isGenerating = ref<boolean>(false)

watch(displayedMessages, () => {
  nextTick(() => {
    window.scrollTo({ top: document.body.scrollHeight, behavior: 'smooth' })
  })
})
function saveMesssages(){
  localStorage.messages = JSON.stringify(messages.value)
}

const generateNextPart = (userMessage: Message) => {
  isGenerating.value = true
  messages.value.push(userMessage)
  const data = {
    model: 'meta-llama/Llama-3.3-70B-Instruct',
    messages: messages.value,
    response_format: {"type":"json_object"}
  }
  const config = {
    method: 'post',
    maxBodyLength: Infinity,
    url: import.meta.env.VITE_API_URL,
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${import.meta.env.VITE_API_KEY}`
    },
    data: data,
  }

  axios
    .request(config)
    .then(response => {
      const assistantMessage = response.data.choices[0].message
      assistantMessage.content = assistantMessage.content.replace(/(\r\n|\n|\r)/gm, '')
      messages.value.push(assistantMessage)
      storyTitle.value = JSON.parse(assistantMessage.content).title
    })
    .catch(error => {
      console.log(error)
      errorWhileGenerating.value = true
    })
    .finally(() => {
      isGenerating.value = false
    })
}

const startStory = (systemMessage: Message) => {
  isGenerating.value = true
  messages.value.push(systemMessage)
  generateNextPart({
    role: 'user',
    content: selectedStory.value.userPrompt,
  })
}

const chooseOption = (messageIndex: number, userMessage: Message) => {
  selectedOptions.value[messageIndex] = true
  generateNextPart(userMessage)
}

onMounted(() => {
  if (
    props.id === undefined ||
    props.id < 0 ||
    props.id >= stories.length ||
    !stories[props.id]
  ) {
    errorWhileGenerating.value = true
  } else {
    selectedStory.value = stories[props.id]
    startStory({
      role: 'system',
      content: selectedStory.value?.systemPrompt,
    })
  }
})
</script>

<template>
  <div class="background"></div>
  <div
    v-if="errorWhileGenerating"
    class="pt-6 max-w-screen-lg flex flex-col items-center mx-auto gap-12 text-center"
  >
    <Message severity="error" class="text-center">
      <p class="text-xl">Es ist ein Fehler aufgetreten!</p>
    </Message>
    <Button
      label="Zurück zur Startseite"
      @click="$router.push({ name: 'index' })"
    />
  </div>
  <div
    v-if="!errorWhileGenerating && storyTitle !== ''"
    class="pt-6 max-w-screen-lg flex flex-col items-center mx-auto gap-4"
  >
    <h1 class="text-7xl font-bold text-center pb-2">{{ storyTitle }}</h1>
    <div
      v-for="(message, messageIndex) in displayedMessages"
      :key="messageIndex"
    >
      <div v-if="messageIndex <= 0">
        <p>{{ message.story }}</p>
        <p class="font-bold pt-2">Entscheidungspunkt {{ messageIndex + 1 }}:</p>
        <ul class="list-decimal pl-8">
          <li v-for="option in message.options" :key="option">
            {{ option }}
          </li>
        </ul>
        <div class="flex flex-row justify-center gap-12 pt-4 pb-6">
          <Button
            v-for="(option, optionIndex) in message.options"
            :key="optionIndex"
            :label="`Option ${optionIndex + 1}`"
            class="w-64"
            :disabled="selectedOptions[messageIndex]"
            @click="
              chooseOption(messageIndex, {
                role: 'user',
                content: `Option ${optionIndex + 1}`,
              })
            "
          />
        </div>
      </div>
      <div v-else class="pb-8">
        <p>{{ message.story }}</p>
        <div class="flex flex-row justify-center gap-12 pt-4">
          <Button
            label="Weiter zum Quiz"
            class="w-64"
            @click="
              saveMesssages();
              $router.push({
                name: 'quiz',
              });
            "
          />
        </div>
      </div>
    </div>
    <div v-if="isGenerating" class="flex flex-row justify-center gap-12 pt-4 pb-6">
      <ProgressSpinner />
    </div>
  </div>
</template>
