<template>
  <div v-if="discordUsername">
    <div class="title-bar">
      <div class="title-left">
        <div class="title-text">Naga Dao</div>
      </div>

      <div class="title-right" @click="logout()">{{ discordUsername }}</div>
    </div>

    <div class="loading" v-if="loading"></div>

    <div class="content-area">
      <div class="part-select-container">
        <div
          :class="'part-select ' + (parts[activePartIndex] == part ? 'active' : '')"
          v-for="(part, i) in parts"
          :key="part"
          @click="setActivePart(i)"
        >
          {{ part }}
        </div>
      </div>

      <div class="voting-container">
        <div class="voting-card card" v-for="i in variantCount" :key="i">
          <div class="card-img-top voting-card-img-container">
            <img :src="buildPartImgSrc(j - 1, j == activePartIndex + 1 ? i : 1)" class="voting-card-img" v-for="j in activePartIndex + 1" :key="j">
          </div>

          <div class="card-body">
            <div class="voting-action-btn-container">
              <button :class="['btn', 'btn-success', rarityVote[parts[activePartIndex] + '_' + i] || rarityCount[parts[activePartIndex]] >= 20 ? 'disabled' : '']" @click="voteRarityOnClick(i)">+1</button>
              <button :class="['btn', 'btn-success', formattedPartData[parts[activePartIndex] + '_' + i] && formattedPartData[parts[activePartIndex] + '_' + i].length ? '' : '']" @click="voteNameOnClick(i)">Name It</button>
            </div>

            <div v-if="formattedPartData[parts[activePartIndex] + '_' + i]">
              <div class="voting-name-container" v-for="(name, j) in formattedPartData[parts[activePartIndex] + '_' + i]" :key="j">
                <div class="voting-name">{{name.name}}</div>
                <div class="voting-discord-username">{{name.discordUsername}}</div>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>

    <div class="loading-modal" v-if="loading"></div>
  </div>

  <div v-else>
    Please refresh this page when you are ready
  </div>
</template>

<style>
html, body {
  height: 100%;
  background-color: rgb(233, 252, 233);
  color: rgb(0, 66, 23)
}

.title-bar {
  padding: 16px;
  color: white;
  background-color: rgb(36, 106, 60);

  display: flex;
}

.title-left {
  flex-grow: 1;
}

.title-right {
  display: flex;
  align-items: center;
}

.title-text {
  font-size: 1.5rem;
}

.content-area {
  padding: 16px;
}

.part-select-container {
  display: flex;
  flex-wrap: wrap;
}

.part-select {
  border: 2px solid rgb(36, 106, 60);
  border-radius: 10px;
  padding: 4px 16px;
  margin-right: 12px;

  background-color: rgb(194, 247, 194);

  box-shadow: rgba(0, 0, 0, 0.24) 0px 3px 8px;
}

.part-select:not(.active):hover {
  cursor: pointer;
  background-color: rgb(170, 247, 170);
}

.part-select.active {
  background-color: rgb(36, 106, 60);
  color: white;
}

.voting-container {
  display: flex;
  justify-content: center;
  flex-wrap: wrap;
  margin-top: 16px;
}

.voting-card {
  width: 200px;
  margin: 8px;
}

.voting-card-img-container {
  position: relative;
}

.voting-card-img:first-child {
  width: 100%;
}

.voting-card-img:not(:first-child) {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
}

.voting-card-img:not(:last-child) {
  opacity: 0.4;
}

.voting-action-btn-container {
  display: flex;
  justify-content: center;
}

.voting-action-btn-container button {
  margin: 0px 8px;
}

.voting-name-container {
  margin-top: 8px;
}

.voting-discord-username {
  color: grey;
  font-size: 0.8rem;
}
</style>

<script>
// import HelloWorld from "./components/HelloWorld.vue";
import { initializeApp } from "firebase/app";
import { getFirestore, collection, addDoc, query, where, onSnapshot } from "firebase/firestore";
import _ from "lodash";
// import { getAnalytics } from "firebase/analytics";

const firebaseConfig = {
  apiKey: "AIzaSyDmqtyt-orVgzSCRFFTuPwpUj1FFBs5n7Q",
  authDomain: "naga-dao.firebaseapp.com",
  projectId: "naga-dao",
  storageBucket: "naga-dao.appspot.com",
  messagingSenderId: "878698903485",
  appId: "1:878698903485:web:0433035736b4914ce484d4",
  measurementId: "G-Z4J7B6M8W3"
};

// Initialize Firebase
const app = initializeApp(firebaseConfig);
// const analytics = getAnalytics(app);
const db = getFirestore();

const PARTS = ["Bg", "Body", "Cheek", "Eye", "Mouth", "Head", "Clothes"];

const VARIANT_COUNT = 100;

export default {
  name: "App",
  data() {
    let discordUsername = window.localStorage.getItem("DISCORD_USERNAME");

    if (!discordUsername) {
      discordUsername = window.prompt("Please enter your discord username (Ex: Chom#1652)");

      if (!discordUsername) {
        window.alert("Please refresh this page when you are ready");
      }
    }

    if (discordUsername) {
      window.localStorage.setItem("DISCORD_USERNAME", discordUsername)
    }

    return {
      loading: false,
      parts: PARTS,
      variantCount: VARIANT_COUNT,
      discordUsername: discordUsername,
      activePartIndex: 0,
      rawPartData: [],
      formattedPartData: {},

      rawRarityData: [],
      rarityVote: {},
      rarityCount: {},
    };
  },
  async mounted() {
    this.listenPartName();
    this.listenPartRarity();
  },
  components: {
    // HelloWorld,
  },
  methods: {
    async listenPartName() {
      const q = query(collection(db, "part-name"));
      const unsubscribe = onSnapshot(q, (querySnapshot) => {
        const rawPartData = [];
        querySnapshot.forEach((doc) => {
          rawPartData.push(doc.data());
        });

        this.rawPartData = rawPartData;
        this.formattedPartData = _.groupBy(rawPartData, 'key');
      });
    },

    async listenPartRarity() {
      const q = query(collection(db, "part-rarity"), where("discordUsername", "==", this.discordUsername));
      const unsubscribe = onSnapshot(q, (querySnapshot) => {
        const rawRarityData = [];
        querySnapshot.forEach((doc) => {
          rawRarityData.push(doc.data());
        });

        this.rawRarityData = rawRarityData;
        
        let rarityVote = {};
        let rarityCount = {};

        for (let rarity of rawRarityData) {
          if (!rarityCount[rarity.part]) rarityCount[rarity.part] = 0;
          rarityVote[rarity.key] = true;
          rarityCount[rarity.part]++;
        }

        // console.log(rawRarityData);
        // console.log(rarityVote);
        // console.log(rarityCount);

        this.rarityVote = rarityVote;
        this.rarityCount = rarityCount;
      });
    },

    setActivePart(i) {
      this.activePartIndex = i;
    },

    async voteName(key, name) {
      this.loading = true;

      try {
        const docRef = await addDoc(collection(db, "part-name"), {
          key,
          part: key.split('_')[0],
          variant: parseInt(key.split('_')[1]),
          name,
          discordUsername: this.discordUsername,
        });
        console.log("Document written with ID: ", docRef.id);
      } finally {
        this.loading = false;
      }
    },

    async voteNameOnClick(variantId) {
      let key = this.parts[this.activePartIndex] + "_" + variantId;

      let name = window.prompt('Please enter name for this part');

      if (!name) return;

      if (!/^[A-Za-z0-9 ]*$/.test(name)) {
        window.alert("Please enter english name only");
        return;
      }

      let formattedName = "";

      formattedName += name[0].toUpperCase();
      for (let i = 1; i < name.length; i++) {
        if (name[i-1] == ' ') {
          formattedName += name[i].toUpperCase();
        } else {
          formattedName += name[i];
        }
      }

      try {
        await this.voteName(key, formattedName);
      } catch (err) {
        console.error(err);
        alert("Error");
      }
    },

    async voteRarity(key) {
      this.loading = true;

      try {
        const docRef = await addDoc(collection(db, "part-rarity"), {
          key,
          part: key.split('_')[0],
          variant: parseInt(key.split('_')[1]),
          discordUsername: this.discordUsername,
        });
        console.log("Document written with ID: ", docRef.id);
      } finally {
        this.loading = false;
      }
    },

    async voteRarityOnClick(variantId) {
      let key = this.parts[this.activePartIndex] + "_" + variantId;

      if (!window.confirm('Confirm vote this part')) return;

      try {
        await this.voteRarity(key);
      } catch (err) {
        console.error(err);
        alert("Error");
      }
    },

    buildPartImgSrc(partId, variantId) {
      return `./img/parts/${partId + 1}_${this.parts[partId]}/${this.parts[partId]}_${variantId}.png`;
    },

    logout() {
      if (confirm("Confirm logout?")) {
        window.localStorage.removeItem("DISCORD_USERNAME");
        window.location.reload();
      }
    }
  },
};
</script>
