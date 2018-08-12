<template>
  <div>
    <!-- Heading -->
    <h2 class="header z-depth-2">
      NaaS Playground
    </h2>

    <!-- Progress -->
    <div class="row steps">
      <div class="col s4 right-align"><span class="step" v-bind:class="isActive(1)">1</span></div>
      <div class="col s4 center-align "><span class="step" v-bind:class="isActive(2)">2</span></div>
      <div class="col s4 left-align"><span class="step" v-bind:class="isActive(3)">3</span></div>
    </div>

    <!-- Start from beginning  -->
    <div v-if="currentStep != 1" class="restart">
      <button class="waves-effect waves-light btn red" v-on:click="startAgain">Start again</button>
    </div>

    <div v-if="errorMessage != null" v-html="errorMessage" class="errorMessage"></div>

    <!-- Step 1: Credentials --> 
    <div v-if="currentStep === 1" class="credentials" >
      <div class="row">
        <h4>Authentication</h4>
        <div class="col s3 offset-s3 input-field">
          <input id="clientId" type="text" v-model="clientId" />
          <label for="clientId">Client Id</label>
        </div>
        <div class="col s3 input-field">
          <input id="clientSecret" type="text" v-model="clientSecret" />
          <label for="clientSecret">Client Secret</label>
        </div>
      </div>
      <div style="text-align:center">
        <a class="waves-effect waves-light btn" v-bind:class="{ disabled: disableAuthButton }" v-on:click="authenticate">Authenticate</a>
      </div>
    </div> 


    <!-- Step 2: Eline -->
    <div v-if="currentStep === 2" class="service" >
      <div class="row">
        <h4><strong>ELine</strong></h4>
        <h5>Activation and Configuration</h5>
        <div style="padding: 20px;">
          <button class="waves-effect waves-light btn" v-bind:class="{ disabled: disableSendButton }" v-on:click="sendRequest">Send Request</button>
          &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
          <button class="waves-effect waves-ripple orange btn" v-on:click="resetData">Reset changes</button>
        </div>
        <h6>Edit payload below and click on <strong>Send Request</strong> or <strong>Reset changes</strong> to undo changes.</h6>
        <div class="col s6 offset-s3 input-field">
          <textarea v-model="serviceCharacteristics" class="materialize-textarea" rows="20"></textarea>
        </div>
      </div>
    </div> 

    <!-- Results -->
    <div v-if="currentStep === 3">
      <h4>ELine</h4>
      <h5>Activation and Configuration</h5>
      <div>Response</div>
      <div class="row">
        <div class="col s8 offset-s2 response">
          <pre>{{ responsePayload | pretty}}</pre>
        </div>
      </div>
    </div>
  </div> 

</template>

<script>

import ElineData from "../data/data";

export default {
  name: 'Demo',
  data() {
    return {
       currentStep: 1,
       clientId: null,
       clientSecret: null,
       apiBaseUrl: null,
       authToken: null,
       responsePayload: null,

       authError: false,
       sendError: false,
       responseError: false,
       errorMessage: null,

       disableAuthButton: false,
       disableSendButton: false,
       serviceCharacteristics: null,
       serviceCharacteristicsOrig: null
    }
  },
  methods: {
    isActive: function(step) {
      if (step == 1 && this.authError) {
        return { stepError: true }
      }

      if (step == 2 && this.sendError) {
        return { stepError: true }
      }

      if (step == 3 && this.responseError) {
        return { stepError: true }
      }

      if (this.currentStep >= step) {
        return { stepActive: true };
      }
      return { stepActive: false };
    },
    authenticate: async function() {
      this.disableAuthButton = true;
      this.authError = false;
      this.errorMessage = null;

      const tokenPayload = {
          grant_type: "client_credentials"
      };
      tokenPayload['client_id'] = this.clientId;
      tokenPayload['client_secret'] = this.clientSecret;

      try {
        let authResponse = await axios.post(`${this.apiBaseUrl}/token`, tokenPayload);
        this.authToken = authResponse.data.access_token;        
        console.log(this.authToken)
        this.currentStep = 2;
      }
      catch(err) {
        M.toast({html: err})
        this.authError = true;
        this.errorMessage = err.response.statusText;
      }
      finally {
        this.disableAuthButton = false;
      }
    },
    sendRequest: async function() {
      this.sendError = false;
      this.errorMessage = null;

      let url = `${this.apiBaseUrl}/activationAndConfiguration/v2/service/Eline` 
      let payload = null;
      try {
        payload = {
          serviceCharacteristic: JSON.parse(this.serviceCharacteristics)
        }
      }
      catch(err) {
        M.toast({ html: err })
        this.sendError = true;
        this.errorMessage = err;
        return;
      }

      try {
        this.disableSendButton = true;
        let config = {
          headers: {
            "Authorization": `${this.authToken}`,
            "Content-Type": "application/json",
            "X-Group-Id": "test" 
          }
        }
        let response = await axios.post(url, payload, config);
        this.responsePayload = response.data;
        this.currentStep = 3;
      }
      catch(err) {
        M.toast({ html: 'Error sending request.' })
        this.sendError = true;
        this.errorMessage = err.response.data.reason;
      }
      finally {
        this.disableSendButton = false;
      }
      
    },
    resetData: function () {
      this.serviceCharacteristics = this.serviceCharacteristicsOrig;
      M.toast({html: "Reset complete."})
    },
    startAgain: function() {
      this.currentStep = 1;
      this.authToken = null;
      this.errorMessage = null;
      this.authError = false;
      this.sendError = false;
      this.responseError = false;
      this.responsePayload = null;
      this.serviceCharacteristics = this.serviceCharacteristicsOrig;
    }
  },
  filters: {
    pretty: function(value) {
      return JSON.stringify(value, null, 2);
    }
  },
  created: function() {
    this.clientId = "fcf71d73-9d4a-40b4-bf95-0a2f7130e8d9";
    this.clientSecret = "e96bf843-0979-42a1-8604-76ce80124ce7";
    this.apiBaseUrl = "http://api.poc.ohub.media/naas"
    this.authToken = null;
    this.currentStep = 1;
    let serviceCharacteristics = JSON.stringify(ElineData, null, 2); 
    this.serviceCharacteristics = serviceCharacteristics;
    this.serviceCharacteristicsOrig = serviceCharacteristics;   
    if (this.authToken) {
      this.currentStep = 2;
    }
  }

}

</script>

<style scoped>
  .header {
    background-color: #444;
    color: #fff;
    text-align: center;
    padding: 10px;
    margin-top: 0px;
    margin-bottom: 0px;
  }

  .steps {
    margin-top: 20px;
    padding: 20px;
  }
  
  .step {
    padding: 15px 25px;
    font-size: 20px;
    border-radius: 50%;
    border: 1px solid #666;
    background-color: #888;
    color: #fff;
  }

  .stepActive {
    background-color: #26a69a;
  }

  .stepError {
    background-color: #F44336;
  }

  .credentials {
    margin-top: 20px;
  }

  .service {
    margin-top: 20px;
  }

  .response {
    margin-top: 20px;
    background-color: #444;
    color: #fff;
    padding: 30px;
    text-align: left;
  }

  .restart {
    margin-top: 20px;
    margin-bottom: 60px;
  }

  .errorMessage {
    margin: 20px;
    padding: 20px;
    background-color: #F44336;
    color: #fff;
  }
  .materialize-textarea {
    min-height: 130rem !important;
  }
</style>
