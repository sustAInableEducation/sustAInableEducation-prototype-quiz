<template>
  <div class="quiz-container">
    <!-- Ladeanzeige -->
    <div v-if="isLoading" class="loading">Lädt...</div>

    <!-- Fehleranzeige -->
    <div v-if="errorWhileLoading" class="error">Fehler beim Laden der Antwort!</div>

    <!-- Nachrichten anzeigen -->
    <div v-for="(message, index) in displayedQuestions.slice(0, currentQuestionIndex)" :key="index"
      class="message-container">
      <div class="assistant-message">
        <h3 class="message-title">{{ message.quizTitle }}</h3>
        <p class="message-question">{{ message.questionTitle }}</p>
        <ul class="message-options">
          <li  v-for="(option, optionIndex) in message.options" :key="optionIndex" @click="selectAnswer(option)" :class="{ 'selected': option === selectedAnswer}">
            {{ option }}
          </li>
        </ul>

        <!-- Container für den Button zum Senden der Antwort -->
        <div v-if="selectedAnswer !== null" class="submit-button-container">
          <button @click="submitAnswer(index)" class="submit-button" :class="getClassForAnswer(index)">
            Antwort überprüfen
          </button>
        </div>
        <p v-if="displayedQuestions[index].solved !== 0" 
          :class="{'correct': displayedQuestions[index].solved === 1, 'wrong': displayedQuestions[index].solved === -1}">
          {{ displayedQuestions[index].solved === 1 ? 'Richtig' : 'Falsch' }}
        </p>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">

import { onMounted, ref, computed, watch, nextTick } from 'vue';
import axios from 'axios';
import type { Message } from '../types/message';
import type { DisplayedQuestion } from '@/types/displayedQuestion';
import { quizzes } from '@/data/quizzes';


const errorWhileLoading = ref<boolean>(false);
const messages = ref<Message[]>([]);
const quizTitle = ref<string>('');
const selectedAnswer = ref<(string)>(null);




const currentQuestionIndex = ref<number>(1);
const displayedQuestions = computed<DisplayedQuestion[]>(() => {
  const filteredMessages = messages.value.filter(
    message => message.role === 'assistant' && JSON.parse(message.content).story === undefined,
  );
  const questions: DisplayedQuestion[] = [];
  filteredMessages.forEach(message => {
    const content = JSON.parse(message.content);
    content.questionarray.forEach((el: any) => {
      questions.push({
        quizTitle: content.title,
        questionTitle: el.question,
        options: el.options,
        correctAnswer: el.correctAnswer,
        answered: false,
        solved: 0,
      });
    });
  });
  return questions;
});

const isLoading = ref<boolean>(false);

function loadMessages() {
  if (localStorage.messages) {
    return JSON.parse(localStorage.messages);
  }
  return []; // Fallback, falls keine Nachrichten gefunden werden
}

watch(displayedQuestions, () => {
  nextTick(() => {
    window.scrollTo({ top: document.body.scrollHeight, behavior: 'smooth' });
  });
});

const generateNextQuestion = (userMessage: Message) => {
  isLoading.value = true;
  messages.value.push(userMessage);

  const data = {
    model: 'meta-llama/Llama-3.3-70B-Instruct',
    messages: messages.value,
    response_format: { type: 'json_object' },
  };

  const config = {
    method: 'post',
    maxBodyLength: Infinity,
    url: import.meta.env.VITE_API_URL,
    headers: {
      'Content-Type': 'application/json',
      Authorization: `Bearer ${import.meta.env.VITE_API_KEY}`,
    },
    data: data,
  };

  axios
    .request(config)
    .then(response => {
      const assistantMessage = response.data.choices[0].message;
      assistantMessage.content = assistantMessage.content.replace(/(\r\n|\n|\r)/gm, '');
      messages.value.push(assistantMessage);
      const content = JSON.parse(assistantMessage.content);
      quizTitle.value = content.title;
    })
    .catch(error => {
      console.error(error);
      errorWhileLoading.value = true;
    })
    .finally(() => {
      isLoading.value = false;
    });
};

const startQuiz = (systemMessage: Message) => {
  isLoading.value = true;
  messages.value.push(systemMessage);
  generateNextQuestion({
    role: 'user',
    content: quizzes.userPrompt,
  });
};

const selectAnswer = (option: string) => {
  selectedAnswer.value = option;
};

const checkAnswer = (index: number) => {
  return displayedQuestions.value[index].correctAnswer === selectedAnswer.value;
};

const getClassForAnswer = (index: number) => {
  const isCorrect = checkAnswer(index);
  return isCorrect ? 'correct' : 'wrong';
};

const submitAnswer = (questionIndex: number) => {
  const result = checkAnswer(questionIndex);

  if (selectedAnswer.value !== null) {
     // alert(`Antwort ${result ? 'korrekt' : 'falsch'}`);

    if (result === true) {
      displayedQuestions.value[questionIndex].solved = 1;

    }

    else {
      displayedQuestions.value[questionIndex].solved = -1;

    }

    currentQuestionIndex.value++;
  } else {
    alert('Bitte eine Antwort auswählen!');
  }
};

onMounted(() => {
  messages.value = loadMessages();
  startQuiz({
    role: 'system',
    content: quizzes.systemPrompt,
  });
});
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

.loading,
.error {
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

.correct {
  background-color: green;
}

.wrong {
  background-color: #e74c3c;
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



.selected {
  background-color: #2f87a0!important;
  color: rgb(255, 255, 255);
  
}


.submit-button-container {
  text-align: center;
  margin-top: 10px;
}

.submit-button {
  background-color: #2C692F;
  color: white;
  border: none;
  padding: 10px 20px;
  border-radius: 5px;
  cursor: pointer;
  font-size: 1.2em;
}

.submit-button:hover {
  background-color: #1d4e1c;
}
</style>