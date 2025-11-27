<script setup lang="ts">
import { ref, onMounted } from 'vue'
import PouchDB from 'pouchdb'
import PouchDBFind from 'pouchdb-find'

  // Activation du plugin find
  ; (PouchDB as unknown as { plugin: (p: unknown) => void }).plugin(PouchDBFind)

/* -------------------------------------------------------
 * TYPES
 * ----------------------------------------------------- */

interface InfraComment {
  text: string
  created_at: string
}

interface InfraDoc {
  _id?: string
  _rev?: string
  name: string
  content: string
  created_at: string
  updated_at?: string
  likes: number
  comments: InfraComment[]
}

/* PouchDB types */
type LocalDB = PouchDB.Database<InfraDoc> & {
  createIndex: (opt: { index: { fields: string[] } }) => Promise<unknown>
  find: (query: {
    selector: Record<string, unknown>
    sort?: Array<Record<string, 'asc' | 'desc'>>
  }) => Promise<{ docs: InfraDoc[] }>
}

/* -------------------------------------------------------
 * STATE
 * ----------------------------------------------------- */

const LOCAL_DB = 'vini_local'
const REMOTE_DB = 'http://admin:MbappeVini135@127.0.0.1:5984/vini'

const localDb = ref<LocalDB | null>(null)
const remoteDb = ref<PouchDB.Database<InfraDoc> | null>(null)

const docs = ref<InfraDoc[]>([])
const loading = ref(false)
const error = ref('')

const online = ref(true)
const isSyncing = ref(false)

const form = ref({
  _id: '',
  _rev: '',
  name: '',
  content: ''
})
const isEdit = ref(false)

const searchTerm = ref('')
const sortByLikes = ref(false)

const commentDrafts = ref<Record<string, string>>({})

let syncHandler: PouchDB.Replication.Sync<InfraDoc> | null = null

/* -------------------------------------------------------
 * INIT
 * ----------------------------------------------------- */

function initLocalDb(): LocalDB {
  if (!localDb.value) {
    localDb.value = new PouchDB(LOCAL_DB) as LocalDB
  }
  return localDb.value
}

function initRemoteDb(): PouchDB.Database<InfraDoc> {
  if (!remoteDb.value) {
    remoteDb.value = new PouchDB(REMOTE_DB)
  }
  return remoteDb.value
}

/* -------------------------------------------------------
 * NORMALISATION
 * ----------------------------------------------------- */

function normalizeDoc(raw: unknown): InfraDoc {
  const doc = raw as Partial<InfraDoc>

  return {
    _id: doc._id,
    _rev: doc._rev,
    name: doc.name ?? '',
    content: doc.content ?? '',
    created_at: doc.created_at ?? new Date().toISOString(),
    updated_at: doc.updated_at,
    likes: typeof doc.likes === 'number' ? doc.likes : 0,
    comments: Array.isArray(doc.comments) ? (doc.comments as InfraComment[]) : []
  }
}

/* -------------------------------------------------------
 * INDEXES
 * ----------------------------------------------------- */

async function ensureIndexes(): Promise<void> {
  const db = initLocalDb()
  await db.createIndex({ index: { fields: ['name'] } })
  await db.createIndex({ index: { fields: ['likes'] } })
  // index pour le tri par date (sans JS)
  await db.createIndex({ index: { fields: ['created_at'] } })
}

/* -------------------------------------------------------
 * FETCH DOCS
 * (tri FAIT PAR POUCHDB, pas par JS)
 * ----------------------------------------------------- */

async function fetchDocs(): Promise<void> {
  const db = initLocalDb()
  loading.value = true
  error.value = ''

  try {
    if (sortByLikes.value) {
      // Tri par likes (indexé)
      const res = await db.find({
        selector: { likes: { $gte: 0 } },
        sort: [{ likes: 'desc' }, { created_at: 'desc' }]
      })
      docs.value = res.docs.map(normalizeDoc)
    } else {
      // Tri par date (indexé, SANS .sort en JS)
      const res = await db.find({
        selector: { created_at: { $gte: '\u0000' } },
        sort: [{ created_at: 'desc' }]
      })
      docs.value = res.docs.map(normalizeDoc)
    }
  } catch (e) {
    const err = e as Error
    error.value = err.message
  } finally {
    loading.value = false
  }
}

/* -------------------------------------------------------
 * CRUD
 * ----------------------------------------------------- */

function resetForm(): void {
  isEdit.value = false
  form.value = {
    _id: '',
    _rev: '',
    name: '',
    content: ''
  }
}

async function createDoc(): Promise<void> {
  const db = initLocalDb()

  const newDoc: InfraDoc = {
    name: form.value.name.trim(),
    content: form.value.content.trim(),
    created_at: new Date().toISOString(),
    likes: 0,
    comments: []
  }

  await db.post(newDoc)
  await fetchDocs()
  resetForm()

  if (online.value) {
    await manualSync()
  }
}

function startEdit(doc: InfraDoc): void {
  isEdit.value = true
  form.value._id = doc._id ?? ''
  form.value._rev = doc._rev ?? ''
  form.value.name = doc.name
  form.value.content = doc.content
}

async function updateDoc(): Promise<void> {
  if (!form.value._id) return

  const db = initLocalDb()
  const latest = await db.get(form.value._id)
  const normalized = normalizeDoc(latest)

  const updated: InfraDoc = {
    ...normalized,
    name: form.value.name.trim(),
    content: form.value.content.trim(),
    updated_at: new Date().toISOString()
  }

  await db.put(updated)
  await fetchDocs()
  resetForm()

  if (online.value) {
    await manualSync()
  }
}

async function deleteDoc(doc: InfraDoc): Promise<void> {
  const db = initLocalDb()
  if (!doc._id) return

  const latest = await db.get(doc._id)
  await db.remove(latest._id as string, latest._rev as string)

  await fetchDocs()

  if (online.value) {
    await manualSync()
  }
}

async function submitForm(): Promise<void> {
  if (isEdit.value) {
    await updateDoc()
  } else {
    await createDoc()
  }
}

/* -------------------------------------------------------
 * LIKES
 * ----------------------------------------------------- */

async function likeDoc(doc: InfraDoc): Promise<void> {
  const db = initLocalDb()

  const latest = await db.get(doc._id!)
  const normalized = normalizeDoc(latest)

  const updated: InfraDoc = {
    ...normalized,
    likes: normalized.likes + 1
  }

  await db.put(updated)
  await fetchDocs()

  if (online.value) {
    await manualSync()
  }
}

function toggleSortLikes(): void {
  sortByLikes.value = !sortByLikes.value
  void fetchDocs()
}

/* -------------------------------------------------------
 * COMMENTAIRES
 * ----------------------------------------------------- */

function getDraft(id: string): string {
  return commentDrafts.value[id] ?? ''
}

function onCommentInput(id: string, e: Event): void {
  const target = e.target as HTMLInputElement
  commentDrafts.value[id] = target.value
}

async function addComment(doc: InfraDoc): Promise<void> {
  const db = initLocalDb()
  const draft = getDraft(doc._id!)

  if (!draft.trim()) return

  const latest = await db.get(doc._id!)
  const normalized = normalizeDoc(latest)

  const newComment: InfraComment = {
    text: draft.trim(),
    created_at: new Date().toISOString()
  }

  const updated: InfraDoc = {
    ...normalized,
    comments: [...normalized.comments, newComment]
  }

  await db.put(updated)
  commentDrafts.value[doc._id!] = ''
  await fetchDocs()

  if (online.value) {
    await manualSync()
  }
}

/* -------------------------------------------------------
 * SEARCH INDEXÉE (name)
 * ----------------------------------------------------- */

async function onSearch(): Promise<void> {
  const db = initLocalDb()
  const term = searchTerm.value.trim()

  if (!term) return fetchDocs()

  const res = await db.find({
    selector: { name: { $eq: term } }
  })

  docs.value = res.docs.map(normalizeDoc)
}

/* -------------------------------------------------------
 * FACTORY
 * ----------------------------------------------------- */

async function generateFake(n = 50): Promise<void> {
  const db = initLocalDb()
  const now = Date.now()
  const data: InfraDoc[] = []

  for (let i = 0; i < n; i++) {
    data.push({
      _id: `fake_${now}_${i}`,
      name: `Doc ${i}`,
      content: `Contenu numéro ${i}`,
      created_at: new Date().toISOString(),
      likes: Math.floor(Math.random() * 10),
      comments: []
    })
  }

  await db.bulkDocs(data)
  await fetchDocs()

  if (online.value) {
    await manualSync()
  }
}

/* -------------------------------------------------------
 * REPLICATION
 * ----------------------------------------------------- */

async function replicateFromRemote(): Promise<void> {
  const local = initLocalDb()
  const remote = initRemoteDb()
  await local.replicate.from(remote)
  await fetchDocs()
}

async function replicateToRemote(): Promise<void> {
  const local = initLocalDb()
  const remote = initRemoteDb()
  await local.replicate.to(remote)
}

async function manualSync(): Promise<void> {
  await replicateFromRemote()
  await replicateToRemote()
}

/* -------------------------------------------------------
 * LIVE SYNC
 * ----------------------------------------------------- */

function startLiveSync(): void {
  const local = initLocalDb()
  const remote = initRemoteDb()

  if (syncHandler) return

  isSyncing.value = true

  syncHandler = local
    .sync(remote, { live: true, retry: true })
    .on('change', () => {
      void fetchDocs()
    })
    .on('error', () => {
      isSyncing.value = false
    })
}

function stopLiveSync(): void {
  syncHandler?.cancel()
  syncHandler = null
  isSyncing.value = false
}

async function toggleOnline(): Promise<void> {
  online.value = !online.value

  if (online.value) {
    await manualSync()
    startLiveSync()
  } else {
    stopLiveSync()
  }
}

/* -------------------------------------------------------
 * MOUNT
 * ----------------------------------------------------- */

onMounted(async () => {
  initLocalDb()
  initRemoteDb()

  await ensureIndexes()
  await manualSync()
  await fetchDocs()

  startLiveSync()
})
</script>

<template>
  <section style="padding:2rem;background:#fafafa;max-width:1000px;margin:0 auto;">
    <header style="display:flex;justify-content:space-between;align-items:center;flex-wrap:wrap;gap:1rem;">
      <h1>InfraDon2 — CRUD + Réplication</h1>

      <div style="display:flex;gap:1rem;align-items:center;">
        <label style="display:flex;align-items:center;gap:0.4rem;">
          <input type="checkbox" :checked="online" @change="toggleOnline" />
          <span>{{ online ? 'ONLINE' : 'OFFLINE' }}</span>
        </label>

        <span :style="{
          padding: '0.3rem 0.6rem',
          borderRadius: '12px',
          fontSize: '0.8rem',
          background: online && isSyncing ? '#ccf3cc' : '#f8cccc',
          border: '1px solid #ccc'
        }">
          Sync : {{ online && isSyncing ? 'active' : 'inactive' }}
        </span>
      </div>
    </header>

    <div v-if="loading" style="margin-top:1rem;color:#666;">Chargement…</div>
    <div v-if="error" style="margin-top:1rem;color:#c00;">{{ error }}</div>

    <!-- BARRE DE RÉPLICATION -->
    <section style="margin-top:1.5rem;padding:1rem;background:#fff;border:1px solid #ccc;border-radius:8px;">
      <h2 style="margin-top:0;">Réplication manuelle</h2>
      <button @click="replicateFromRemote">distant → local</button>
      <button @click="replicateToRemote" style="margin-left:0.5rem;">local → distant</button>
      <button @click="manualSync" style="margin-left:0.5rem;">sync (2 sens)</button>
    </section>

    <!-- FORM CRUD -->
    <form @submit.prevent="submitForm"
      style="margin-top:1.5rem;padding:1rem;background:#fff;border:1px solid #ccc;border-radius:8px;">
      <h2>{{ isEdit ? 'Modifier' : 'Créer' }}</h2>

      <label>
        Nom :
        <input v-model="form.name" required style="width:100%;margin-top:0.2rem;" />
      </label>

      <label style="margin-top:1rem;">
        Contenu :
        <textarea v-model="form.content" required style="width:100%;min-height:70px;margin-top:0.2rem;"></textarea>
      </label>

      <div style="margin-top:1rem;">
        <button type="submit">{{ isEdit ? 'Enregistrer' : 'Créer' }}</button>

        <button v-if="isEdit" type="button" @click="resetForm" style="margin-left:0.5rem;">
          Annuler
        </button>

        <button type="button" @click="generateFake(50)" style="margin-left:0.5rem;">
          Générer 50 docs
        </button>
      </div>
    </form>

    <!-- RECHERCHE + TRI -->
    <section style="margin-top:1.5rem;padding:1rem;background:#fff;border:1px solid #ccc;border-radius:8px;">
      <h2>Recherche (indexée)</h2>

      <input v-model="searchTerm" @input="onSearch" placeholder="Nom exact…" style="width:100%;padding:0.4rem;" />

      <p style="margin-top:0.5rem;">
        Tri :
        <strong>{{ sortByLikes ? 'par likes (index)' : 'par date (index)' }}</strong>
      </p>

      <button @click="toggleSortLikes">
        {{ sortByLikes ? 'Trier par date' : 'Trier par likes' }}
      </button>
    </section>

    <!-- LISTE DOCS -->
    <section style="margin-top:1.5rem;padding:1rem;background:#fff;border:1px;">
      <h2>Documents</h2>

      <p v-if="docs.length === 0">Aucun document.</p>

      <div v-for="doc in docs" :key="doc._id"
        style="margin-bottom:1rem;padding-bottom:1rem;border-bottom:1px solid #ddd;">
        <h3>{{ doc.name }}</h3>
        <p>{{ doc.content }}</p>

        <p style="color:#666;font-size:0.85rem;">
          Créé : {{ doc.created_at }}<br />
          <span v-if="doc.updated_at">MàJ : {{ doc.updated_at }}<br /></span>
          Likes : {{ doc.likes }}
        </p>

        <!-- Commentaires -->
        <div style="margin-top:0.5rem;">
          <h4>Commentaires</h4>

          <ul v-if="doc.comments.length">
            <li v-for="(c, idx) in doc.comments" :key="idx">
              {{ c.text }}
              <small style="color:#666;">({{ c.created_at }})</small>
            </li>
          </ul>

          <p v-else>Aucun commentaire.</p>

          <div style="margin-top:0.4rem;display:flex;gap:0.5rem;">
            <input :value="getDraft(doc._id!)" @input="onCommentInput(doc._id!, $event)"
              placeholder="Ajouter commentaire…" style="flex:1;padding:0.3rem;" />
            <button @click="addComment(doc)">Ajouter</button>
          </div>
        </div>

        <div style="margin-top:0.7rem;">
          <button @click="startEdit(doc)">Modifier</button>
          <button @click="deleteDoc(doc)" style="margin-left:0.4rem;">Supprimer</button>
          <button @click="likeDoc(doc)" style="margin-left:0.4rem;">❤️ Like</button>
        </div>
      </div>
    </section>
  </section>
</template>

<style scoped>
button {
  padding: 0.4rem 0.8rem;
  background: #f6f6f6;
  border: 1px solid #ccc;
  border-radius: 6px;
  cursor: pointer;
}

button:hover {
  background: #eaeaea;
}
</style>