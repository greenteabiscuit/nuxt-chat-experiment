<template>
  <v-app>
    <v-app-bar app>
      <v-toolbar-title class="headline text-uppercase">
        <span class="font-weight-light">MY CHATROOM</span>
      </v-toolbar-title>
      <v-spacer></v-spacer>
      <v-btn v-if="logined" secondary @click="googleLogout">
        <v-icon>mdi-google</v-icon>:logout
      </v-btn>
    </v-app-bar>
    <v-content>
      <v-container v-if="!logined" class="fill-height" fluid>
        <v-row align="center" justify="center">
          <v-col cols="12" sm="8" md="4">
            <v-card class="elevation-12">
              <v-toolbar color="primary" dark flat>
                <v-toolbar-title>ログイン</v-toolbar-title>
              </v-toolbar>
              <v-card-text>Googleアカウントでログインしてください。</v-card-text>
              <v-card-actions>
                <v-spacer></v-spacer>
                <v-btn color="primary" @click="googleLogin()">Google Login</v-btn>
                <v-spacer></v-spacer>
              </v-card-actions>
            </v-card>
          </v-col>
        </v-row>
      </v-container>

      <v-container v-if="logined" fluid transition="scroll-x-transition">
        <v-row align="start" justify="center">
          <v-col xs="12" sm="6" :class="xsDisplayRooms">
            <v-list dense>
              <v-list-item>
                <v-text-field v-model="inputRoomName" :counter="18" label="Room Name" required></v-text-field>
                <v-btn color="primary" :disabled="!inputRoomName" @click="createRoom">部屋を作る</v-btn>
              </v-list-item>
              <v-list-item
                v-for="item in rooms"
                :key="item.id"
                @click="roomId = item.id;roomName = item.name"
              >
                <v-list-item-content>
                  <v-list-item-title>
                    {{item.name}}
                    <v-icon class="float-right">mdi-chevron-right</v-icon>
                  </v-list-item-title>
                </v-list-item-content>
              </v-list-item>
            </v-list>
          </v-col>
          <v-col xs="12" sm="6">
            <v-card :loading="cardLoading">
              <v-card-title>
                <v-icon
                  v-if="$vuetify.breakpoint.xs"
                  class="mr-2"
                  @click="roomId = null"
                >mdi-arrow-left</v-icon>
                {{roomName}}
              </v-card-title>
              <v-card-text id="messageArea" style="height:60vh;overflow:scroll">
                <v-list dense rounded>
                  <v-list-item v-for="item in messages" :key="item.id">
                    <v-list-item-content :class="alignMessage(item.userName)" color="primary">
                      <v-list-item-title class="py-4">{{item.message}}</v-list-item-title>
                      <v-list-item-subtitle>{{item.userName + '('+displayTime(item.createdAt.seconds)+ ')'}}</v-list-item-subtitle>
                    </v-list-item-content>
                  </v-list-item>
                </v-list>
              </v-card-text>
              <v-spacer></v-spacer>
              <v-card-actions v-if="roomId">
                <v-text-field
                  v-model="inputMessage"
                  max="200"
                  label="Message"
                  required
                  :loading="messageLoading"
                  :disabled="messageLoading"
                  @keydown.enter="enterAddMessage"
                ></v-text-field>
                <v-btn color="primary" :disabled="!inputMessage" @click="addMessage">送信</v-btn>
              </v-card-actions>
            </v-card>
          </v-col>
        </v-row>
      </v-container>
    </v-content>
  </v-app>
</template>

<script>
import moment from "moment";
import firebase from "firebase/app";
import "firebase/firestore";
import "firebase/auth";

const firebaseConfig = {
  apiKey: "*************************",
  authDomain: "*************************",
  databaseURL: "*************************",
  projectId: "*************************",
  storageBucket: "*************************",
  messagingSenderId: "*************************",
  appId: "*************************"
};
firebase.initializeApp(firebaseConfig);
const firebaseApp = firebase.firestore();
const firebaseAuth = firebase.auth();

export default {
  name: "App",
  props: {
    source: String
  },
  data: () => ({
    drawer: null,
    logined: false,
    inputRoomName: null,
    roomId: null,
    rooms: [],
    roomName: null,
    messages: [],
    inputMessage: null,
    userName: null,
    messageLoading: false,
    cardLoading: false
  }),
  computed: {
    xsDisplayRooms() {
      if (this.$vuetify.breakpoint.xs) {
        if (this.roomId) {
          return "overlay-slide overlay-out";
        } else {
          return "overlay-slide";
        }
      } else {
        return "";
      }
    }
  },
  watch: {
    roomId() {
      if (this.roomId) {
        this.cardLoading = true;
        firebaseApp
          .collection("room")
          .doc(this.roomId)
          .collection("messages")
          .orderBy("createdAt")
          .onSnapshot(res => {
            this.messages = [];
            res.docs.forEach(elem => {
              const room = elem.data();
              this.messages.push(room);
              this.$nextTick(() => {
                this.scrollBottom();
              });
            });
            this.cardLoading = false;
          });
      }
    }
  },
  mounted() {
    firebaseAuth.onAuthStateChanged(user => {
      console.log(user);
      if (!user) {
        this.deleteLoginUser();
      } else {
        this.userName = user.displayName;
        this.logined = true;
        firebaseApp
          .collection("room")
          .orderBy("createdAt", "desc")
          .onSnapshot(res => {
            this.rooms = [];
            res.docs.forEach(elem => {
              const room = elem.data();
              room.id = elem.id;
              this.rooms.push(room);
            });
          });
      }
    });
  },
  methods: {
    displayTime(timestamp) {
      return moment(timestamp * 1000).format("YYYY/MM/DD HH:mm");
    },
    alignMessage(messageName) {
      if (messageName === this.userName) {
        return "text-right";
      }
    },
    enterAddMessage() {
      if (event.keyCode !== 13) return;
      this.addMessage();
    },
    createRoom() {
      const name = this.inputRoomName;
      if (name) {
        firebaseApp
          .collection("room")
          .add({
            name,
            createdAt: new Date(),
            createUser: this.userName
          })
          .then(res => {
            this.name = null;
          });
      }
    },
    addMessage() {
      this.messageLoading = true;
      const message = this.inputMessage;
      if (message) {
        console.log("add");
        firebaseApp
          .collection("room")
          .doc(this.roomId)
          .collection("messages")
          .add({
            message,
            createdAt: new Date(),
            userName: this.userName
          })
          .then(res => {
            this.inputMessage = null;
          })
          .finally(() => {
            this.messageLoading = false;
          });
      }
    },
    scrollBottom() {
      const container = this.$el.querySelector("#messageArea");
      container.scrollTop = container.scrollHeight;
    },
    googleLogin() {
      const provider = new firebase.auth.GoogleAuthProvider();
      firebaseAuth
        .signInWithPopup(provider)
        .then(result => {
          this.userName = result.user.displayName;
          this.logined = true;
        })
        .catch(error => {
            console.log(error)
          this.deleteLoginUser();
        });
    },
    googleLogout() {
      firebaseAuth.signOut().finally(() => {
        this.deleteLoginUser();
      });
    },
    deleteLoginUser() {
      this.userName = null;
      this.logined = false;
    }
  }
};
</script>
<style lang="scss">
.overlay-slide {
  position: absolute;
  z-index: 100;
  left: auto;
  right: auto;
  height: 100vh;
  background: #fff;
  transition: transform 0.5s ease;
}
.overlay-out {
  transform: translate(-100%);
}
</style>