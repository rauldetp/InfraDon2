<script setup lang="ts">
import { ref, onMounted } from 'vue'
import PouchDB from 'pouchdb'

interface UserDoc {
  _id?: string
  _rev?: string
  type?: string
  name: {
    first: string
    last: string
  }
  email: string
  tags: string[]
  created_at: string
}

const db = ref<PouchDB.Database<UserDoc> | null>(null)
const users = ref<UserDoc[]>([])

// Modèle pour le formulaire
const newUser = ref<UserDoc>({
  type: 'user',
  name: { first: '', last: '' },
  email: '',
  tags: [],
  created_at: ''
})

function connectToCouchDB(): void {
  try {
    const database: PouchDB.Database<UserDoc> = new PouchDB('http://admin:MbappeVini135@127.0.0.1:5984/vini')
    db.value = database
    console.log('Connecté à CouchDB')
  } catch (error) {
    console.error('Erreur de connexion :', error)
  }
}

async function fetchUsers(): Promise<void> {
  if (!db.value) {
    console.warn('Base non initialisée')
    return
  }

  try {
    const result = await db.value.allDocs<UserDoc>({ include_docs: true })
    const rows = result.rows.filter(row => row.doc)
    users.value = rows.map(row => row.doc as UserDoc)
  } catch (error) {
    console.error('Erreur de lecture CouchDB :', error)
  }
}

// Fonction pour ajouter un utilisateur
async function addUser(): Promise<void> {
  if (!db.value) return

  if (!newUser.value.name.first || !newUser.value.email) {
    alert('Nom et email obligatoires')
    return
  }

  const userToAdd: UserDoc = {
    _id: 'user:' + newUser.value.name.first.toLowerCase() + '.' + newUser.value.name.last.toLowerCase(),
    type: 'user',
    name: { ...newUser.value.name },
    email: newUser.value.email,
    tags: newUser.value.tags.length ? newUser.value.tags : ['new'],
    created_at: new Date().toISOString()
  }

  try {
    const response = await db.value.put(userToAdd)
    console.log('Utilisateur ajouté :', response)
    await fetchUsers()
    newUser.value = {
      type: 'user',
      name: { first: '', last: '' },
      email: '',
      tags: [],
      created_at: ''
    }
  } catch (error) {
    console.error('Erreur lors de l’ajout :', error)
  }
}

onMounted(() => {
  connectToCouchDB()
  setTimeout(fetchUsers, 500)
})
</script>

<template>
  <section style="padding:2rem;background:#f9f9f9;">
    <h1>Connexion à CouchDB</h1>

    <form @submit.prevent="addUser" style="margin-bottom:2rem;padding:1rem;border:1px solid #ccc;border-radius:8px;">
      <h3>Ajouter un utilisateur</h3>
      <div style="margin-bottom:0.5rem;">
        <label>Prénom :</label>
        <input v-model="newUser.name.first" required />
      </div>
      <div style="margin-bottom:0.5rem;">
        <label>Nom :</label>
        <input v-model="newUser.name.last" />
      </div>
      <div style="margin-bottom:0.5rem;">
        <label>Email :</label>
        <input v-model="newUser.email" required />
      </div>
      <button type="submit">Ajouter</button>
    </form>

    <div v-if="users.length === 0">
      <p>Aucune donnée trouvée dans la base.</p>
    </div>

    <article
      v-for="user in users"
      :key="user._id"
      style="margin-bottom:1rem;padding:1rem;border:1px solid #ccc;border-radius:8px;"
    >
      <h2>{{ user.name?.first }} {{ user.name?.last }}</h2>
      <p><strong>Email :</strong> {{ user.email }}</p>
      <p><strong>Tags :</strong> {{ user.tags?.join(', ') }}</p>
      <small><strong>Créé le :</strong> {{ user.created_at }}</small>
    </article>
  </section>
</template>

<style scoped>
h1 {
  color: #333;
  margin-bottom: 1rem;
}
input {
  margin-left: 0.5rem;
  padding: 0.25rem;
}
button {
  padding: 0.5rem 1rem;
  cursor: pointer;
}
</style>