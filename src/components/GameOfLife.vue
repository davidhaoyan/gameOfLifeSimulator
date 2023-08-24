  <template>
      <div class="gol">
      <div v-for="n in colSize" class="flex-container">
        <div class="dead"
             v-for="m in rowSize"
             :id="'box.' + n + '.' + m"
             v-on:mouseenter="select" v-on:mousedown="enableToggle" v-on:mouseup="disableToggle"></div>
      </div>
    </div>
    <button icon="fa-solid fa-sliders" v-on:click="show=true" id="show-sidebar" v-if="!show">
      Toggle sidebar
    </button>
    <div id="sidebar" :style="{ transform: show ? 'translateX(0)' : 'translateX(-100%)'}">
      <button id="hide-sidebar" v-on:click="show=false"> Toggle sidebar </button>
      <button class="play-pause-button" id="pause-button" v-on:click="pause" v-if="isPlaying">Pause</button>
      <button class="play-pause-button" id="play-button" v-on:click="play" v-else>Play</button>
      <span id="turn-counter">Turn 0</span>
      <div class="seed-picker">
        <select v-model="selected" style="width: 150px" @change="onChange($event)">
          <option disabled value="">Select seed</option>
          <option v-for="seed in seeds" v-bind:value="seed.value">
            {{ seed.text}}
          </option>
        </select>
      </div>
      <input type="range" min="50" max="300" value="50" id="slider"/>
      <span id="slider-value"> Timeout {{ this.timeout}} </span>
      <div id="text-box">
        {{ this.info }}
      </div>
    </div>
  </template>

  <script>
  import config from '../config.js'
  const localServerUrl = config.localServerUrl
  export default {
    name: 'GameOfLife',
    data() {
      return {
        rowSize: this.findRowSize(),
        colSize: this.findColSize(),
        turn: 0,
        turns: -1,
        isPlaying: 0,
        pauseWaiting: 0,
        isToggling: 0,
        data: {},
        quickData: {},
        fetchCounter: 0,
        initialWorld: [[]],
        usingRLE: false,
        selected: localStorage.getItem('seedCache') ? localStorage.getItem('seedCache') : '',
        seeds: [],
        timeout: 50,
        show: true,
        info: 'Loading...',
      }
    },
    methods: {
      onChange(event) {
        localStorage.setItem('seedCache', event.target.value)
        location.reload()
        this.rle(localStorage.getItem('seedCache') ? localStorage.getItem('seedCache') : event.target.value)
      },
      rle(seed) {
        this.usingRLE = true
        const url = localServerUrl + '/api_rle'
        fetch(url, {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json'
          },
          body: JSON.stringify({
            seed: seed
          })
        })
        .then((response) => {
          if (!response.ok) {
            throw new Error('Network response was not ok.');
          }
          return response.json(); // If expecting JSON response
        })
        .then((responseData) => {
          let initialiseArray = responseData.data
          let world = this.createEmptyWorld()
          for (let y = 0; y < this.colSize; y++) {
            for (let x = 0; x < this.rowSize; x++) {
              const e = document.getElementById('box.' + (y+1) + '.' + (x+1))
              e.className = 'dead'
            }
          }
          for (let i = 0; i < Object.keys(initialiseArray).length; i++) {
            let x = initialiseArray[i]['X']
            let y = initialiseArray[i]['Y']
            const e = document.getElementById('box.' + (y+1+40) + '.' + (x+1+110))
            e.className = 'alive'
            world[y+40][x+110] = 1
            this.initialWorld = world
            this.info = responseData.info
          }
        })
        .catch((error) => {
          console.error('Fetch error:', error);
        });
      },
      findRowSize() {
        return (Math.floor(window.innerWidth/6) - 4)
      },
      findColSize() {
        return (Math.floor(window.innerHeight/6) - 6)
      },
      createEmptyWorld() {
        let world = new Array(this.colSize)
        for (let i = 0; i < this.colSize; i++) {
          world[i] = new Array(this.rowSize).fill(0)
        }
        return world
      },
      roundTurn() { //rounds turn up to the nearest 10
        let x = this.turn
        let y = 0
        while (x > 10) {
          x -= 10
          y++
        }
        if (this.turn == 0) return 0
        else return (y*10) + 10
      },
      async fetch() {
        let world;
        if (this.turn == 0) {
          if (this.usingRLE) {
            world = this.initialWorld
          }
          else {
            world = this.createEmptyWorld()
                for (let y = 0; y < this.colSize; y++) {
                  for (let x = 0; x < this.rowSize; x++) {
                if (document.getElementById('box.' + (y+1) + '.' + (x+1)).className == 'alive') {
                  world[y][x] = 1
                }
              }
            }
          }
        }
        else {
          world = this.data[this.roundTurn()-1]
        }
        const url = localServerUrl + '/api_gol'
        fetch(url, {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json'
          },
          body: JSON.stringify({
            world: world,
            turn: this.roundTurn(),
          })
        })
          .then((response) => {
            if (!response.ok) {
              throw new Error('Network response was not ok.');
            }
            return response.json(); // If expecting JSON response
          })
          .then((responseData) => {
            let worldArray = JSON.parse(responseData.data)
            let quickArray = JSON.parse(responseData.quickData)
            let turns = Object.keys(worldArray).length
            for (let t = 0; t < turns; t++) {
              this.data[this.fetchCounter*10+t] = worldArray[(this.fetchCounter*10)+t]
            }
            for (let t = 0; t < turns; t++) {
              this.quickData[this.fetchCounter*10+t] = quickArray[(this.fetchCounter*10)+t]
            }
            this.turns = responseData.turns
            if (this.turn == 0) {
              this.play()
            }
          })
          .catch((error) => {
            console.error('Fetch error:', error);
          });
      },
      play() {
        if (this.turn == 0) {
          this.fetch()
        }
        if ((this.turn % 10) - 3 == 0) {
          this.fetch()
          this.fetchCounter++
        }
        document.getElementById('turn-counter').innerHTML = 'Turn ' + this.turn
        const world = this.data[this.turn]
        if (!this.pauseWaiting) { this.isPlaying = 1}
        for (let c = 0; c < Object.keys(this.quickData[this.turn]).length; c++) {
          let x = this.quickData[this.turn][c]['X']
          let y = this.quickData[this.turn][c]['Y']
          const e = document.getElementById('box.' + (y+1) + '.' + (x+1))
          if (world[y][x] == 1) {
            e.className = 'alive'
          }
          else {
            e.className = 'dead'
          }
        }
        this.turn++
        if ((this.turn < this.turns) && (!this.pauseWaiting)) {
          setTimeout(this.play, this.timeout)
        }
        this.pauseWaiting = 0
      },
      pause() {
        this.isPlaying = 0
        this.pauseWaiting = 1
      },
      select(event) {
        if (!this.isToggling) {return}
        if (this.turn != 0) {return}
        const targetId = event.currentTarget.id
        const e = document.getElementById(targetId)
        if (e.className == 'dead') {e.className = 'alive'}
        else {e.className = 'dead'}
        const y = targetId.split(".")[1]
        const x = targetId.split(".")[2]
        this.initialWorld[y-1][x-1] = 1
      },
      enableToggle(event) {
        this.isToggling = 1
        this.select(event)
      },
      disableToggle() {
        this.isToggling = 0
      },
    },
    mounted() {
      if (localStorage.getItem('seedCache')) {
        this.rle(localStorage.getItem('seedCache'))
      }
      const slider = document.getElementById("slider")
      slider.addEventListener("input", () => {
        const sliderValue = slider.value
        this.timeout = sliderValue
      })
      this.initialWorld = this.createEmptyWorld()
      const url = localServerUrl + '/api_seed'
      fetch(url, {
        method: 'GET',
      })
      .then((response) => {
        if (!response.ok) {
          throw new Error('Network response was not ok.');
        }
        return response.json(); // If expecting JSON response
      })
      .then((responseData) => {
        const fetchedSeeds = responseData.seeds.map(seed => ({
          text: seed,
          value: seed
        }));
        this.seeds = fetchedSeeds;
      })
      .catch((error) => {
        console.error('Fetch error:', error);
      });
    }
  }
  </script>

  <style>
  .flex-container {
    display: flex;
    flex-direction: row;
    justify-content: center;
  }
  .gol {
    position: fixed;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%)
  }
  .play-pause-button {
    position: absolute;
    top: 10%;
    left: 50%;
    transform: translate(-50%, -50%);
    background-color: #76bd6a;
  }
  #turn-counter {
    position: absolute;
    top: 15%;
    left: 50%;
    transform: translate(-50%, -50%);
    color: #d5d9e8;
  }
  #slider {
    position: absolute;
    top: 20%;
    left: 50%;
    transform: translate(-50%, -50%);
  }
  #slider-value {
    position: absolute;
    top: 25%;
    left: 50%;
    transform: translate(-50%, -50%);
    color: #d5d9e8;
  }
  .dead {
    background-color: #5b5e69;
    height: 5px;
    width: 5px;
    margin: 0.4px;
    transition: background-color 0.1s;
  }
  .alive {
    background-color: #d5d9e8;
    height: 5px;
    width: 5px;
    margin: 0.4px;
    transition: background-color 0.1s;
  }
  .seed-picker {
    position: absolute;
    top: 30%;
    left: 50%;
    transform: translate(-50%, -50%);
  }
  #sidebar {
    position: absolute;
    top: 0px;
    left: 0px;
    background-color: #4d5057;
    width: 175px;
    height: 100%;
    transform: translateX(0);
    transition: transform 0.3s ease
  }
  #show-sidebar {
    position: absolute;
    top: 5px;
    left: 5px;
    background-color: #d5d9e8;
  }
  #hide-sidebar {
    position: absolute;
    top: 5px;
    left: 5px;
    background-color: #d5d9e8;
  }
  #text-box {
    position: absolute;
    background-color: #52555e;
    width: 165px;
    height: 300px;
    top: 35%;
    left: 50%;
    transform: translate(-50%, 0);
    color: #d5d9e8;
  }
  </style>