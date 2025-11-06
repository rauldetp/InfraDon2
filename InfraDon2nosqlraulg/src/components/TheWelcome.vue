<script setup lang="ts">
import { ref, onMounted } from 'vue'
import PouchDB from 'pouchdb'

/** Modèle strict d'un utilisateur stocké dans CouchDB */
interface UserDoc {
  _id?: string
  _rev?: string
  type: 'user'
  name: { first: string; last: string }
  email: string
  tags: string[]
  created_at: string
  updated_at?: string
  /** champ UI local, non persisté si on veut l’ignorer */
  editing?: boolean
}

/* =========================
   Références réactives
   ========================= */
const db = ref<PouchDB.Database<UserDoc> | null>(null)
const users = ref<UserDoc[]>([])

/* Modèle pour le formulaire d'ajout */
const newUser = ref<UserDoc>({
  type: 'user',
  name: { first: '', last: '' },
  email: '',
  tags: [],
  created_at: ''
})

/* =========================
   Connexion CouchDB distante
   ========================= */
function connectToCouchDB(): void {
  try {
    // ⚠️ Assure-toi que CORS est activé dans CouchDB (voir note en bas)
    const remote = new PouchDB<UserDoc>('http://admin:MbappeVini135@127.0.0.1:5984/vini')
    db.value = remote
    console.log('Connecté à CouchDB (base: vini)')
  } catch (error) {
    console.error('Erreur de connexion CouchDB :', error)
  }
}

/* =========================
   READ — récupérer tous les docs
   ========================= */
async function fetchUsers(): Promise<void> {
  if (!db.value) return
  try {
    const result = await db.value.allDocs({ include_docs: true })
    users.value = result.rows
      .filter(r => r.doc)
      .map(r => {
        const d = r.doc as UserDoc
        return { ...d, editing: false }
      })
      // tri simple par date de création descendante
      .sort((a, b) => new Date(b.created_at).getTime() - new Date(a.created_at).getTime())
  } catch (error) {
    console.error('Erreur de lecture (allDocs) :', error)
  }
}

/* =========================
   CREATE — ajouter un document
   ========================= */
async function addUser(): Promise<void> {
  if (!db.value) return

  if (!newUser.value.name.first.trim() || !newUser.value.email.trim()) {
    alert('Prénom et email sont obligatoires.')
    return
  }

  // ID déterministe basé sur prénom.nom + timestamp pour éviter les conflits
  const baseId = `user:${newUser.value.name.first.toLowerCase()}.${(newUser.value.name.last || '').toLowerCase()}`
  const _id = `${baseId}:${Date.now()}`

  const userToAdd: UserDoc = {
    _id,
    type: 'user',
    name: { first: newUser.value.name.first.trim(), last: (newUser.value.name.last || '').trim() },
    email: newUser.value.email.trim(),
    tags: [...newUser.value.tags],
    created_at: new Date().toISOString()
  }

  try {
    await db.value.put(userToAdd)
    console.log('Utilisateur ajouté :', userToAdd)
    await fetchUsers()

    // reset formulaire
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

/* =========================
   UPDATE — modifier un document
   - On relit la dernière révision pour éviter un conflit basique.
   - On met à jour updated_at.
   ========================= */
async function updateUser(user: UserDoc): Promise<void> {
  if (!db.value || !user._id) return
  try {
    // Relire la dernière version (rev la plus récente)
    const latest = await db.value.get(user._id)
    const toSave: UserDoc = {
      ...latest,
      type: 'user',
      name: { first: user.name.first.trim(), last: (user.name.last || '').trim() },
      email: user.email.trim(),
      tags: Array.isArray(user.tags) ? [...user.tags] : [],
      created_at: latest.created_at, // on conserve la date de création
      updated_at: new Date().toISOString()
    }

    await db.value.put(toSave)
    console.log('Utilisateur mis à jour :', toSave)
    user.editing = false
    await fetchUsers()
  } catch (error) {
    // Conflit 409 ? On peut conseiller de recharger/fusionner si besoin
    console.error('Erreur lors de la mise à jour :', error)
    alert('Échec de la mise à jour. Recharge les données et réessaie.')
  }
}

/* =========================
   DELETE — supprimer un document
   ========================= */
async function deleteUser(user: UserDoc): Promise<void> {
  if (!db.value || !user._id) return
  try {
    const latest = await db.value.get(user._id)
    await db.value.remove(latest._id!, latest._rev!)
    console.log('Utilisateur supprimé :', latest._id)
    await fetchUsers()
  } catch (error) {
    console.error('Erreur lors de la suppression :', error)
  }
}

/* =========================
   Tags helpers
   ========================= */
function addTag(user: UserDoc, tag: string): void {
  const value = tag.trim()
  if (!value) return
  if (!Array.isArray(user.tags)) user.tags = []
  if (!user.tags.includes(value)) user.tags.push(value)
}

function removeTag(user: UserDoc, tag: string): void {
  if (!Array.isArray(user.tags)) return
  user.tags = user.tags.filter(t => t !== tag)
}

function onTagEnter(user: UserDoc, event: KeyboardEvent): void {
  const input = event.target as HTMLInputElement
  const value = (input?.value || '').trim()
  if (value) {
    addTag(user, value)
    input.value = ''
  }
}

/* =========================
   Cycle de vie
   ========================= */
onMounted(async () => {
  connectToCouchDB()
  await fetchUsers()
})
</script>

<template>
  <section style="padding:2rem;background:#f9f9f9;max-width:900px;margin:0 auto;">
    <h1 style="margin-bottom:1rem;">Gestion de la base CouchDB (vini)</h1>

    <!-- Formulaire d'ajout -->
    <form
      @submit.prevent="addUser"
      style="margin-bottom:2rem;padding:1rem;border:1px solid #ddd;border-radius:8px;background:#fff;"
    >
      <h3 style="margin:0 0 1rem;">Ajouter un utilisateur</h3>

      <div style="display:flex;gap:0.5rem;flex-wrap:wrap;">
        <label style="flex:1 1 180px;">Prénom
          <input v-model="newUser.name.first" required style="width:100%;padding:0.4rem;margin-top:0.2rem;" />
        </label>
        <label style="flex:1 1 180px;">Nom
          <input v-model="newUser.name.last" style="width:100%;padding:0.4rem;margin-top:0.2rem;" />
        </label>
        <label style="flex:2 1 240px;">Email
          <input v-model="newUser.email" required style="width:100%;padding:0.4rem;margin-top:0.2rem;" />
        </label>
      </div>

      <div style="margin-top:0.6rem;">
        <label>Tags</label>
        <input
          @keyup.enter.prevent="onTagEnter(newUser, $event)"
          placeholder="Entrer un tag puis Entrée"
          style="padding:0.4rem;margin-left:0.6rem;min-width:280px;"
        />
        <span
          v-for="tag in newUser.tags"
          :key="`new-${tag}`"
          style="display:inline-flex;align-items:center;margin:0.3rem; background:#eef; padding:0.2rem 0.5rem; border-radius:12px;"
        >
          {{ tag }}
          <button @click.prevent="removeTag(newUser, tag)" style="margin-left:0.4rem;">x</button>
        </span>
      </div>

      <div style="margin-top:0.8rem;">
        <button type="submit">Ajouter</button>
      </div>
    </form>

    <!-- Liste -->
    <div v-if="users.length === 0">
      <p>Aucune donnée trouvée dans la base.</p>
    </div>

    <article
      v-for="user in users"
      :key="user._id"
      style="margin-bottom:1rem;padding:1rem;border:1px solid #ddd;border-radius:8px;background:#fff;"
    >
      <!-- Mode lecture -->
      <div v-if="!user.editing">
        <div style="display:flex;justify-content:space-between;align-items:start;gap:1rem;flex-wrap:wrap;">
          <div>
            <h2 style="margin:0 0 0.3rem;">
              {{ user.name.first || '—' }} {{ user.name.last || '' }}
            </h2>
            <p style="margin:0.2rem 0;"><strong>Email :</strong> {{ user.email }}</p>
            <p style="margin:0.2rem 0;">
              <strong>Tags :</strong>
              <template v-if="user.tags && user.tags.length">
                {{ user.tags.join(', ') }}
              </template>
              <template v-else>—</template>
            </p>
          </div>
          <div style="text-align:right;min-width:240px;">
            <small style="display:block;"><strong>Créé le :</strong> {{ new Date(user.created_at).toLocaleString() }}</small>
            <small v-if="user.updated_at" style="display:block;"><strong>Modifié le :</strong> {{ new Date(user.updated_at).toLocaleString() }}</small>
          </div>
        </div>

        <div style="margin-top:0.6rem;">
          <button @click="user.editing = true">Modifier</button>
          <button @click="deleteUser(user)" style="margin-left:0.4rem;">Supprimer</button>
        </div>
      </div>

      <!-- Mode édition -->
      <div v-else>
        <h3 style="margin:0 0 0.6rem;">Éditer l’utilisateur</h3>

        <div style="display:flex;gap:0.5rem;flex-wrap:wrap;">
          <label style="flex:1 1 180px;">Prénom
            <input v-model="user.name.first" style="width:100%;padding:0.4rem;margin-top:0.2rem;" />
          </label>
          <label style="flex:1 1 180px;">Nom
            <input v-model="user.name.last" style="width:100%;padding:0.4rem;margin-top:0.2rem;" />
          </label>
          <label style="flex:2 1 240px;">Email
            <input v-model="user.email" style="width:100%;padding:0.4rem;margin-top:0.2rem;" />
          </label>
        </div>

        <div style="margin-top:0.6rem;">
          <label>Tags</label>
          <input
            @keyup.enter.prevent="onTagEnter(user, $event)"
            placeholder="Entrer un tag puis Entrée"
            style="padding:0.4rem;margin-left:0.6rem;min-width:280px;"
          />
          <span
            v-for="tag in user.tags"
            :key="`${user._id}-${tag}`"
            style="display:inline-flex;align-items:center;margin:0.3rem; background:#eef; padding:0.2rem 0.5rem; border-radius:12px;"
          >
            {{ tag }}
            <button @click.prevent="removeTag(user, tag)" style="margin-left:0.4rem;">x</button>
          </span>
        </div>

        <div style="margin-top:0.8rem;">
          <button @click="updateUser(user)">Sauvegarder</button>
          <button @click="user.editing = false" style="margin-left:0.4rem;">Annuler</button>
        </div>
      </div>
    </article>
  </section>
</template>

<style scoped>
button {
  padding: 0.45rem 0.9rem;
  border: 1px solid #ccc;
  background: #f6f6f6;
  border-radius: 6px;
}
button:hover { background: #eee; }
input {
  border: 1px solid #ccc;
  border-radius: 6px;
}
</style>