<template>
  <div id="app" v-if="!$auth.loading">
    <nav class="navbar navbar-expand-lg navbar-dark bg-dark">
      <a class="navbar-brand" href="#">The Emoji Game</a>
      <button
        class="navbar-toggler"
        type="button"
        data-toggle="collapse"
        data-target="#navbarText"
        aria-controls="navbarText"
        aria-expanded="false"
        aria-label="Toggle navigation"
      >
        <span class="navbar-toggler-icon"></span>
      </button>
      <div class="collapse navbar-collapse" id="navbarText">
        <div class="navbar-nav mr-auto user-details">
          <span v-if="$auth.isAuthenticated">{{ $auth.user.name }} ({{ $auth.user.email }})</span>
          <span v-else>&nbsp;</span>
        </div>
        <span class="navbar-text">
          <ul class="navbar-nav float-right">
            <li class="nav-item" v-if="!$auth.isAuthenticated">
              <a class="nav-link" href="#" @click="login()">Log In</a>
            </li>
            <li class="nav-item" v-if="$auth.isAuthenticated">
              <a class="nav-link" href="#" @click="logout()">Log Out</a>
            </li>
          </ul>
        </span>
      </div>
    </nav>
    <div class="container mt-5" v-if="!$auth.isAuthenticated">
      <div class="row">
        <div class="col-md-8 offset-md-2">
          <div class="jumbotron">
            <h1 class="display-4">Play the Emoji Game!</h1>
            <p class="lead">Instructions</p>
            <ul>
              <li>Login to play</li>
              <li>Select a camera feed of your choice from the drop down</li>
              <li>Click “Play” to start a new round</li>
              <li>Capture the correct image that matches the emoji displayed</li>
              <li>You can skip an emoji during a round if you can’t find the corresponding object around you to capture</li>
              <li>Click ‘Predict Image’ to compare your captured image against the emoji</li>
              <li>You gain 10 points for a correct prediction and loose 5 points for a wrong prediction</li>
              <li>If the clock runs out, its Game Over!</li>
            </ul>
            <a
              class="btn btn-primary btn-lg mr-auto ml-auto"
              href="#"
              role="button"
              @click="login()"
            >Log In to Play</a>
          </div>
        </div>
      </div>
    </div>
    <!-- The emoji game -->
    <div class="container mt-5" v-else>
      <div class="row">
        <div class="col-md-4">
          <select class="form-control" v-model="selectedSource" @change="getStream()">
            <option
              v-for="source in videoSources"
              :key="source.id"
              :value="source.id"
            >{{source.text}}</option>
          </select>
        </div>
        <div class="col-md-8">
          <div class="row">
            <div class="col-md-4">Countdown : {{timerStart}}</div>
            <div class="col-md-4">Total Score: {{totalScore}}</div>
          </div>
        </div>
      </div>
      <div class="row mt-5" v-if="gameOver">
        <button class="btn btn-primary" @click="startNewGame()">Start New Game</button>
      </div>
      <div class="row mt-5" v-else>
        <div class="col-md-6">
          <button class="btn btn-success" :disabled="inPlay" @click="playGame()">Play</button>
          &nbsp;
          <button
            class="btn btn-warning"
            :disabled="!inPlay"
            @click="skipEmoji()"
          >Skip Emoji</button>
        </div>
      </div>
      <!-- row -->
      <div class="row mt-5">
        <div class="col-md-6">
          <div>
            <video autoplay id="video1" ref="video1"></video>
            <canvas style="display:none;" id="canvas1" ref="canvas1"></canvas>
          </div>
          <div>
            <button class="btn btn-primary" @click="captureImage()">Capture Image</button>
          </div>
        </div>
        <div class="col-md-6">
          <div class="row">
            <div class="col-md-6">
              <img
                id="usercapture"
                ref="usercapture"
                src="http://via.placeholder.com/150x150"
                width="200"
                height="200"
              />
              <div class="mt-2">
                <button
                  class="btn btn-success"
                  @click="predictImage()"
                  :disabled="predictingImage"
                >{{predictingImage? 'Please Wait...' : 'Predict Image'}}</button>
              </div>
            </div>
            <div class="col-md-6">
              <p class="currentEmoji">{{currentEmoji.char}}</p>
            </div>
          </div>
        </div>
      </div>
      <!-- Modal -->
      <modal
        v-if="modal.show"
        @close="modal.show = false"
        :header="modal.header"
        :content="modal.content"
      ></modal>
    </div>
  </div>
</template>
<script>
import "bootstrap/dist/css/bootstrap.css";
import axios from "axios";
import emojis from "emoji.json";
import modal from "./components/modal";
export default {
  name: "app",
  data() {
    return {
      videoSources: [],
      selectedSource: null,
      totalScore: 0,
      currentEmoji: {},
      imageURL: null,
      predictingImage: false,
      gCloudVisionUrl:
        "https://vision.googleapis.com/v1/images:annotate?key=AIzaSyDl66LUypIKFYT6NvGCtmQHrMwiyRLuwp8",
      timerHandle: null,
      timerStart: 60,
      pointsIncrement: 10,
      pointsDecrement: 5,
      inPlay: false,
      gameOver: false,
      modal: {
        show: false,
        header: "My header",
        content: "My Content"
      }
    };
  },
  components: {
    modal
  },
  mounted() {
    navigator.mediaDevices
      .enumerateDevices()
      .then(this.gotDevices)
      .catch(this.handleError);
  },
  computed: {
    gameEmojis() {
      return emojis.filter(emoji => {
        return (
          emoji.category.includes("Objects") &&
          emoji.char.charCodeAt(0) != 55358
        );
      });
    }
  },
  methods: {
    // Log the user in
    login() {
      this.$auth.loginWithRedirect();
    },
    // Log the user out
    logout() {
      this.$auth.logout({
        returnTo: window.location.origin
      });
    },
    gotDevices(deviceInfos) {
      for (var i = 0; i !== deviceInfos.length; ++i) {
        var deviceInfo = deviceInfos[i];
        if (deviceInfo.kind === "videoinput") {
          let option = {};
          option.id = deviceInfo.deviceId;
          option.text = deviceInfo.label || "camera " + (i - 1);
          this.videoSources.push(option);
        }
      }
    },
    getStream() {
      if (window.stream) {
        window.stream.getTracks().forEach(function(track) {
          track.stop();
        });
      }
      var constraints = {
        video: {
          deviceId: { exact: this.selectedSource }
        }
      };
      navigator.mediaDevices
        .getUserMedia(constraints)
        .then(this.gotStream)
        .catch(this.handleError);
    },
    gotStream(stream) {
      this.$refs.video1.srcObject = stream;
    },
     handleError(error) {
      this.showModal(
          "Error",
          error.message
      );
    },
    playGame() {
      //Get Random Emoji
      this.switchEmoji();
      this.inPlay = true;
      //Start timer countdown
      this.setTimer();
    },
    skipEmoji() {
      this.switchEmoji();
      this.imageURL = null;
    },
    captureImage() {
      let canvas = this.$refs.canvas1;
      let videoElement = this.$refs.video1;
      canvas.width = videoElement.videoWidth;
      canvas.height = videoElement.videoHeight;
      canvas.getContext("2d").drawImage(videoElement, 0, 0);
      let imgElement = this.$refs.usercapture;
      // Get image
      let image = canvas.toDataURL("image/png");
      //Trim signature to get pure image data
      this.imageURL = image.replace(/^data:image\/(png|jpg);base64,/, "");
      //Set the image element to the data url
      imgElement.src = image;
    },
    async predictImage() {
      if (this.imageURL) {
        let requestBody = {
          requests: [
            {
              image: {
                content: this.imageURL
              },
              features: [
                {
                  type: "LABEL_DETECTION",
                  maxResults: 10
                }
              ]
            }
          ]
        };
        try {
          this.predictingImage = true;
          let predictionResults = await axios.post(
            this.gCloudVisionUrl,
            requestBody
          );
          let predictionResponse = predictionResults.data.responses[0];
          let annotations = predictionResponse.labelAnnotations;
          let allLabelDescriptions = annotations.map(annotation =>
            annotation.description.toLowerCase()
          );
          //Check if any of the prediction labels match the current emoji
          let match = false;
          allLabelDescriptions.forEach(description => {
            if (this.currentEmoji.name.includes(description)) {
              match = true;
            }
          });
          if (match == true) {
            this.totalScore += this.pointsIncrement;
            this.resetTimer();
            this.showModal(
              "Correct Answer",
              `Congratulations, you have gained ${this.pointsIncrement} points, your total is now ${this.totalScore}`
            );
          } else {
            this.totalScore -= this.pointsDecrement;
            this.showModal(
              "Wrong Answer",
              `The answer you gave was incorrect, Your captured image suggested the following (${allLabelDescriptions}). You have lost ${this.pointsDecrement} points, your total is now ${this.totalScore}`
            );
          }
          this.predictingImage = false;
        } catch (error) {
          this.handleError(error);
        }
      } else {
        this.showModal("Error", `You are yet to capture an image`);
      }
    },
    switchEmoji() {
      let emojiIndex = this.getRandomInt(0, this.gameEmojis.length);
      this.currentEmoji = this.gameEmojis[emojiIndex];
    },
    setTimer() {
      this.resetTimer();
      this.timerHandle = setInterval(() => {
        if (this.timerStart > 0) {
          this.timerStart -= 1;
        } else {
          //Game Over
          this.endGame();
        }
      }, 1000);
    },
    resetTimer() {
      //Stop the Clock
      clearInterval(this.timerHandle);
      this.timerStart = 60;
    },
    endGame() {
      clearInterval(this.timerHandle);
      this.inPlay = false;
      this.gameOver = true;
      this.showModal(
        "Game Over",
        `You could not complete the task before the time ran out. Your total score is ${this.totalScore}`
      );
    },
    startNewGame() {
      this.imageURL = null;
      this.totalScore = 0;
      this.gameOver = false;
      this.currentEmoji = {};
    },
    showModal(title, body) {
      this.modal = {
        show: true,
        header: title,
        content: body
      };
    },
    getRandomInt(min, max) {
      min = Math.ceil(min);
      max = Math.floor(max);
      return Math.floor(Math.random() * (max - min + 1)) + min;
    }
  }
};
</script>
<style scoped>
#video1 {
  width: 500px;
  height: 400px;
  background-color: grey;
}
.currentEmoji {
  font-size: 120px;
}
.user-details {
  color: white;
}
</style>