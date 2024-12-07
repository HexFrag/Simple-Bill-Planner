<template>
  <div id="app">
    <nav class="navbar navbar-dark bg-dark">
      <div class="container-fluid d-flex align-items-center justify-content-between">
        <span class="navbar-brand mb-0 h1">
          Bill Payment Planner - {{ currentMonthName }} {{ currentYear }}
        </span>
        <div class="d-flex align-items-center flex-grow-1 justify-content-center">
          <div class="mx-4 text-center">
            <strong>Balance:</strong> {{ formatCurrency(balance) }}
          </div>
          <div class="mx-4 text-center">
            <strong>Available:</strong> {{ formatCurrency(available) }}
          </div>
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
        :events="events"
        @update-totals="updateTotals"
      />
    </div>
    <PlaidLinkButton @transactions="handleTransactions" />
  </div>
</template>

<script>

import CalendarMain from './components/calendar-main.vue';
import PlaidLinkButton from './components/PlaidLinkButton.vue';

export default {
  name: 'App',
  components: {
    CalendarMain,
    PlaidLinkButton,
  },
  data() {
    return {
      currentYear: new Date().getFullYear(),
      currentMonth: new Date().getMonth(),
      totalIn: 0,
      totalOut: 0,
      netTotal: 0,
      balance: 0,
      available: 0,
      events: {},
      allAccounts: [],
    };
  },
  computed: {
    currentMonthName() {
      const monthNames = [
        'January',
        'February',
        'March',
        'April',
        'May',
        'June',
        'July',
        'August',
        'September',
        'October',
        'November',
        'December',
      ];
      return monthNames[this.currentMonth];
    },
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
    },
    handleTransactions(data) {
      console.log('Transactions:', data);
      
      this.organizeAccounts(data.accounts);
      if(this.allAccounts['depository'])
      {
        let checkingAccounts = this.allAccounts['depository']['checking'];
        if(checkingAccounts.length > 0)
        {
            let totals = checkingAccounts.reduce((totals, account) => {
              totals.currentTotal += account.balances.current || 0;
              totals.availableTotal += account.balances.available || 0;
              return totals;
            }, { currentTotal: 0, availableTotal: 0 });

            this.balance = totals.currentTotal;
            this.available = totals.availableTotal;

        }
      }

      this.processTransactions(data.transactions);

    },
    organizeAccounts(accounts)
    {
      this.allAccounts = accounts.reduce((acc, account) => {
        const { type, subtype } = account;
        if(!acc[type]) acc[type] = {};
        if(!acc[type][subtype]) acc[type][subtype] = []
        acc[type][subtype].push(account);
        return acc;
      }, {});
    },
    processTransactions(transactions)
    {
      const recurringMap = {};
      transactions.forEach(transaction => {
        if(transaction.amount > 0)
        {
          this.addEvent(transaction.name, transaction.amount, transaction.date, true);
        }
        else if(transaction.amount < 0)
        {
          const key = '${transaction.name}:${transaction.date}';
          if(!recurringMap[key]){
            recurringMap[key] = []
          }
          recurringMap[key].push(transaction);
        }
        
      });

      for(const key in recurringMap)
      {
        if(recurringMap[key].length > 1)
        {
          const transaction = recurringMap[key][0];
          this.setPayday(transaction.date, transaction.amount);
          break;
        }        
      }


    },
    addEvent(name, value, date, recurring = true) {
      let _date = new Date(date);
      this.$refs.calendarMain.addEvent({
        name,
        amount: value,
        date: _date,
        repeat: recurring,
      });
    },
    setPayday(date, amount) {
      let _amt = Math.abs(amount);
      this.$refs.calendarMain.setPayday({
        date,
        amount: _amt,
      });
    },
  },
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
