<template>
  <div>
    <button @click="openLink" class="btn btn-primary">Link Bank Account</button>
  </div>
</template>

<script>
export default {
  name: 'PlaidLinkButton',
  methods: {
    async openLink() {
      const response = await fetch('http://localhost:3000/api/create_link_token', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
        },
        body: JSON.stringify({ userId: 'unique-user-id' }),
      });

      const data = await response.json();
      const linkToken = data.link_token;

      const script = document.createElement('script');
      script.src = 'https://cdn.plaid.com/link/v2/stable/link-initialize.js';
      script.onload = () => this.initPlaidLink(linkToken);
      document.head.appendChild(script);
    },
    initPlaidLink(linkToken) {
      const handler = window.Plaid.create({
        token: linkToken,
        onLoad: () => {
          console.log('Plaid Link loaded');
        },
        onSuccess: (public_token) => {
          fetch('http://localhost:3000/api/exchange_public_token', {
            method: 'POST',
            headers: {
              'Content-Type': 'application/json',
            },
            body: JSON.stringify({ publicToken: public_token }),
          })
            .then((response) => response.json())
            .then((data) => {
              const accessToken = data.access_token;
              const startDate = new Date();
              startDate.setMonth(startDate.getMonth() - 2);
              const endDate = new Date();

              fetch('http://localhost:3000/api/transactions', {
                method: 'POST',
                headers: {
                  'Content-Type': 'application/json',
                },
                body: JSON.stringify({
                  accessToken,
                  startDate: startDate.toISOString().split('T')[0],
                  endDate: endDate.toISOString().split('T')[0],
                }),
              })
                .then((response) => response.json())
                .then((data) => {                  
                  this.$emit('transactions', data);
                })
                .catch((error) => {
                  console.error('Error fetching transactions:', error);
                });
            })
            .catch((error) => {
              console.error('Error exchanging public token:', error);
            });
        },
        onExit: (err) => {
          console.error('Plaid Link exited', err);
        },
      });

      handler.open();
    },
  },
};
</script>

<style scoped>
#plaid-link-button {
  display: flex;
  justify-content: center;
  align-items: center;
  padding: 20px;
  background-color: #007bff;
  color: white;
  cursor: pointer;
}
</style>
