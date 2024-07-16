<template>
  <div class="calendar-container container-fluid p-0">
    <EventPopup ref="eventPopup" @add-event="addEvent" />
    <PaydayPopup ref="paydayPopup" @set-payday="setPayday" />
    <div class="calendar">
      <div class="day-header" v-for="day in weekDays" :key="day">{{ day }}</div>
      <div v-for="(day, index) in daysInMonth" :key="index" class="day" :class="{ 'prev-month': day.isPrev, 'next-month': day.isNext, 'current-day': isCurrentDay(day.fullDate) }">
        <div v-if="day.date" class="d-flex justify-content-between align-items-start">
          <span class="day-number">{{ day.date }}</span>
          <span class="remaining-balance">{{ formatCurrency(remainingBalance[`${day.fullDate.getFullYear()}-${day.fullDate.getMonth() + 1}-${day.fullDate.getDate()}`]) }}</span>
          <button class="btn btn-sm btn-success" @click="showEventPopup(day.date, day.isPrev, day.isNext)">+</button>
        </div>
        <div v-if="isPayday(day.fullDate)" class="payday d-flex justify-content-between align-items-center">
          <div><small>Payday</small></div>
          <div><small>{{ formatCurrency(payday.amount) }}</small></div>
        </div>
        <div v-for="event in day.events" :key="event.id" class="event d-flex justify-content-between align-items-center" :class="{ 'paid': event.paid }">
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
  props: ['currentMonth', 'currentYear'],
  components: { EventPopup, PaydayPopup },
  data() {
    return {
      weekDays: ['Sun', 'Mon', 'Tue', 'Wed', 'Thu', 'Fri', 'Sat'],
      events: JSON.parse(localStorage.getItem('events') || '{}'),
      payday: JSON.parse(localStorage.getItem('payday') || 'null'),
      selectedDate: null,
      selectedPrev: false,
      selectedNext: false,
      today: new Date()
    };
  },
  mounted() {
    if (!this.payday) {
      this.$refs.paydayPopup.show();
    }
    this.updateTotals();
  },
  computed: {
    daysInMonth() {
      const days = [];
      const firstDayOfMonth = new Date(this.currentYear, this.currentMonth, 1).getDay();
      const numberOfDays = new Date(this.currentYear, this.currentMonth + 1, 0).getDate();
      const prevMonthDays = new Date(this.currentYear, this.currentMonth, 0).getDate();

      // Days from previous month
      for (let i = firstDayOfMonth - 1; i >= 0; i--) {
        days.push({ date: prevMonthDays - i, fullDate: new Date(this.currentYear, this.currentMonth - 1, prevMonthDays - i), events: this.getEventsForDate(new Date(this.currentYear, this.currentMonth - 1, prevMonthDays - i)), isPrev: true, isNext: false });
      }

      // Days from current month
      for (let i = 1; i <= numberOfDays; i++) {
        days.push({ date: i, fullDate: new Date(this.currentYear, this.currentMonth, i), events: this.getEventsForDate(new Date(this.currentYear, this.currentMonth, i)), isPrev: false, isNext: false });
      }

      // Days from next month
      const nextDays = 42 - days.length; // To fill 6 weeks
      for (let i = 1; i <= nextDays; i++) {
        days.push({ date: i, fullDate: new Date(this.currentYear, this.currentMonth + 1, i), events: this.getEventsForDate(new Date(this.currentYear, this.currentMonth + 1, i)), isPrev: false, isNext: true });
      }

      return days;
    },
    totalIn() {
      let total = 0;
      if (this.payday) {
        const paydayDate = new Date(this.payday.date);
        const firstDayOfMonth = new Date(this.currentYear, this.currentMonth, 1);
        const lastDayOfMonth = new Date(this.currentYear, this.currentMonth + 1, 0);

        // Start from the closest payday before or at the beginning of the month
        let nextPayday = new Date(paydayDate);
        while (nextPayday < firstDayOfMonth) {
          nextPayday.setDate(nextPayday.getDate() + 14);
        }
        // Adjust to the previous payday if necessary
        nextPayday.setDate(nextPayday.getDate() - 14);

        // Iterate through the month to add paydays
        while (nextPayday <= lastDayOfMonth) {
          if (nextPayday >= firstDayOfMonth) {
            total += this.payday.amount;
          }
          nextPayday.setDate(nextPayday.getDate() + 14);
        }
      }
      return total;
    },
    totalOut() {
      let total = 0;
      const numberOfDays = new Date(this.currentYear, this.currentMonth + 1, 0).getDate();
      for (let i = 1; i <= numberOfDays; i++) {
        const date = `${this.currentYear}-${this.currentMonth + 1}-${i}`;
        if (this.events[date]) {
          this.events[date].forEach(event => {
            total += event.amount;
          });
        }
      }
      return total;
    },
    netTotal() {
      return this.totalIn - this.totalOut;
    },
    remainingBalance() {
      const balance = {};
      let currentBalance = this.payday ? this.payday.amount : 0;
      
      for (const day of this.daysInMonth) {
        const dateString = `${day.fullDate.getFullYear()}-${day.fullDate.getMonth() + 1}-${day.fullDate.getDate()}`;
        
        // Reset balance on payday
        if (this.isPayday(day.fullDate)) {
          currentBalance = this.payday.amount;
        }
        
        if (this.events[dateString]) {
          for (const event of this.events[dateString]) {
            currentBalance -= event.amount;
          }
        }
        
        balance[dateString] = currentBalance;
      }
      
      return balance;
    }
  },
  methods: {
    showEventPopup(day, isPrev, isNext) {
      this.selectedDate = day;
      this.selectedPrev = isPrev;
      this.selectedNext = isNext;
      this.$refs.eventPopup.show();
    },
    addEvent(event) {
      const id = Date.now();
      const newEvent = { id, name: event.name, amount: event.amount, repeat: event.repeat, paid: false };
      this.addEventToDate(newEvent, this.selectedDate, this.selectedPrev, this.selectedNext);
      if (event.repeat) {
        this.addRepeatingEvent(newEvent);
      }
      this.saveEvents();
      this.updateTotals();
    },
    addEventToDate(event, day, isPrev, isNext) {
      let date;
      if (isPrev) {
        date = `${this.currentYear}-${this.currentMonth}-${day}`;
      } else if (isNext) {
        date = `${this.currentYear}-${this.currentMonth + 2}-${day}`;
      } else {
        date = `${this.currentYear}-${this.currentMonth + 1}-${day}`;
      }
      if (!this.events[date]) {
        this.events[date] = [];
      }
      this.events[date].push(event);
    },
    addRepeatingEvent(event) {
      const originalDate = new Date(this.currentYear, this.currentMonth, this.selectedDate);
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
      const baseEventId = eventId.toString().split('-')[0]; // Convert to string and get the base ID
      for (const key in this.events) {
        if (Object.prototype.hasOwnProperty.call(this.events, key)) {
          this.events[key] = this.events[key].filter(event => !event.id.toString().startsWith(baseEventId));
          if (this.events[key].length === 0) {
            delete this.events[key];
          }
        }
      }
      this.saveEvents();
      this.updateTotals();
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
      this.updateTotals();
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
              // Check if the event should be repeated on the current date
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
      this.$emit('update-totals', {
        totalIn: this.totalIn,
        totalOut: this.totalOut,
        netTotal: this.netTotal
      });
    },
    isCurrentDay(date) {
      const today = new Date();
      return date.getDate() === today.getDate() && date.getMonth() === today.getMonth() && date.getFullYear() === today.getFullYear();
    }
  },
  watch: {
    currentMonth() {
      this.updateEvents();
    },
    currentYear() {
      this.updateEvents();
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
  overflow-y: auto;
  padding: 0 10px;
  background-color: #121212;
  color: #ffffff;
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
  background-color: #1e1e1e;
  font-size: 0.9rem;
}

.day.current-day {
  background: linear-gradient(135deg, rgba(255,255,255,0.2), rgba(255,255,255,0.1));
}

.day.prev-month,
.day.next-month {
  background-color: #2a2a2a;
  color: #777;
}

.day-number {
  background-color: #333;
  padding: 2px 5px;
  border-radius: 3px;
  font-size: 0.9rem;
}

.remaining-balance {
  font-size: 0.8rem;
  color: #ffcc00;
}

.event
{
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

@media (max-width: 768px) {
  .calendar {
    grid-template-columns: repeat(2, 1fr);
  }
  .day {
    min-height: calc((100vh - 100px) / 3);
  }
}
</style>

