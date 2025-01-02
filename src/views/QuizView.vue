<template>
  <div class="quiz-container">
    <!-- Ladeanzeige -->
    <div v-if="isLoading" class="loading">Lädt...</div>
    
    <!-- Fehleranzeige -->
    <div v-if="errorWhileLoading" class="error">Fehler beim Laden der Antwort!</div>
    
    <!-- Nachrichten anzeigen -->
    <div v-for="(message, index) in displayedQuestions.slice(0, currentQuestionIndex)" :key="index" class="message-container">
      <div class="assistant-message">
        <h3 class="message-title">{{ message.quizTitle }}</h3>
        <p class="message-question">{{ message.questionTitle }}</p>
        <ul class="message-options">
          <li 
            v-for="(option, optionIndex) in message.options" 
            :key="optionIndex" 
            @click="selectAnswer(option, index, optionIndex)" 
            :class="{'selected': selectedAnswers[index] === optionIndex}"
          >
            {{ option }}
          </li>
        </ul>

        <!-- Container für den Button zum Senden der Antwort -->
        <div v-if="selectedAnswers[index] !== null" class="submit-button-container">
          <button @click="submitAnswer(index)" class="submit-button">Antwort überprüfen</button>
        </div>
      </div>
      
    </div>
  </div>
</template>

<script setup lang="ts">
import { onMounted, ref, computed, watch, nextTick } from 'vue'
import axios from 'axios'
import type { Message } from '../types/message'
import type { DisplayedQuestion } from '@/types/displayedQuestion'
import { quizzes } from '@/data/quizzes'

const errorWhileLoading = ref<boolean>(false)
const messages = ref<Message[]>([])
const quizTitle = ref<string>('')
const selectedAnswers = ref<(number | null)[]>([])  // Array, um die ausgewählten Antworten zu verfolgen
const correctAnswers: number[] = []

const currentQuestionIndex = ref<number>(1)
const displayedQuestions = computed<DisplayedQuestion[]>(() => {
  const filteredMessages = messages.value.filter(
    message => message.role === 'assistant' && JSON.parse(message.content).story === undefined,
  )
  const questions: DisplayedQuestion[] = []
  filteredMessages.forEach(message => {
    const content = JSON.parse(message.content)
    content.questionarray.forEach((el: any) => {
      questions.push({
        quizTitle: content.title,
        questionTitle: el.question,
        options: el.options,
        correctAnswer: el.correctAnswer,
        answered: false,
        solved: 0
      })
    })
  })
  return questions
})

const isLoading = ref<boolean>(false)

function loadMessages() {
  if (localStorage.messages) {
    return JSON.parse(localStorage.messages)
  }
  return [] // Fallback, falls keine Nachrichten gefunden werden
}

watch(displayedQuestions, () => {
  nextTick(() => {
    window.scrollTo({ top: document.body.scrollHeight, behavior: 'smooth' })
  })
})

const generateNextQuestion = (userMessage: Message) => {
  isLoading.value = true
  messages.value.push(userMessage)

  const data = {
    model: 'meta-llama/Llama-3.3-70B-Instruct',
    messages: messages.value,
    response_format: { type: 'json_object' },
  }

  const config = {
    method: 'post',
    maxBodyLength: Infinity,
    url: import.meta.env.VITE_API_URL,
    headers: {
      'Content-Type': 'application/json',
      Authorization: `Bearer ${import.meta.env.VITE_API_KEY}`,
    },
    data: data,
  }

  axios
    .request(config)
    .then(response => {
      const assistantMessage = response.data.choices[0].message
      assistantMessage.content = assistantMessage.content.replace(/(\r\n|\n|\r)/gm, '')
      messages.value.push(assistantMessage)
      const content = JSON.parse(assistantMessage.content)
      correctAnswers.push(content.rightAnswerindex)
      quizTitle.value = content.title
    })
    .catch(error => {
      console.error(error)
      errorWhileLoading.value = true
    })
    .finally(() => {
      isLoading.value = false
    })
}

const startQuiz = (systemMessage: Message) => {
  isLoading.value = true
  messages.value.push(systemMessage)
  generateNextQuestion({
    role: 'user',
    content: quizzes.userPrompt,
  })
}

const selectAnswer = (option: string, questionIndex: number, optionIndex: number) => {
  selectedAnswers.value[questionIndex] = optionIndex
}

const submitAnswer = (questionIndex: number) => {
  const selectedAnswerIndex = selectedAnswers.value[questionIndex]

  if (selectedAnswerIndex !== null) {
    const checkAnswer = correctAnswers[questionIndex] === selectedAnswerIndex
    alert(`Antwort ${checkAnswer ? 'korrekt' : 'falsch'}`)

    
    if (checkAnswer == true) {
      displayedQuestions.value[questionIndex].solved = 1
    }

    if (checkAnswer == false) {
      displayedQuestions.value[questionIndex].solved = -1
    }

    console.log(currentQuestionIndex.value)
   currentQuestionIndex.value++;
  // nextTick()
    return checkAnswer
  } else {
    alert('Bitte eine Antwort auswählen!')
  }
}

onMounted(() => {
  messages.value = loadMessages()
  startQuiz({
    role: 'system',
    content: quizzes.systemPrompt,
  })
})
</script>

<style scoped>
.quiz-container {
  padding: 20px;
  background-color: #f4f7fa;
  border-radius: 8px;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
  max-width: 800px;
  margin: 0 auto;
  color: #333;
}

.loading, .error {
  text-align: center;
  font-size: 1.2em;
  padding: 15px;
  border-radius: 5px;
  margin-bottom: 20px;
}

.loading {
  background-color: #2C692F;
  color: #fff;
}

.error {
  background-color: #e74c3c;
  color: #fff;
}

.message-container {
  margin-bottom: 20px;
}

.assistant-message {
  background-color: #fff;
  border-radius: 8px;
  padding: 15px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.message-title {
  font-size: 1.4em;
  font-weight: bold;
  margin-bottom: 10px;
}

.message-question {
  font-size: 1.1em;
  margin-bottom: 10px;
}

.message-options {
  list-style-type: none;
  padding: 0;
  margin: 0;
}

.message-options li {
  background-color: #ecf0f1;
  padding: 10px;
  border-radius: 5px;
  margin-bottom: 8px;
  font-size: 1em;
  cursor: pointer;
}

.message-options li.selected {
  background-color: #276677;   /* 45A249 */
  color: white;
}

.submit-button-container {
  text-align: center; /* Zentriert den Button horizontal */
  margin-top: 10px; /* Optional: Abstand nach oben */
}

.submit-button {
  background-color: #38b1d3;
  color: white;
  border: none;
  padding: 10px 20px;
  border-radius: 5px;
  cursor: pointer;
  font-size: 1.2em;
  width: auto; /* Stellt sicher, dass der Button nur so breit wie nötig ist */
  display: inline-block; /* Macht den Button zu einem Inline-Block, damit er zentriert wird */
}

.submit-button:hover {
  background-color: #223C74;
}
</style>