<template>
  <div id="app">
    <form @submit.prevent="submitForm">
      <label for="instagramUsername">Username do Instagram:</label>
      <input
        type="text"
        id="instagramUsername"
        v-model="instagramUsername"
        placeholder="Insira o username do Instagram"
        required
      />
      <button type="submit" :disabled="loading">
        {{ loading ? 'Enviando...' : 'Enviar' }}
      </button>
    </form>

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
    };
  },
  methods: {
    submitForm() {
      this.loading = true;
      this.submitted = true;

      axios.post(`http://localhost:8000/prediction/${this.instagramUsername}`)
        .then(response => {
          this.prediction = response.data.prediction;
          this.status = this.prediction.status;
          console.log(this.prediction);

          if (this.status === 'completed') {
            this.loading = false;
          } else {
            this.checkStatus();
          }
        })
        .catch(error => {
          console.error('Erro no envio:', error);
          this.loading = false;
        });
    },
    checkStatus() {
      const intervalId = setInterval(() => {
        axios.get(`http://localhost:8000/prediction/${this.instagramUsername}`)
          .then(response => {
            this.status = response.data.prediction.status;
            this.prediction = response.data.prediction;

            if (this.status === 'completed') {
              clearInterval(intervalId);
              this.loading = false;
            }
          })
          .catch(error => {
            console.error('Erro ao verificar o status:', error);
            clearInterval(intervalId);
            this.loading = false;
          });
      }, 15000);
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
