<template>
  <div id="app">
    <nav class="navbar navbar-dark bg-dark">
      <div class="container-fluid d-flex align-items-center justify-content-between">
        <span class="navbar-brand mb-0 h1">
          Bill Payment Planner - {{ currentMonthName }} {{ currentYear }}
        </span>
        <div class="d-flex align-items-center flex-grow-1 justify-content-center">
          <div class="mx-4 text-center">
            <strong>In:</strong> {{ formatCurrency(totalIn) }}
          </div>
          <div class="mx-4 text-center">
            <strong>Out:</strong> {{ formatCurrency(totalOut) }}
          </div>
          <div class="mx-4 text-center">
            <strong>Net:</strong> {{ formatCurrency(netTotal) }}
          </div>
        </div>
        <div class="btn-group">
          <button class="btn btn-secondary btn-sm" @click="previousMonth">&lt; Prev</button>
          <button class="btn btn-secondary btn-sm" @click="nextMonth">Next &gt;</button>
        </div>
      </div>
    </nav>
    <div class="calendar-wrapper">
      <CalendarMain
        ref="calendarMain"
        :currentMonth="currentMonth"
        :currentYear="currentYear"
        @update-totals="updateTotals"
      />
    </div>
  </div>
</template>

<script>
import CalendarMain from './components/calendar-main.vue';

export default {
  name: 'App',
  components: {
    CalendarMain
  },
  data() {
    return {
      currentYear: new Date().getFullYear(),
      currentMonth: new Date().getMonth(),
      totalIn: 0,
      totalOut: 0,
      netTotal: 0
    };
  },
  computed: {
    currentMonthName() {
      const monthNames = [
        'January', 'February', 'March', 'April', 'May', 'June',
        'July', 'August', 'September', 'October', 'November', 'December'
      ];
      return monthNames[this.currentMonth];
    }
  },
  methods: {
    previousMonth() {
      if (this.currentMonth === 0) {
        this.currentMonth = 11;
        this.currentYear--;
      } else {
        this.currentMonth--;
      }
    },
    nextMonth() {
      if (this.currentMonth === 11) {
        this.currentMonth = 0;
        this.currentYear++;
      } else {
        this.currentMonth++;
      }
    },
    formatCurrency(amount) {
      return `$${amount.toFixed(2)}`;
    },
    updateTotals(totals) {
      this.totalIn = totals.totalIn;
      this.totalOut = totals.totalOut;
      this.netTotal = totals.netTotal;
    }
  }
};
</script>

<style>
@import '~bootstrap/dist/css/bootstrap.min.css';

#app {
  height: 100vh;
  display: flex;
  flex-direction: column;
  background-color: #121212;
  color: #ffffff;
}

.navbar {
  height: 50px;
  display: flex;
  align-items: center;
  justify-content: space-between;
}

.navbar-header {
  flex-grow: 1;
}

.navbar-brand {
  margin-right: auto;
}

.d-flex.justify-content-center.flex-grow-1 {
  flex-grow: 1;
}

.btn-group {
  margin-left: auto;
}

.calendar-wrapper {
  flex: 1;
  overflow-y: auto;
}

.calendar-container {
  display: flex;
  justify-content: center;
  align-items: center;
  padding: 0 10px;
}
</style>
