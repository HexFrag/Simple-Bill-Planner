<template>
  <div id="app">
    <nav class="navbar navbar-dark bg-dark">
      <div class="container-fluid d-flex align-items-center justify-content-between">
        <span class="navbar-brand mb-0 h1">
          Bill Payment Planner - Week of {{ formatDate(currentWeekStart) }}
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
        <button class="btn btn-secondary btn-sm" @click="goToToday">Today</button>
      </div>
    </nav>
    <div class="calendar-wrapper" @wheel="handleScroll">
      <CalendarMain
        ref="calendarMain"
        :currentWeekStart="currentWeekStart"
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
      today: new Date(),
      totalIn: 0,
      totalOut: 0,
      netTotal: 0,
      accumulatedScroll: 0,
      scrollThreshold: 100, // Adjust this value to control sensitivity
      weekOffset: 0, // New property to keep track of week offset
      startWeek: this.getStartOfWeek(new Date()) // Start from the current week
    };
  },
  computed: {
    currentWeekStart() {
      const date = new Date(this.startWeek);
      date.setDate(this.startWeek.getDate() + this.weekOffset * 7);
      return date;
    }
  },
  methods: {
    getStartOfWeek(date) {
      const day = date.getDay();
      const diff = date.getDate() - day + (day === 0 ? -6 : 1); // Adjust when day is Sunday
      return new Date(date.setDate(diff));
    },
    goToToday() {
      this.weekOffset = 0;
      this.startWeek = this.getStartOfWeek(new Date());
    },
    formatCurrency(amount) {
      return `$${amount ? amount.toFixed(2) : '0.00'}`;
    },
    formatDate(date) {
      if (!date) return '';
      return new Date(date).toLocaleDateString('en-US', { month: 'long', day: 'numeric', year: 'numeric' });
    },
    updateTotals(totals) {
      this.totalIn = totals.totalIn || 0;
      this.totalOut = totals.totalOut || 0;
      this.netTotal = totals.netTotal || 0;
    },
    handleScroll(event) {
      this.accumulatedScroll += event.deltaY;

      if (this.accumulatedScroll >= this.scrollThreshold) {
        this.nextWeek();
        this.accumulatedScroll = 0;
      } else if (this.accumulatedScroll <= -this.scrollThreshold) {
        this.previousWeek();
        this.accumulatedScroll = 0;
      }
    },
    previousWeek() {
      this.weekOffset -= 1;
    },
    nextWeek() {
      this.weekOffset += 1;
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

.calendar-wrapper {
  flex: 1;
  overflow-y: hidden; /* Hide the scrollbar */
  -ms-overflow-style: none;  /* IE and Edge */
  scrollbar-width: none;  /* Firefox */
}

.calendar-wrapper::-webkit-scrollbar {
  display: none;  /* Chrome, Safari, Opera */
}

.calendar-container {
  display: flex;
  justify-content: center;
  align-items: center;
  padding: 0 10px;
}
</style>
