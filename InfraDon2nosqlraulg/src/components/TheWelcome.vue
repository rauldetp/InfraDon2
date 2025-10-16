<script setup lang="ts">
import { ref, onMounted } from 'vue'
import PouchDB from 'pouchdb'

// Définition du type utilisateur (structure de ton document CouchDB)
interface UserDoc {
  _id?: string
  _rev?: string
  type: string
  name: {
    first: string
    last: string
  }
  email: string
  tags: string[]
  created_at: string
}

// Base typée CouchDB
const db = ref<PouchDB.Database<UserDoc> | null>(null)
const users = ref<UserDoc[]>([])

// Connexion à CouchDB
function connectToCouchDB(): void {
  try {
    const database: PouchDB.Database<UserDoc> = new PouchDB('http://admin:MbappeVini135@127.0.0.1:5984/vini')
    db.value = database
    console.log('✅ Connecté à CouchDB')
  } catch (error) {
    console.error('❌ Erreur de connexion :', error)
  }
}

// Lecture des utilisateurs
async function fetchUsers(): Promise<void> {
  if (!db.value) {
    console.warn('⚠️ Base non initialisée')
    return
  }

  try {
    const result = await db.value.allDocs({ include_docs: true })
    users.value = result.rows
      .filter(row => !!row.doc)
      .map(row => row.doc as UserDoc)

    console.log('✅ Utilisateurs récupérés :', users.value)
  } catch (error) {
    console.error('❌ Erreur de lecture CouchDB :', error)
  }
}

// Initialisation automatique
onMounted(() => {
  connectToCouchDB()
  setTimeout(fetchUsers, 500)
})
</script>

<template>
  <section style="padding:2rem;background:#f9f9f9;color:black;">
    <h1>Connexion à CouchDB</h1>

    <div v-if="users.length === 0">
      <p>Aucune donnée trouvée dans la base.</p>
    </div>

    <article
      v-for="user in users"
      :key="user._id"
      style="margin-bottom:1rem;padding:1rem;border:1px solid #ccc;border-radius:8px;background:white;color:black;"
    >
      <h2>{{ user.name.first }} {{ user.name.last }}</h2>
      <p><strong>Email :</strong> {{ user.email }}</p>
      <p><strong>Tags :</strong> {{ user.tags.join(', ') }}</p>
      <small><strong>Créé le :</strong> {{ user.created_at }}</small>
    </article>
  </section>
</template>

<style scoped>
h1 {
  color: #333;
  margin-bottom: 1rem;
}
</style>