<script setup lang="ts">
import { ref, onMounted } from 'vue'
import PouchDB from 'pouchdb'
import PouchDBFind from 'pouchdb-find'

/**
 * Typage de PouchDB avec la méthode plugin (évite le "as any")
 */
type PouchWithPlugin = typeof PouchDB & {
  plugin: (plugin: unknown) => void
}

// Activation du plugin pouchdb-find (createIndex / find)
;(PouchDB as PouchWithPlugin).plugin(PouchDBFind)

/** Modèle strict d'un utilisateur dans nos bases (locale + distante) */
interface UserDoc {
  _id?: string          // identifiant du document
  _rev?: string         // révision (gérée par CouchDB / PouchDB)
  type: 'user'
  name: {
    first: string
    last: string
  }
  email: string
  tags: string[]
  created_at: string    // date de création
  updated_at?: string   // date de dernière modification (facultative)
  editing?: boolean     // utilisé uniquement dans l'UI (mode édition)
}

/**
 * Typage de la base locale avec les méthodes ajoutées par pouchdb-find
 * (createIndex, find)
 */
type LocalDB = PouchDB.Database<UserDoc> & {
  createIndex: (def: { index: { fields: string[] } }) => Promise<unknown>
  find: (query: {
    selector: Record<string, unknown>
    sort?: string[]
  }) => Promise<{ docs: UserDoc[] }>
}

/* =========================
   Bases locales / distantes
   ========================= */

// Base PouchDB locale (dans le navigateur)
const localDb = ref<LocalDB | null>(null)

// Base CouchDB distante (serveur)
const remoteDb = ref<PouchDB.Database<UserDoc> | null>(null)

// Liste des utilisateurs affichés dans l'UI
const users = ref<UserDoc[]>([])

// États pour l'interface
const offline = ref(false)   // true = mode offline (pas de sync)
const searchTerm = ref('')   // texte de recherche (sur email)
const isSyncing = ref(false) // indique si une synchronisation est active

// Handler retourné par .sync() pour pouvoir annuler la réplication
let syncHandler: PouchDB.Replication.Sync<UserDoc> | null = null

/* =========================
   Formulaire d'ajout
   ========================= */

// Modèle local pour le formulaire "Ajouter un utilisateur"
const newUser = ref<UserDoc>({
  type: 'user',
  name: { first: '', last: '' },
  email: '',
  tags: [],
  created_at: ''
})

/* =========================
   INIT : bases + index + sync
   ========================= */
async function initDatabases(): Promise<void> {
  try {
    // 1) Connexion à la base distante CouchDB (serveur)
    remoteDb.value = new PouchDB<UserDoc>('http://admin:MbappeVini135@127.0.0.1:5984/vini')

    // 2) Création/connexion de la base PouchDB locale (dans le navigateur)
    localDb.value = new PouchDB<UserDoc>('vini_local') as LocalDB

    // 3) Création d'un index sur le champ "email" dans la base locale
    await localDb.value.createIndex({
      index: { fields: ['email'] }
    })
    console.log('Index sur "email" créé dans la base locale')

    // 4) Lancer la synchro locale ⇆ distante si on n'est pas offline
    if (!offline.value) {
      startSync()
    }

    // 5) Charger la liste des utilisateurs depuis la base LOCALE
    await fetchUsers()
  } catch (err) {
    console.error('Erreur lors de l’initialisation des bases :', err)
  }
}

/* =========================
   RÉPLICATION (sync locale ⇆ distante)
   ========================= */

// Démarrage de la synchronisation bidirectionnelle (live + retry)
function startSync(): void {
  if (!localDb.value || !remoteDb.value || syncHandler) return

  console.log('Démarrage de la synchronisation locale ⇆ distante')
  isSyncing.value = true

  syncHandler = localDb.value
    .sync(remoteDb.value, {
      live: true,   // réplication continue
      retry: true   // réessaie en cas d’erreur réseau
    })
    .on('change', () => {
      console.log('Changement détecté via sync → rafraîchissement')
      // Si une recherche est active, on relance la recherche
      // sinon, on recharge tous les docs
      if (searchTerm.value.trim()) {
        runSearch()
      } else {
        fetchUsers()
      }
    })
    .on('paused', info => {
      console.log('Sync pausée :', info)
    })
    .on('active', () => {
      console.log('Sync active')
    })
    .on('error', err => {
      console.error('Erreur de synchronisation :', err)
      isSyncing.value = false
    })
}

// Arrêt de la synchronisation (utilisé pour le mode offline)
function stopSync(): void {
  if (syncHandler) {
    console.log('Arrêt de la synchronisation')
    syncHandler.cancel()
    syncHandler = null
  }
  isSyncing.value = false
}

// Toggle du mode offline/online
function toggleOfflineMode(): void {
  offline.value = !offline.value
  if (offline.value) {
    // En mode offline, on coupe la synchronisation
    stopSync()
  } else {
    // Lorsque l'on repasse en online, on relance la sync
    startSync()
  }
}

/* =========================
   READ — récupérer depuis la base locale
   ========================= */
async function fetchUsers(): Promise<void> {
  if (!localDb.value) return
  try {
    const result = await localDb.value.allDocs({ include_docs: true })
    users.value = result.rows
      .filter(r => r.doc)
      .map(r => {
        const d = r.doc as UserDoc
        // on force editing = false pour ne pas rester en mode édition
        return { ...d, editing: false }
      })
      // tri par date de création décroissante (les plus récents en premier)
      .sort((a, b) => new Date(b.created_at).getTime() - new Date(a.created_at).getTime())
  } catch (error) {
    console.error('Erreur de lecture (allDocs local) :', error)
  }
}

/* =========================
   CREATE — ajout dans la base locale
   (sera répliqué vers CouchDB quand online)
   ========================= */
async function addUser(): Promise<void> {
  if (!localDb.value) return

  // Validation minimale
  if (!newUser.value.name.first.trim() || !newUser.value.email.trim()) {
    alert('Prénom et email sont obligatoires.')
    return
  }

  // Construction d’un id unique : user:prenom.nom:timestamp
  const baseId = `user:${newUser.value.name.first.toLowerCase()}.${(newUser.value.name.last || '').toLowerCase()}`
  const _id = `${baseId}:${Date.now()}`

  const userToAdd: UserDoc = {
    _id,
    type: 'user',
    name: {
      first: newUser.value.name.first.trim(),
      last: (newUser.value.name.last || '').trim()
    },
    email: newUser.value.email.trim(),
    tags: [...newUser.value.tags],
    created_at: new Date().toISOString()
  }

  try {
    // On ajoute dans la base LOCALE uniquement
    await localDb.value.put(userToAdd)
    console.log('Utilisateur ajouté en local :', userToAdd)

    // Si pas de recherche, on recharge tout, sinon on met juste à jour la recherche
    if (!searchTerm.value.trim()) {
      await fetchUsers()
    } else {
      await runSearch()
    }

    // Reset du formulaire
    newUser.value = {
      type: 'user',
      name: { first: '', last: '' },
      email: '',
      tags: [],
      created_at: ''
    }
  } catch (error) {
    console.error('Erreur lors de l’ajout en local :', error)
  }
}

/* =========================
   UPDATE — modifier un document (local)
   ========================= */
async function updateUser(user: UserDoc): Promise<void> {
  if (!localDb.value || !user._id) return
  try {
    // On récupère la dernière version du doc pour avoir la bonne _rev
    const latest = await localDb.value.get(user._id)

    const toSave: UserDoc = {
      ...latest, // contient _id, _rev, created_at, etc.
      type: 'user',
      name: {
        first: user.name.first.trim(),
        last: (user.name.last || '').trim()
      },
      email: user.email.trim(),
      tags: Array.isArray(user.tags) ? [...user.tags] : [],
      created_at: latest.created_at,
      updated_at: new Date().toISOString()
    }

    // Mise à jour dans la base locale
    await localDb.value.put(toSave)
    console.log('Utilisateur mis à jour en local :', toSave)
    user.editing = false

    // Mise à jour de la liste ou des résultats de recherche
    if (!searchTerm.value.trim()) {
      await fetchUsers()
    } else {
      await runSearch()
    }
  } catch (error) {
    console.error('Erreur lors de la mise à jour :', error)
    alert('Échec de la mise à jour. Recharge les données et réessaie.')
  }
}

/* =========================
   DELETE — supprimer un document (local)
   ========================= */
async function deleteUser(user: UserDoc): Promise<void> {
  if (!localDb.value || !user._id) return
  try {
    // On récupère la dernière version pour avoir la _rev
    const latest = await localDb.value.get(user._id)
    await localDb.value.remove(latest._id!, latest._rev!)
    console.log('Utilisateur supprimé (local) :', latest._id)

    if (!searchTerm.value.trim()) {
      await fetchUsers()
    } else {
      await runSearch()
    }
  } catch (error) {
    console.error('Erreur lors de la suppression :', error)
  }
}

/* =========================
   TAGS helpers (ajout / suppression de tags)
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

function onTagEnter(user: UserDoc, e: KeyboardEvent): void {
  const target = e.target as HTMLInputElement | null
  const value = target?.value?.trim() ?? ''
  if (!value) return
  addTag(user, value)
  if (target) {
    target.value = ''
  }
}

/* =========================
   INDEXATION / RECHERCHE (pouchdb-find sur email)
   ========================= */
async function runSearch(): Promise<void> {
  if (!localDb.value) return

  const term = searchTerm.value.trim()
  if (!term) {
    // Si le champ de recherche est vide, on recharge tous les docs
    await fetchUsers()
    return
  }

  try {
    // On cherche les emails qui commencent par "term" (préfixe)
    const lower = term.toLowerCase()
    const res = await localDb.value.find({
      selector: {
        email: { $gte: lower, $lte: lower + '\uffff' }
      },
      sort: ['email']
    })

    users.value = res.docs
      .map(d => ({ ...d, editing: false }))
      .sort((a, b) => a.email.localeCompare(b.email))
  } catch (err) {
    console.error('Erreur lors de la recherche :', err)
  }
}

/* =========================
   FACTORY — génération de faux utilisateurs
   ========================= */
   function buildFakeUser(index: number): UserDoc {
  const firstNames: string[] = ['Kylian', 'Bukayo', 'Vinicius', 'Erling', 'Jude', 'Lamine']
  const lastNames: string[] = ['Mbappe', 'Saka', 'Junior', 'Haaland', 'Bellingham', 'Yamal']

  const first: string =
    firstNames[Math.floor(Math.random() * firstNames.length)] ?? ''
  const last: string =
    lastNames[Math.floor(Math.random() * lastNames.length)] ?? ''

  const email: string = `${first.toLowerCase()}.${last.toLowerCase()}.${index}@heig-vd.ch`

  return {
    type: 'user',
    name: { first, last },
    email,
    tags: ['factory'],
    created_at: new Date().toISOString()
  }
}

// Ajoute "count" faux utilisateurs dans la base locale (pour tester indexation + sync)
async function populateFactory(count = 20): Promise<void> {
  if (!localDb.value) return
  const docs: UserDoc[] = []

  for (let i = 0; i < count; i++) {
    const fake = buildFakeUser(i)
    // _id unique par doc factory
    fake._id = `factory:${fake.email}:${Date.now()}:${i}`
    docs.push(fake)
  }

  try {
    await localDb.value.bulkDocs(docs)
    console.log(`${count} faux utilisateurs ajoutés en local`)
    if (!searchTerm.value.trim()) {
      await fetchUsers()
    } else {
      await runSearch()
    }
  } catch (err) {
    console.error('Erreur factory bulkDocs :', err)
  }
}

/* =========================
   CYCLE DE VIE DU COMPOSANT
   ========================= */
onMounted(async () => {
  await initDatabases()
})

/* =========================
   Hack pour TS/ESLint :
   on indique explicitement que ces fonctions
   sont utilisées (dans le template)
   ========================= */
void removeTag
void onTagEnter
</script>

<template>
  <section style="padding:2rem;background:#f9f9f9;max-width:1000px;margin:0 auto;">
    <!-- Header avec titre + toggle offline + état de sync -->
    <header style="display:flex;justify-content:space-between;align-items:center;gap:1rem;flex-wrap:wrap;">
      <h1 style="margin:0;">Réplication &amp; Indexation – base CouchDB "vini"</h1>

      <div style="display:flex;align-items:center;gap:1rem;">
        <!-- Toggle offline -->
        <label style="display:flex;align-items:center;gap:0.4rem;cursor:pointer;">
          <input type="checkbox" :checked="offline" @change="toggleOfflineMode" />
          <span>Mode offline</span>
        </label>
        <!-- Badge d'état de la synchronisation -->
        <span
          :style="{
            padding: '0.2rem 0.6rem',
            borderRadius: '999px',
            fontSize: '0.8rem',
            background: isSyncing && !offline ? '#d4fdd4' : '#f5d7d7',
            border: '1px solid #ccc'
          }"
        >
          Sync : {{ !offline && isSyncing ? 'active' : 'inactive' }}
        </span>
      </div>
    </header>

    <!-- Zone de recherche + factory pour générer des données -->
    <section
      style="margin-top:1rem;margin-bottom:1rem;padding:0.8rem 1rem;border-radius:8px;border:1px solid #ddd;background:#fff;display:flex;flex-wrap:wrap;gap:0.8rem;align-items:center;"
    >
      <div style="flex:2 1 260px;">
        <label style="display:block;font-size:0.9rem;margin-bottom:0.2rem;">Recherche par email (indexée)</label>
        <input
          v-model="searchTerm"
          @input="runSearch"
          placeholder="Commence à taper un email…"
          style="width:100%;padding:0.4rem;border-radius:6px;border:1px solid #ccc;"
        />
      </div>

      <div style="flex:1 1 200px;display:flex;gap:0.4rem;flex-wrap:wrap;">
        <button type="button" @click="populateFactory(5)">+5 faux users</button>
        <button type="button" @click="populateFactory(20)">+20 faux users</button>
      </div>
    </section>

    <!-- Formulaire d'ajout d'utilisateur -->
    <form
      @submit.prevent="addUser"
      style="margin-bottom:2rem;padding:1rem;border:1px solid #ddd;border-radius:8px;background:#fff;"
    >
      <h3 style="margin:0 0 1rem;">Ajouter un utilisateur (local → répliqué)</h3>

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

    <!-- Liste des utilisateurs -->
    <div v-if="users.length === 0">
      <p>Aucune donnée trouvée dans la base locale.</p>
    </div>

    <article
      v-for="user in users"
      :key="user._id"
      style="margin-bottom:1rem;padding:1rem;border:1px solid #ddd;border-radius:8px;background:#fff;"
    >
      <!-- Mode lecture (affichage normal) -->
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
            <small style="display:block;">
              <strong>Créé le :</strong> {{ new Date(user.created_at).toLocaleString() }}
            </small>
            <small v-if="user.updated_at" style="display:block;">
              <strong>Modifié le :</strong> {{ new Date(user.updated_at).toLocaleString() }}
            </small>
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
  cursor: pointer;
}
button:hover {
  background: #eee;
}
input {
  border: 1px solid #ccc;
  border-radius: 6px;
}
</style>