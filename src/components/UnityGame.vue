<template>
  <div>
    <div class="columns is-vcentered" style="margin-top:250px" v-show="isLoading == true">
      <div class="column" >
          <strong>LOADING</strong>
          <progress class="progress is-large is-info" max="100">60%</progress>
      </div>
    </div>
    <div class="columns">
      <div class="column">
        <div ref="unity_container" v-show="isLoading == false">
          <canvas ref="unity_canvas" v-bind:style="canvasStyle" />
        </div>
      </div>
    </div>      
    <div class="modal" v-bind:class="{'is-active': this.activeModal == true}">
      <div class="modal-background"></div>
      <div class="modal-content">
        <div class="box">
          <p>{{this.activeAddress['title']}}</p>
          <p>{{this.activeAddress['contents']}}</p>
        </div>
      </div>
      <button class="modal-close is-large" v-on:click="closeModal()" aria-label="close"></button>
    </div>
  </div>
</template>

<script>
class GameMessage {
  constructor(cmd, data) {
    this.command = cmd
    this.data = data
  }

  toJSON() {
    return JSON.stringify({"command": this.command, "data": this.data })
  }

  static fromJSON(strData) {
    let cmd = ''
    let data = {}
    let parseData = JSON.parse(strData)
    cmd = parseData['command']
    data = JSON.parse(parseData['data'])
    return new GameMessage(cmd, data)
  }
}
export default {
  name: 'UnityGame',
  data: function () {
    return {
        isLoading:true,
        canvasStyle: {
          width: "100px",
          height: "100px"
        },
        socket: null,
        activeAddress: {},
        activeModal: false,
        buildUrl: "game/Build",
        loaderUrl: "game/Build" + "/game.loader.js",
        config: {
          dataUrl: "game/Build" + "/game.data",
          frameworkUrl: "game/Build" + "/game.framework.js",
          codeUrl: "game/Build" + "/game.wasm",
          streamingAssetsUrl: "StreamingAssets",
          companyName: "DefaultCompany",
          productName: "VRWebSite",
          productVersion: "0.1",
        }
      }
    },
  mounted: function () {
    window.$vm.triggerJSAction = this.triggerJSAction;
    var canvas = this.$refs.unity_canvas;
      if (/iPhone|iPad|iPod|Android/i.test(navigator.userAgent)) {
        console.log('its on mobile!');
        return;
      } else {
        this.canvasStyle.width = "100%";
        this.canvasStyle.height = "100%";
      }
      var self = this;
      self.socket = new WebSocket(process.env.VUE_APP_BASE_URL + "/ws")
      console.log(self.socket)
      self.socket.onopen = function() {
        let a = new GameMessage("Conn","ok")
        self.socket.send(a.toJSON())
      }

      self.socket.onmessage = function(event) {        
        // The information sent from the server will be in the 
        // data key for the event
        let evtData = GameMessage.fromJSON(event.data)
        if (evtData.command == "Send Record") {
          self.activeAddress = evtData.data
          self.activeModal = true
        }
        

      }
      window.createUnityInstance(canvas, self.config, () => {
          self.isLoading = true;
        }).then((unityInstance) => {
          self.isLoading = false;
          self.unityInstance = unityInstance;
          let msg = new GameMessage("Game", "ok")
          self.socket.send(msg)
        }).catch((message) => {
          alert(message);
        });
  },
  methods: {
    sendToServer: function (msg){
      this.socket.send(msg.toJSON())
    },
    closeModal: function () {
      this.activeModal = false;
      this.activeAddress = {};
    },
    triggerJSAction: function(data) {
      console.log("from vue component " + data);
      data = JSON.parse(data);
      if (data["_command"] === "changeActiveAddress") {
        this.activeAddress = parseInt(data['_data']);
        let msg = new GameMessage("View Record", data['_data'])
        this.sendToServer(msg);
      } 

    }

  }
}
</script>

<style scoped>
h3 {
  margin: 40px 0 0;
}
ul {
  list-style-type: none;
  padding: 0;
}
li {
  display: inline-block;
  margin: 0 10px;
}
a {
  color: #42b983;
}
</style>
