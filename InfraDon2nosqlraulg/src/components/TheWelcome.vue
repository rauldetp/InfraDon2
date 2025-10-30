<script setup lang="ts">
import { ref, onMounted } from 'vue'
import PouchDB from 'pouchdb'

// Définition stricte du type d'utilisateur
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
  editing?: boolean
}

// Références réactives
const db = ref<PouchDB.Database<UserDoc> | null>(null)
const users = ref<Array<UserDoc>>([])

// Modèle de création
const newUser = ref<UserDoc>({
  type: 'user',
  name: { first: '', last: '' },
  email: '',
  tags: [],
  created_at: ''
})

// Connexion à CouchDB
function connectToCouchDB(): void {
  try {
    const database = new PouchDB<UserDoc>('http://admin:MbappeVini135@127.0.0.1:5984/vini')
    db.value = database
    console.log('Connecté à CouchDB')
  } catch (error) {
    console.error('Erreur de connexion :', error)
  }
}

// Lecture (FETCH)
async function fetchUsers(): Promise<void> {
  if (!db.value) return
  try {
    const result = await db.value.allDocs({ include_docs: true })
    users.value = result.rows
      .filter(row => !!row.doc)
      .map(row => ({ ...(row.doc as UserDoc), editing: false }))
  } catch (error) {
    console.error('Erreur de lecture CouchDB :', error)
  }
}

// Création (CREATE)
async function addUser(): Promise<void> {
  if (!db.value) return
  if (!newUser.value.name.first || !newUser.value.email) {
    alert('Nom et email obligatoires')
    return
  }

  const userToAdd: UserDoc = {
    _id: `user:${newUser.value.name.first.toLowerCase()}.${newUser.value.name.last.toLowerCase()}`,
    type: 'user',
    name: { ...newUser.value.name },
    email: newUser.value.email,
    tags: [...newUser.value.tags],
    created_at: new Date().toISOString(),
    editing: false
  }

  try {
    await db.value.put(userToAdd)
    console.log('Utilisateur ajouté :', userToAdd)
    await fetchUsers()
    newUser.value = { type: 'user', name: { first: '', last: '' }, email: '', tags: [], created_at: '' }
  } catch (error) {
    console.error('Erreur lors de l’ajout :', error)
  }
}

// Mise à jour (UPDATE)
async function updateUser(user: UserDoc): Promise<void> {
  if (!db.value || !user._id) return
  try {
    await db.value.put(user)
    console.log('Utilisateur mis à jour :', user)
    user.editing = false
    await fetchUsers()
  } catch (error) {
    console.error('Erreur lors de la mise à jour :', error)
  }
}

// Suppression (DELETE)
async function deleteUser(user: UserDoc): Promise<void> {
  if (!db.value || !user._id || !user._rev) return
  try {
    await db.value.remove(user._id, user._rev)
    console.log('Utilisateur supprimé :', user._id)
    await fetchUsers()
  } catch (error) {
    console.error('Erreur lors de la suppression :', error)
  }
}

// Gestion des tags
function addTag(user: UserDoc, tag: string): void {
  const value = tag.trim()
  if (value && !user.tags.includes(value)) {
    user.tags.push(value)
  }
}

function removeTag(user: UserDoc, tag: string): void {
  user.tags = user.tags.filter(t => t !== tag)
}

function onTagEnter(user: UserDoc, event: KeyboardEvent): void {
  const input = event.target as HTMLInputElement
  const value = input.value.trim()
  if (value) {
    addTag(user, value)
    input.value = ''
  }
}

// Montage initial
onMounted(() => {
  connectToCouchDB()
  setTimeout(fetchUsers, 500)
})
</script>

<template>
  <section style="padding:2rem;background:#f9f9f9;">
    <h1>Gestion de la base CouchDB</h1>

    <!-- Formulaire d'ajout -->
    <form @submit.prevent="addUser" style="margin-bottom:2rem;padding:1rem;border:1px solid #ccc;border-radius:8px;">
      <h3>Ajouter un utilisateur</h3>
      <div><label>Prénom :</label><input v-model="newUser.name.first" required /></div>
      <div><label>Nom :</label><input v-model="newUser.name.last" /></div>
      <div><label>Email :</label><input v-model="newUser.email" required /></div>
      <div>
        <label>Tags :</label>
        <input
          @keyup.enter.prevent="onTagEnter(newUser, $event)"
          placeholder="Appuyer sur Entrée pour ajouter un tag"
        />
        <span
          v-for="tag in newUser.tags"
          :key="tag"
          style="margin:0 0.3rem; background:#e0e0e0; padding:0.2rem 0.5rem; border-radius:5px;"
        >
          {{ tag }}
          <button @click.prevent="removeTag(newUser, tag)">x</button>
        </span>
      </div>
      <button type="submit">Ajouter</button>
    </form>

    <!-- Liste -->
    <div v-if="users.length === 0"><p>Aucune donnée trouvée dans la base.</p></div>

    <article
      v-for="user in users"
      :key="user._id"
      style="margin-bottom:1rem;padding:1rem;border:1px solid #ccc;border-radius:8px;"
    >
      <!-- Lecture -->
      <div v-if="!user.editing">
        <h2>{{ user.name.first }} {{ user.name.last }}</h2>
        <p><strong>Email :</strong> {{ user.email }}</p>
        <p><strong>Tags :</strong> {{ user.tags.join(', ') }}</p>
        <small><strong>Créé le :</strong> {{ new Date(user.created_at).toLocaleString() }}</small>
        <div style="margin-top:0.5rem;">
          <button @click="user.editing = true">Modifier</button>
          <button @click="deleteUser(user)">Supprimer</button>
        </div>
      </div>

      <!-- Édition -->
      <div v-else>
        <h3>Éditer utilisateur</h3>
        <label>Prénom :</label><input v-model="user.name.first" />
        <label>Nom :</label><input v-model="user.name.last" />
        <label>Email :</label><input v-model="user.email" />
        <div>
          <label>Tags :</label>
          <input @keyup.enter.prevent="onTagEnter(user, $event)" placeholder="Entrer un tag et appuyer sur Entrée" />
          <span
            v-for="tag in user.tags"
            :key="tag"
            style="margin:0 0.3rem; background:#e0e0e0; padding:0.2rem 0.5rem; border-radius:5px;"
          >
            {{ tag }}
            <button @click.prevent="removeTag(user, tag)">x</button>
          </span>
        </div>
        <div style="margin-top:0.5rem;">
          <button @click="updateUser(user)">Sauvegarder</button>
          <button @click="user.editing = false">Annuler</button>
        </div>
      </div>
    </article>
  </section>
</template>

<style scoped>
h1 {
  color: #333;
  margin-bottom: 1rem;
}
input {
  margin: 0.3rem;
  padding: 0.25rem;
}
button {
  margin-left: 0.3rem;
  padding: 0.3rem 0.6rem;
  cursor: pointer;
}
</style>