<template>
  <div>
    <h1>Smart Axe!</h1>
    <p class="sub-header">The sharpest tool in the shed.</p>

    <template v-if="tournaments.length > 0">
      <h2>Recent Tournaments</h2>
      <div class="container">
        <template v-for="tournament in tournaments">

            <div @click="continueTournament(tournament)" v-bind:class="[tournament.target === 'WATL' ? 'watl-target' : 'natf-target', 'organization']" >
              <button class="btn tournament">Continue Tournament</button>
            </div>

        </template>
      </div>
    </template>

    <h2>Start a New Tournament</h2>
    <div class="container">
      <div @click="newTournament('WATL')" class="organization watl-target">
        <button class="btn tournament">New WATL Tournament</button>
      </div>

      <div @click="newTournament('NATF')" class="organization natf-target">
        <button class="btn tournament">New NATF Tournament</button>
      </div>
    </div>

  </div>
</template>


<script>
import store from '@/store'

function newTournament (target) {
  store
    .create('tournament', { target: target, started: new Date() })
    .then(tournament => {
      continueTournament.call(this, tournament.payload.records[0])
    })
    .catch(error => { this.$emit('error', error) })
}

function continueTournament (tournament) {
  this.$router.push({ name: 'throwers', params: { tournamentId: tournament.id } })
}

function loadTournaments (to, from, next) {
  store
    .find('tournament', null, { sort: { started: true }, limit: 5 })
    .then(result => {
      next(vm => { vm.tournaments = result.payload.records })
    })
    .catch(error => { this.$emit('error', error) })
}

function data () {
  let model =
    { tournaments: []
    }

  return model
}

let result =
  { beforeRouteEnter: loadTournaments
  , beforeRouteUpdate: loadTournaments
  , data: data
  , methods:
    { newTournament: newTournament
    , continueTournament: continueTournament
    }
  }

export default result
</script>


<style scoped>
  h1 {
    margin-bottom: 0;
  }

  img {
    display: block;
  }

  .container {
    display: flex;
    justify-content: center;
    cursor: pointer;
  }

  .sub-header {
    margin-bottom: 3em;
  }

  .organization {
    width: 200px;
    height: 200px;
    position: relative;
    background-repeat: no-repeat;
    background-position: center;
    background-size: contain;
    margin-bottom: 30px;
    border: 1px solid #ededed;
    border-radius: 10px;
    margin: 10px;
  }

  .natf-target {
    background-image: url('~/static/NATF-target.png')
  }

  .watl-target {
    background-image: url('~/static/WATL-target.png')
  }

  .tournament {
    width:100%;
    position:absolute;
    bottom: 0;
  }

</style>
