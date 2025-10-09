<script lang="ts">
import { defineComponent } from 'vue'
import PouchDB from 'pouchdb'

export default defineComponent({
  name: 'TestDatabase',

  data() {
    return {
      counter: 110
    }
  },

  methods: {
    increment() {
      this.counter++
    },

    initDatabase() {
      // Connexion à CouchDB (adapter l’URL selon ta base)
      const db = new PouchDB('http://admin:admin@localhost:5984/infra_53_0850')

      if (db) {
        console.log('Connexion réussie à CouchDB')
      } else {
        console.error('Erreur : objet DB nul')
      }

      // Vérifie les informations de la base
      db.info()
        .then((info: PouchDB.Core.DatabaseInfo) => {
          console.log('Informations de la base :', info)
        })
        .catch((err: Error) => {
          console.error('Erreur lors de la lecture :', err.message)
        })
    }
  },

  mounted() {
    // Appel automatique quand le composant est monté
    this.initDatabase()
  }
})
</script>

<template>
  <h1>SALUT LE JIIIIIIISTRERIIIIEWWWW</h1>
  <p>Counter {{ counter }}</p>
  <button @click="increment">Increment</button>
</template>