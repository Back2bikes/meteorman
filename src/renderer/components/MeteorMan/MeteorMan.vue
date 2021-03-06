<template>
  <v-container fluid>
    <server-connection ref="serverRef" @onUpdateConnection="updateConnection"></server-connection>
    <div class="connection-wrapper">
      <v-row>
        <v-col class="d-flex">
          <div class="small-field">
            <v-select :items="dppTypes" v-model="typeSelected" dense outlined class="start-addon"
                      background-color="#ececec"/>
          </div>
          <v-text-field v-if="typeSelected==='Method'" placeholder="Enter method name" class="end-addon"
                        background-color="#ececec"
                        v-model="meteorMethod.name" dense outlined type="text">
            <template v-slot:append>
              <v-btn icon @click="meteorMethod.name = ''" tabindex="-1" class="pb-5">
                <v-icon v-if="meteorMethod.name">mdi-close</v-icon>
              </v-btn>
            </template>
          </v-text-field>
          <v-text-field v-else placeholder="Enter publication name" class="mid-addon"
                        background-color="#ececec"
                        v-model="publication.name" dense outlined clearable type="text"/>
          <v-text-field v-if="typeSelected==='Subscription'" placeholder="Enter collection name" class="end-addon"
                        background-color="#ececec"
                        v-model="publication.collectionName" dense outlined clearable type="text"/>
          <v-btn v-on:click="ddp()" class="ml-2" color="#387be5" dark elevation="0" height="40px">
            Send
            <v-icon right small>mdi-send</v-icon>
          </v-btn>
        </v-col>
      </v-row>
      <Split style="height: calc(100vh - 250px);" direction="vertical">
        <SplitArea>
          <arguments ref="argsRef"></arguments>
        </SplitArea>
        <SplitArea>
          <method-response ref="methodResponseRef"></method-response>
        </SplitArea>
      </Split>
    </div>
  </v-container>
</template>

<script>
import ServerConnection from '../ServerConnection/ServerConnection';
import Arguments from './Arguments';
import MethodResponse from './MethodResponse';

export default {
  name: 'MeteorMan', // TODO: Refactor the name of this component
  components: { MethodResponse, Arguments, ServerConnection },
  data() {
    return {
      dppTypes: [
        'Method',
        'Subscription'
      ],
      typeSelected: 'Method',
      meteorMethod: {
        name: null,
        args: null
      },
      publication: {
        name: null,
        collectionName: null
      },
      methodResponse: '',
      isSubscriptionInProgress: null,
      connected: false,
      methodResponseParsed: '',
      defaultHeight: window.innerHeight - 288
    };
  },
  methods: {
    updateConnection(value) {
      this.connected = value;
    },
    async callMethod(args) {
      try {
        this.methodResponseParsed = await this.$refs.serverRef.Meteor.call(this.meteorMethod.name, ...args);
        this.$refs.methodResponseRef.loadResponse(this.methodResponseParsed);
        this.methodResponse = JSON.stringify(this.methodResponseParsed, undefined, 4);
      } catch (exception) {
        console.error('meteor method error:', exception);
        this.methodResponseParsed = exception;
        this.$refs.methodResponseRef.loadResponse(this.methodResponseParsed);
        this.methodResponse = JSON.stringify(exception, undefined, 4);
      }
    },
    async subscribeToPublication(args) {
      try {
        if(this.isSubscriptionInProgress){
          if (await this.isSubscriptionInProgress.isOn()) {
            await this.isSubscriptionInProgress.stop();
            this.$refs.serverRef.Meteor.stopChangeListeners();
          }
        }
        this.isSubscriptionInProgress = this.$refs.serverRef.Meteor.subscribe(this.publication.name, ...args);
        await this.isSubscriptionInProgress.start();
        await this.isSubscriptionInProgress.ready();
        const firstResponse = this.$refs.serverRef.Meteor.collection(this.publication.collectionName).filter(e => e).fetch();
        this.$refs.methodResponseRef.loadResponse(firstResponse);
        this.$refs.serverRef.Meteor.collection(this.publication.collectionName).filter(e => e).onChange(({ prev, next }) => {
          this.$refs.methodResponseRef.loadResponse(next);
        });
      } catch (exception) {
        console.error('subscription error: ', exception);
        this.$refs.methodResponseRef.loadResponse(exception);
      }
    },
    loadArguments(args) {
      let finalArgs = [];
      for (let arg of args) {
        switch (arg.type.name) {
          case 'object':
            finalArgs.push(arg.json);
            break;
          case 'array':
            finalArgs.push(arg.array);
            break;
          case 'string':
            finalArgs.push(arg.value);
            break;
          case 'boolean':
            finalArgs.push(!!arg.value);
            break;
          case 'number':
            finalArgs.push(arg.value);
            break;
        }
      }
      return finalArgs;
    },
    ddp() {
      const args = this.loadArguments(this.$refs.argsRef.args);
      if (this.typeSelected === 'Method') {
        this.callMethod(args);
      } else {
        this.subscribeToPublication(args);
      }
    }
  }
};
</script>

<style scoped>
.connection-wrapper /deep/ .v-input__control {
  height: 40px;
}
</style>
<style>
.tree-view-item-key {
  color: #9e3731;
  font-weight: normal;
}

.tree-view-item-value-string, .tree-view-item-value-boolean {
  color: #2251a0;
}
.tree-view-item-value-number {
  color: #3c845c;
}
</style>
