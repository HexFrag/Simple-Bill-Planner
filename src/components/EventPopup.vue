<template>
    <div v-if="visible" class="popup-overlay" @click="hide">
      <div class="popup" @click.stop>
        <h5>Add New Event</h5>
        <form @submit.prevent="submit">
          <div class="form-group">
            <label for="name">Event Name</label>
            <input v-model="name" id="name" class="form-control" required />
          </div>
          <div class="form-group">
            <label for="amount">Amount Due</label>
            <input v-model="amount" id="amount" type="number" step="0.01" class="form-control" required />
          </div>
          <div class="form-group form-check">
            <input v-model="repeat" id="repeat" type="checkbox" class="form-check-input" />
            <label for="repeat" class="form-check-label">Repeat Monthly</label>
          </div>
          <button type="submit" class="btn btn-primary btn-block">Add Event</button>
        </form>
      </div>
    </div>
  </template>
  
  <script>
  export default {
    data() {
      return {
        visible: false,
        name: '',
        amount: '',
        repeat: false
      };
    },
    methods: {
      show() {
        this.visible = true;
        this.name = '';
        this.amount = '';
        this.repeat = false;
      },
      hide() {
        this.visible = false;
      },
      submit() {
        this.$emit('add-event', { name: this.name, amount: parseFloat(this.amount), repeat: this.repeat });
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
  
  .form-check-input {
    margin-right: 10px;
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
  