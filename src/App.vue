<template>
  <div id="app">
    <form @submit.prevent="submitForm" v-if="!hasPendingPrediction">
      <label for="instagramUsername">Username do Instagram:</label>
      <input
        type="text"
        id="instagramUsername"
        v-model="instagramUsername"
        placeholder="Insira o username do Instagram"
        required
      />
      <button type="submit" :disabled="loading || status === 'processing'">
        {{ loading && status === 'pending' ? 'Enviando...' : (status === 'processing' ? 'Processando...' : 'Enviar') }}
      </button>
    </form>

    <div v-else>
      <p v-if="resumingOperation">Retomando operação...</p>
      <p v-else-if="status === 'failed'">A previsão falhou. Por favor, revise seus dados e tente novamente.</p>
      <p v-else>Você tem uma operação pendente. Deseja retomar?</p>
      <button @click="resumePrediction" :disabled="resumingOperation">{{ loading && status === 'pending' ? 'Enviando...' : (status === 'processing' ? 'Processando...' : 'Retomar Operação') }}</button>
    </div>

    <div v-if="submitted && status === 'completed' && !loading">
      <p>Username do Instagram enviado: {{ instagramUsername }}</p>
      <hr />

      <div class="products-container">
        <div v-for="product in prediction.products" :key="product.id" class="product">
          <img :src="product.image" alt="Product Image" />
          <p>{{ product.name }}</p>
          <p>Rating: {{ product.rating }}</p>
          <p>No. of Reviews: {{ product.no_of_reviews }}</p>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import axios from 'axios';

export default {
  data() {
    return {
      instagramUsername: '',
      submitted: false,
      loading: false,
      status: 'pending',
      prediction: null,
      hasPendingPrediction: false,
      resumingOperation: false
    };
  },
  mounted() {
    this.checkPendingPrediction();
  },
  methods: {
    submitForm() {
      localStorage.setItem('pendingData', JSON.stringify({ instagramUsername: this.instagramUsername }));
      window.addEventListener('beforeunload', this.beforeUnloadHandler);

      this.loading = true;
      this.submitted = true;

      axios.post(`${process.env.VUE_APP_BASE_URL}/prediction/${this.instagramUsername}`)
        .then(response => {
          this.prediction = response.data.prediction;
          this.status = this.prediction.status;

          if (this.status === 'completed') {
            localStorage.removeItem('pendingData');
            window.removeEventListener('beforeunload', this.beforeUnloadHandler);
            this.resumingOperation = false;
            this.loading = false;
          } else if (this.status === 'failed') {
            this.resumingOperation = false;
            localStorage.removeItem('pendingData');
            this.loading = false;
            alert('A previsão falhou. Por favor, revise seus dados e tente novamente.');
          } else {
            this.checkStatus();
          }
        })
        .catch(error => {
          console.error('Erro no envio:', error);
          this.loading = false;
          window.removeEventListener('beforeunload', this.beforeUnloadHandler);
          localStorage.removeItem('pendingData');
        });
    },
    checkStatus() {
      const intervalId = setInterval(() => {
        axios.get(`${process.env.VUE_APP_BASE_URL}/prediction/${this.instagramUsername}`)
          .then(response => {
            this.status = response.data.prediction.status;
            this.prediction = response.data.prediction;

            if (this.status === 'completed') {
              clearInterval(intervalId);
              localStorage.removeItem('pendingData');
              this.resumingOperation = false;
              this.loading = false;
            } else if (this.status === 'failed') {
              localStorage.removeItem('pendingData');
              alert('A previsão falhou. Por favor, revise seus dados e tente novamente.');
              this.resumingOperation = false;
              this.loading = false;
            }
          })
          .catch(error => {
            console.error('Erro ao verificar o status:', error);
            localStorage.removeItem('pendingData');
            clearInterval(intervalId);
            this.resumingOperation = false;
            this.loading = false;
          });
      }, 15000);
    },
    checkPendingPrediction() {
      const pendingData = localStorage.getItem('pendingData');
      if (pendingData) {
        this.hasPendingPrediction = true;
      }
    },
    resumePrediction() {
      this.resumingOperation = true;
      const pendingData = JSON.parse(localStorage.getItem('pendingData'));
      if (pendingData) {
        this.instagramUsername = pendingData.instagramUsername;
        this.submitForm();
      }
    },
    beforeUnloadHandler(event) {
      const confirmationMessage = 'Você tem certeza que deseja sair? Se você sair agora, os dados não processados podem ser perdidos.';
      event.returnValue = confirmationMessage;
      return confirmationMessage;
    },
  },
};
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}

form {
  display: flex;
  flex-direction: column;
  max-width: 300px;
  margin: 0 auto;
}

label {
  margin-bottom: 5px;
}

input {
  margin-bottom: 15px;
  padding: 8px;
}

button {
  padding: 10px;
  background-color: #3498db;
  color: #fff;
  border: none;
  cursor: pointer;
}

button:disabled {
  background-color: #bdc3c7;
  cursor: not-allowed;
}

button:hover {
  background-color: #2980b9;
}

.products-container {
  display: flex;
  justify-content: space-around;
  margin-top: 20px;
}

.product {
  text-align: center;
  max-width: 200px;
}

.product img {
  max-width: 100%;
  height: auto;
  margin-bottom: 10px;
}
</style>
