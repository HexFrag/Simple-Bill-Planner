<template>
  <div class="calendar-container container-fluid p-0">
    <EventPopup ref="eventPopup" @add-event="addEvent" />
    <PaydayPopup ref="paydayPopup" @set-payday="setPayday" />
    <div class="calendar">
      <div class="day-header" v-for="day in weekDays" :key="day">{{ day }}</div>
      <div
        v-for="(day, index) in visibleDays"
        :key="index"
        class="day"
        :class="[
          'day',
          { 'current-day': isCurrentDay(day.fullDate) },
          { 'dark-month': isDarkMonth(day.fullDate) },
          { 'light-month': isLightMonth(day.fullDate) }
        ]"
      >
        <div v-if="day.date" class="d-flex justify-content-between align-items-start">
          <span class="day-number">{{ day.date }}</span>
          <button class="btn btn-sm btn-success" @click="showEventPopup(day.fullDate)">+</button>
        </div>
        <div v-if="isPayday(day.fullDate)" class="payday d-flex justify-content-between align-items-center">
          <div><small>Payday</small></div>
          <div><small>{{ formatCurrency(payday.amount) }}</small></div>
        </div>
        <div
          v-for="event in day.events"
          :key="event.id"
          class="event d-flex justify-content-between align-items-center"
          :class="{ 'paid': event.paid }"
        >
          <div><small>{{ event.name }}</small></div>
          <div><small>{{ formatCurrency(event.amount) }}</small></div>
          <div>
            <button class="btn btn-xs btn-primary paid-btn" @click="markAsPaid(event.id)">P</button>
            <button class="btn btn-xs btn-danger delete-btn" @click="removeEvent(event.id)">x</button>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import EventPopup from './EventPopup.vue';
import PaydayPopup from './PaydayPopup.vue';

export default {
  props: ['currentWeekStart'],
  components: { EventPopup, PaydayPopup },
  data() {
    return {
      weekDays: ['Sun', 'Mon', 'Tue', 'Wed', 'Thu', 'Fri', 'Sat'],
      events: JSON.parse(localStorage.getItem('events') || '{}'),
      payday: JSON.parse(localStorage.getItem('payday') || 'null'),
      selectedDate: null,
      today: new Date(),
      totalIn: 0,
      totalOut: 0,
      netTotal: 0
    };
  },
  mounted() {
    if (!this.payday) {
      this.$refs.paydayPopup.show();
    }
    this.updateEvents();
  },
  computed: {
    visibleDays() {
      const days = [];
      const start = new Date(this.currentWeekStart);
      const dayOffset = start.getDay();

      // Include 2 weeks before and 3 weeks after the current week to center it in the view
      const startDate = new Date(start);
      startDate.setDate(startDate.getDate() - (dayOffset + 14)); // 14 days (2 weeks) before the current week

      for (let i = 0; i < 42; i++) { // 42 days to display 6 weeks
        const date = new Date(startDate);
        date.setDate(startDate.getDate() + i);
        days.push({
          date: date.getDate(),
          fullDate: date,
          events: this.getEventsForDate(date)
        });
      }

      return days;
    }
  },
  methods: {
    showEventPopup(date) {
      this.selectedDate = date;
      this.$refs.eventPopup.show();
    },
    addEvent(event) {
      const id = Date.now();
      const newEvent = { id, name: event.name, amount: event.amount, repeat: event.repeat, paid: false };
      this.addEventToDate(newEvent, this.selectedDate);
      if (event.repeat) {
        this.addRepeatingEvent(newEvent);
      }
      this.saveEvents();
      this.updateEvents();
    },
    addEventToDate(event, date) {
      const dateString = `${date.getFullYear()}-${date.getMonth() + 1}-${date.getDate()}`;
      if (!this.events[dateString]) {
        this.events[dateString] = [];
      }
      this.events[dateString].push(event);
    },
    addRepeatingEvent(event) {
      const originalDate = new Date(this.selectedDate);
      for (let i = 1; i <= 12; i++) {
        const date = new Date(originalDate);
        date.setMonth(date.getMonth() + i);
        const dateString = `${date.getFullYear()}-${date.getMonth() + 1}-${date.getDate()}`;
        if (!this.events[dateString]) {
          this.events[dateString] = [];
        }
        if (!this.events[dateString].some(e => e.id === event.id)) {
          this.events[dateString].push({ ...event, id: `${event.id}-${i}` });
        }
      }
    },
    removeEvent(eventId) {
      const baseEventId = eventId.toString().split('-')[0];
      for (const key in this.events) {
        if (Object.prototype.hasOwnProperty.call(this.events, key)) {
          this.events[key] = this.events[key].filter(event => !event.id.toString().startsWith(baseEventId));
          if (this.events[key].length === 0) {
            delete this.events[key];
          }
        }
      }
      this.saveEvents();
      this.updateEvents();
    },
    markAsPaid(eventId) {
      for (const key in this.events) {
        if (Object.prototype.hasOwnProperty.call(this.events, key)) {
          this.events[key].forEach(event => {
            if (event.id === eventId) {
              event.paid = !event.paid;
            }
          });
        }
      }
      this.saveEvents();
      this.updateEvents();
    },
    saveEvents() {
      localStorage.setItem('events', JSON.stringify(this.events));
    },
    updateEvents() {
      this.events = JSON.parse(localStorage.getItem('events') || '{}');
      this.updateTotals();
    },
    setPayday(payday) {
      this.payday = payday;
      localStorage.setItem('payday', JSON.stringify(payday));
      this.updateTotals();
    },
    isPayday(date) {
      if (!this.payday) return false;
      const paydayDate = new Date(this.payday.date);
      const diffInDays = Math.floor((date - paydayDate) / (1000 * 60 * 60 * 24));
      return diffInDays % 14 === 0;
    },
    formatCurrency(amount) {
      return `$${amount.toFixed(2)}`;
    },
    getEventsForDate(date) {
      const day = date.getDate();
      const month = date.getMonth() + 1;
      const year = date.getFullYear();
      const dateString = `${year}-${month}-${day}`;
      let events = this.events[dateString] || [];

      // Add repeating events
      for (const key in this.events) {
        if (Object.prototype.hasOwnProperty.call(this.events, key)) {
          this.events[key].forEach(event => {
            if (event.repeat) {
              const eventDate = new Date(key);
              if (eventDate.getDate() === day && eventDate.getMonth() === month - 1 && eventDate.getFullYear() === year) {
                const repeatDate = new Date(eventDate);
                for (let i = 1; i <= 12; i++) {
                  repeatDate.setMonth(repeatDate.getMonth() + 1);
                  if (repeatDate.getDate() === date.getDate() && repeatDate.getMonth() === date.getMonth() && repeatDate.getFullYear() === date.getFullYear()) {
                    events.push(event);
                    break;
                  }
                }
              }
            }
          });
        }
      }

      return events;
    },
    updateTotals() {
      const firstDayOfMonth = new Date(this.currentWeekStart.getFullYear(), this.currentWeekStart.getMonth(), 1);
      const lastDayOfMonth = new Date(this.currentWeekStart.getFullYear(), this.currentWeekStart.getMonth() + 1, 0);
      let totalIn = 0;
      let totalOut = 0;

      for (let d = new Date(firstDayOfMonth); d <= lastDayOfMonth; d.setDate(d.getDate() + 1)) {
        const dateString = `${d.getFullYear()}-${d.getMonth() + 1}-${d.getDate()}`;
        if (this.events[dateString]) {
          this.events[dateString].forEach(event => {
            totalOut += event.amount;
          });
        }
        if (this.isPayday(d)) {
          totalIn += this.payday.amount;
        }
      }

      this.totalIn = totalIn;
      this.totalOut = totalOut;
      this.netTotal = totalIn - totalOut;

      this.$emit('update-totals', {
        totalIn: this.totalIn,
        totalOut: this.totalOut,
        netTotal: this.netTotal
      });
    },
    isCurrentDay(date) {
      const today = new Date();
      return date.getDate() === today.getDate() && date.getMonth() === today.getMonth() && date.getFullYear() === today.getFullYear();
    },
    isDarkMonth(date) {
      return date.getMonth() % 2 === 0;
    },
    isLightMonth(date) {
      return date.getMonth() % 2 !== 0;
    }
  },
  watch: {
    currentWeekStart() {
      this.updateEvents();
    },
    visibleDays() {
      this.updateTotals();
    }
  }
};
</script>

<style scoped>
.calendar-container {
  height: calc(100vh - 50px);
  display: flex;
  justify-content: center;
  align-items: flex-start;
  overflow-y: hidden; /* Hide the scrollbar */
  padding: 0 10px;
  background-color: #121212;
  color: #ffffff;
}

.calendar-container::-webkit-scrollbar {
  display: none;  /* Chrome, Safari, Opera */
}

.calendar {
  display: grid;
  grid-template-columns: repeat(7, 1fr);
  grid-template-rows: auto;
  gap: 10px;
  width: 100%;
  max-width: 100%;
}

.day-header {
  font-weight: bold;
  text-align: center;
  padding: 5px;
  background-color: #282828;
  border: 1px solid #333;
  font-size: 0.8rem;
  position: sticky;
  top: 0;
  z-index: 10;
}

.day {
  border: 1px solid #333;
  padding: 5px;
  min-height: calc((100vh - 100px) / 6);
  font-size: 0.9rem;
}

.day.current-day {
  background: linear-gradient(135deg, rgba(255,255,255,0.2), rgba(255,255,255,0.1));
}

.day.dark-month {
  background-color: #1e1e1e;
}

.day.light-month {
  background-color: #2a2a2a;
}

.day-number {
  background-color: #333;
  padding: 2px 5px;
  border-radius: 3px;
  font-size: 0.9rem;
}

.event {
  background-color: #00849b;
  color: white;
  padding: 2px 5px;
  margin-top: 5px;
  border-radius: 5px;
  font-size: 0.8rem;
  display: flex;
  justify-content: space-between;
}
.payday {
  background-color: #0ea501;
  color: white;
  padding: 2px 5px;
  margin-top: 5px;
  border-radius: 5px;
  font-size: 0.8rem;
  display: flex;
  justify-content: space-between;
}

.paid {
  background-color: #6c757d !important;
}

.paid-btn,
.delete-btn {
  font-size: 0.7rem;
  padding: 0 5px;
  margin-left: 5px;
  margin-right: 0; /* Ensure buttons are closely aligned */
}

.delete-btn {
  margin-right: 5px; /* Add spacing to the delete button for consistent look */
}
</style>
