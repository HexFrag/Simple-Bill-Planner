<template>
    <div v-if="visible" class="popup-overlay" @click="hide">
      <div class="popup" @click.stop>
        <h5>Set Payday</h5>
        <form @submit.prevent="submit">
          <div class="form-group">
            <label for="date">Payday Start Date</label>
            <input v-model="date" id="date" type="date" class="form-control" required />
          </div>
          <div class="form-group">
            <label for="amount">Pay Amount</label>
            <input v-model="amount" id="amount" type="number" step="0.01" class="form-control" required />
          </div>
          <button type="submit" class="btn btn-primary btn-block">Set Payday</button>
        </form>
      </div>
    </div>
  </template>
  
  <script>
  export default {
    data() {
      return {
        visible: false,
        date: '',
        amount: ''
      };
    },
    methods: {
      show() {
        this.visible = true;
        this.date = '';
        this.amount = '';
      },
      hide() {
        this.visible = false;
      },
      submit() {
        this.$emit('set-payday', { date: this.date, amount: parseFloat(this.amount) });
        this.hide();
      }
    }
  };
  </script>
  
  <style scoped>
  .popup-overlay {
    position: fixed;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background: rgba(0, 0, 0, 0.5);
    display: flex;
    justify-content: center;
    align-items: center;
    z-index: 1000;
  }
  
  .popup {
    background: #333;
    color: white;
    padding: 20px;
    border-radius: 8px;
    box-shadow: 0 0 10px rgba(0, 0, 0, 0.5);
  }
  
  .form-group {
    margin-bottom: 15px;
  }
  
  .form-control {
    background: #444;
    border: 1px solid #555;
    color: white;
  }
  
  .btn-primary {
    background-color: #007bff;
    border-color: #007bff;
  }
  </style>
  