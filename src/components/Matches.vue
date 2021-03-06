<template>
  <div>
    <h1>Start Match</h1>

    <div class="container">
      <h2>Select Throwers</h2>
      <label class="form-checkbox">
        <input type="checkbox" @change="toggleAllThrowers()" v-model="selectedAllThrowers"/>
        <i class="form-icon"></i> [select all]
      </label>

      <div v-for="thrower in throwers">
        <label class="form-checkbox">
          <input type="checkbox" :value="thrower" v-model="selectedThrowers"/>
          <i class="form-icon"></i> {{thrower.name}}
        </label>
      </div>
    </div>

    <div class="container">
      <h2>Select Match Type</h2>
      <div v-for="game in gameData">
        <label class="form-radio">
          <input type="radio" name="game" :value="game" v-model="selectedGame" />
          <i class="form-icon"></i> {{game.title}}
        </label>
      </div>
    </div>

    <div class="container" v-if="selectedGame && selectedGame.allowMultipleThrows">
      <h2>Select Number of Throws Per Turn</h2>
      <div class="form-group">
        <select class="form-select" v-model="selectedThrows">
          <option value="1">1</option>
          <option value="2" selected="selected">2</option>
          <option value="3">3</option>
        </select>
      </div>
    </div>

    <div class="container" v-if="selectedGame && selectedGame.allowTeams && selectedThrowers.length > 2">
      <h2>Select Teams?</h2>
      <label class="form-radio">
        <input type="radio" name="form-team" checked @change="formTeams(false)"/>
        <i class="form-icon"></i> No, throw as individuals.
      </label>
      <label class="form-radio">
        <input type="radio" name="form-team" @change="formTeams(true)"/>
        <i class="form-icon"></i> Yes, create two teams.
      </label>

      <div v-if="teams" class="split-container">
        <div v-for="team in teams" class="split-item">
          <input class="form-input input-lg" type="text" placeholder="Name" v-model="team.name"/>
          <div v-for="thrower in team.throwers" class="team-thrower" @click="moveToTeam(thrower, team.id)">
            {{thrower.name}}
            <icon :name="team.id === 1 ? 'arrow-circle-right' : 'arrow-circle-left'" label="move" scale=1></icon>
          </div>
        </div>
      </div>
    </div>

    <button class="btn" @click="createMatch()" v-if="isMatchReady()">Start Match</button>

  </div>
</template>


<script>
import store from '@/store'
import games from '@/components/games'
import AppError from '@/errors/AppError.js'

let gameData = Object.getOwnPropertyNames(games)
  .filter(name => name !== '$keys')
  .map(key => {
    let game = games[key].data()
    game.id = key
    return game
  })

function routeEnter (to, from, next) {
  let options = { match: { tournament: to.params.tournamentId } }
  store
    .find('thrower', null, options)
    .then(result => {
      next(vm => {
        if (result.payload.records.length === 0) return vm.$emit('error', new AppError('Could not load throwers', 'throwers'))
        vm.throwers = result.payload.records
        vm.selectedThrowers = vm.throwers
      })
    })
    .catch(error => {
      console.log(error)
      next(vm => vm.$emit('error', new AppError('Could not load throwers', 'throwers')))
    })
}

function created () {
  this.tournamentId = this.$route.params.tournamentId
  this.$watch('selectedThrowers', () => {
    this.selectedAllThrowers = this.selectedThrowers.length === this.throwers.length
  }, { deep: true })
}

function toggleAllThrowers () {
  if (this.selectedAllThrowers) {
    this.selectedThrowers = this.throwers.map(thrower => { return thrower })
  } else {
    this.selectedThrowers = []
  }
}

function formTeams (shouldFormTeams) {
  this.teams = shouldFormTeams ? [ {id: 1, name: 'Team 1', throwers: []}, {id: 2, name: 'Team 2', throwers: []} ] : null
  if (!this.teams) return

  this.selectedThrowers.forEach((thrower, index) => {
    this.teams[index % 2].throwers.push(thrower)
  })
}

function moveToTeam (thrower, teamId) {
  let toIndex = teamId === 1 ? 1 : 0
  this.teams[toIndex].throwers.push(thrower)

  let fromIndex = teamId - 1
  this.teams[fromIndex].throwers = this.teams[fromIndex].throwers.filter(t => { return t !== thrower })
}

function isMatchReady () {
  /* eslint-disable operator-linebreak */
  return this.selectedGame
    && this.selectedThrowers.length > 0
    && (!this.teams || Math.abs(this.teams[0].throwers.length - this.teams[1].throwers.length) <= 1)
}

function createMatch () {
  store.create('match',
    { game: this.selectedGame.id
    , tournament: this.tournamentId
    , throwers: this.selectedThrowers.map(t => t.id)
    }
  ).then(result => createRound.call(this, result.payload.records[0]))
}

function createRound (match) {
  let teams = this.teams || []
  let team1 = !teams[0] ? null : { id: teams[0].id, name: teams[0].name, throwers: teams[0].throwers.map(t => t.id) }
  let team2 = !teams[1] ? null : { id: teams[1].id, name: teams[1].name, throwers: teams[1].throwers.map(t => t.id) }

  store.create('round',
    { tournament: this.tournamentId
    , match: match.id
    , turnSize: parseInt(this.selectedThrows)
    , throwers: this.selectedThrowers.map(t => t.id)
    , team1: team1
    , team2: team2
    , started: new Date()
    }
  ).then(result => {
    let round = result.payload.records[0]
    this.$router.push({ name: 'rounds', params: { matchId: match.id, roundId: round.id } })
  })
}

function data () {
  let model =
    { selectedGame: null
    , selectedThrowers: []
    , selectedAllThrowers: false
    , selectedThrows: 2
    , throwers: []
    , teams: null
    , gameData
    }

  return model
}

let result =
  { beforeRouteEnter: routeEnter
  , beforeRouteUpdate: routeEnter
  , tournamentId: null
  , data
  , created
  , methods:
    { toggleAllThrowers
    , createMatch
    , isMatchReady
    , formTeams
    , moveToTeam
    }
  }

export default result
</script>


<style scoped>

.container {
  margin-bottom: 30px;
  display: block;
}

.split-container {
  display: flex;
  flex-direction: row;
  margin-bottom: 30px;
}

.split-item {
  display: flex;
  flex-direction: column;
  width: 50%;
  padding: 10px;
}

.team-thrower {
  font-size: 1.5em;
  margin-top: .5em;
}

</style>
