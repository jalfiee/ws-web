<template>
  <div id="app">
    <p @click="this.tryInitiatingNotification">tryInitiatingNotification</p>
    <p @click="this.tryAuthedTest">tryAuthedTest</p>
    <div v-if="userLoggedIn">
      <p @click="this.tryLogout">Logout</p>
    </div>
    <div v-else>
      <p @click="this.tryLogin">Login</p>
    </div>
  </div>
</template>

<script>
import axios from 'axios'
// import Echo from 'laravel-echo'
import Pusher from 'pusher-js' // eslint-disable-line no-unused-vars
axios.defaults.baseURL = 'http://ws-api.lndo.site/api/v1'
axios.defaults.withCredentials = true;

export default {
  name: 'App',
  data: function() {
    return {
      userLoggedIn: false,
      userId: null,
      clientToken: null,
      xrsfToken: null,
    }
  },
  created() {
    if (this.userLoggedIn) {
      console.log('user logged in, subscribing to private channel');
      this.echoSubscribe()
    }
    else {
      console.log('user NOT logged in, can\'t subscribe to private channel');
    }
  },
  methods: {
    async tryInitiatingNotification() {
      console.log('START OF /authed/notify');
      
      await axios.get('/authed/notify')
      .then(res => {
        console.log('tryInitiatingNotification resp', res.data);
      })
      .catch(err => {
        console.log("tryInitiatingNotification error", err);
      })
      console.log('END OF /authed/notify');
    },
    async tryAuthedTest() {
      console.log('START OF /authed/test');
      
      await axios.get('/authed/test')
      .then(res => {
        console.log('tryAuthedTest resp', res.data);
      })
      .catch(err => {
        console.log("tryAuthedTest error", err);
      })
      console.log('END OF /authed/test');
    },
    async tryLogin () {
      // initialise CSRF
      await axios({
        method: 'get',
        url: '/sanctum/csrf-cookie',
        baseURL: 'http://ws-api.lndo.site',
      }).then(() => {
        // Login...
        axios.get('/auth/login')
        .then(res => {
          this.userLoggedIn = res.data.auth
          this.userId = res.data.id
          this.clientToken = res.data.client_token
          this.xrsfToken = res.config.headers['X-XSRF-TOKEN']

          console.log('authenticated, subscribing to private channel')
          this.pusherSubscribe()
        })
        .catch(err => {
          console.log("tryLogin error", err)
        })
      });
    },
    async tryLogout () {
      await axios.get('/auth/logout')
      .then(res => {
        this.userLoggedIn = res.data.auth
        this.userId = res.data.id
        this.clientToken = res.data.client_token
        this.xrsfToken = null
      })
      .catch(err => {
        console.log("tryLogout error", err)
      })
    },
    pusherSubscribe() {
      let pusher = new Pusher(process.env.PUSHER_APP_KEY,
        {
          cluster: 'eu',
          authEndpoint: 'http://ws-api.lndo.site/api/broadcasting/auth',
          auth: {
            headers: {
              'X-CSRF-Token': this.xrsfToken
            }
          }
        }
      )
      pusher.subscribe('private-user.' + this.userId)
      
      // pusher.subscribe('my-channel')
      pusher.bind('my-event', data => {
        this.pusherHandleEvent(data)
      })
    },
    pusherHandleEvent(data) {
      console.log('event received', data);
    }


    // echoSubscribe() {
    //   window.Echo = new Echo({
    //     broadcaster: "pusher",
    //     cluster: 'eu',
    //     encrypted: true,
    //     key: process.env.PUSHER_APP_KEY,
    //     authorizer: (channel, options) => { // eslint-disable-line no-unused-vars
    //       return {
    //         authorize: (socketId, callback) => { // eslint-disable-line no-unused-vars
    //           axios({
    //             method: 'post',
    //             url: '/api/broadcasting/auth',
    //             baseURL: 'http://ws-api.lndo.site',
    //             data: {
    //               socket_id: socketId,
    //               channel_name: 'private-user.' + this.userId
    //             },
    //           })
    //           .then(response => {
    //             console.log("echoSubscribe.then", false, response.data)
    //             callback(false, response.data)
    //           })
    //           .catch(error => {
    //             console.log("echoSubscribe.catch", true, error)
    //             callback(true, error)
    //           });
    //         }
    //       };
    //     },
    //   })
    // },
    // echoHandleEvent(data) {
    //   console.log('event received', data);
    // },
  }
}
</script>

<style>

</style>
