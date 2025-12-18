<script setup lang="ts">
  import { ref, onMounted, onBeforeUnmount } from 'vue'
  import PouchDB from 'pouchdb'
  import PouchDBFind from 'pouchdb-find'


  // Activation du plugin find (nécessaire pour db.find + indexation)
  PouchDB.plugin(PouchDBFind)

  /* -------------------------------------------------------
   * TYPES
   * ----------------------------------------------------- */

  interface InfraComment {
    _id?: string
    _rev?: string
    // lien vers le document parent
    docId: string
    text: string
    created_at: string
  }

  interface InfraDoc {
    _id?: string
    _rev?: string

    // CouchDB native attachments (clé = nom du fichier)
    _attachments?: Record<string, unknown>

    name: string
    content: string
    created_at: string
    updated_at?: string
    likes: number
    // comments: InfraComment[] // supprimé : plus de champ imbriqué
    // attachments?: string[]   // supprimé si présent
  }

  // Typage simplifié pour calmer TS
  type LocalDB = PouchDB.Database

  /* -------------------------------------------------------
   * STATE
   * ----------------------------------------------------- */

  const LOCAL_DB = 'vini_local'
  const REMOTE_DB = 'http://admin:MbappeVini135@127.0.0.1:5984/vini'

  const localDb = ref<LocalDB | null>(null)
  const remoteDb = ref<PouchDB.Database<InfraDoc> | null>(null)

  const docs = ref<InfraDoc[]>([])
  // Nouvelle collection séparée pour les commentaires
  const comments = ref<InfraComment[]>([])
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

  // Pagination (consigne: afficher 10 documents et pouvoir afficher les 10 suivants)
  const page = ref(0)
  const PAGE_SIZE = 10

  // Mode "Top 10 likés"
  const topLikedMode = ref(false)

  // UI: afficher tous les commentaires ou uniquement le dernier
  const showAllComments = ref<Record<string, boolean | undefined>>({})

  // Cache des URLs d’attachments (évite de refetch à chaque render)
  const attachmentUrls = ref<Record<string, string>>({})

  // Preview attachment state
  const previewAttachment = ref<{
    docId: string
    name: string
    url: string
    type: string
  } | null>(null)

  let syncHandler: any = null
  function cleanupDocAttachmentUrls(_docId: string): void {
    // Fonction conservée pour compatibilité avec la structure existante
    // (pas de cache d’URL utilisé dans la version actuelle)
  }

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
      _attachments: doc._attachments,
      name: doc.name ?? '',
      content: doc.content ?? '',
      created_at: doc.created_at ?? new Date().toISOString(),
      updated_at: doc.updated_at,
      likes: typeof doc.likes === 'number' ? doc.likes : 0
      // plus de comments ni attachments ici (stockage séparé)
    }
  }

  /* -------------------------------------------------------
   * INDEXES
   * ----------------------------------------------------- */

  async function ensureIndexes(): Promise<void> {
    const db = initLocalDb()

    /**
     * IMPORTANT (consigne TP):
     * - Les tris ne doivent pas être exécutés en TypeScript (pas de .sort() JS).
     * - Avec PouchDB Find, si on fait un sort, il faut un index qui "match" le sort.
     *   Sinon: erreur "no_usable_index".
     */

    // Pour: sort: [{ created_at: 'desc' }]
    await db.createIndex({
      index: { fields: ['created_at'] }
    })

    // Pour: sort: [{ likes: 'desc' }, { created_at: 'desc' }]
    await db.createIndex({
      index: { fields: ['likes', 'created_at'] }
    })

    // Pour: recherche exact sur name
    await db.createIndex({
      index: { fields: ['name'] }
    })

    // Index pour la collection comments (si elle existe)
    // On suppose que c'est la même base, mais collection séparée logique
    // Pour recherche sur docId + tri created_at
    await db.createIndex({
      index: { fields: ['docId', 'created_at'] }
    })
  }
  // FETCH / COMMENTAIRES
  async function fetchCommentsForDoc(docId: string): Promise<InfraComment[]> {
    const db = initLocalDb()
    const res = await db.find<InfraComment>({
      selector: { docId },
      sort: [{ docId: 'asc' }, { created_at: 'asc' }]
    })
    return res.docs
  }

  /* -------------------------------------------------------
   * FETCH DOCS
   * ----------------------------------------------------- */

  async function fetchDocs(): Promise<void> {
    const db = initLocalDb()
    loading.value = true
    error.value = ''
    try {
      const res = await db.allDocs({
        include_docs: true,
        attachments: true,
        limit: PAGE_SIZE,
        skip: page.value * PAGE_SIZE
      })

      docs.value = res.rows
        .map(row => row.doc)
        .filter(
          (doc): doc is InfraDoc =>
            !!doc && typeof doc._id === 'string' && !doc._id.startsWith('_design/')
        )
        .map(normalizeDoc)

      // Recharger tous les commentaires liés aux documents affichés
      const allComments: InfraComment[] = []
      for (const d of docs.value) {
        if (!d._id) continue
        const res = await fetchCommentsForDoc(d._id)
        allComments.push(...res)
      }
      comments.value = allComments
    } catch (e) {
      const err = e as Error
      error.value = err.message
    } finally {
      loading.value = false
    }
  }

  async function fetchTopLiked(): Promise<void> {
    const db = initLocalDb()
    loading.value = true
    error.value = ''

    /**
     * Mode "Top 10 likés" (consigne TP):
     * - Requête optimisée: limit 10 + tri côté PouchDB via index likes.
     * - Pas de tri JS.
     */
    try {
      const res = await db.find<InfraDoc>({
        selector: { likes: { $gte: 0 } },
        sort: [{ likes: 'desc' }, { created_at: 'desc' }],
        limit: 10,
        skip: 0
      })
      docs.value = res.docs.map(normalizeDoc)
      // Charger tous les commentaires liés aux documents affichés
      const allComments: InfraComment[] = []
      for (const d of docs.value) {
        if (!d._id) continue
        const res = await fetchCommentsForDoc(d._id)
        allComments.push(...res)
      }
      comments.value = allComments
    } catch (e) {
      const err = e as Error
      error.value = err.message
    } finally {
      loading.value = false
    }
  }

  function refreshList(): void {
    if (topLikedMode.value) {
      void fetchTopLiked()
    } else {
      void fetchDocs()
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
      likes: 0
    }

    await db.post(newDoc)
    resetForm()

    // Re-synchroniser uniquement si online (mécanique de réplication conservée)
    if (online.value) {
      await manualSync()
    }

    refreshList()
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
    resetForm()

    if (online.value) {
      await manualSync()
    }

    refreshList()
  }

  async function deleteDoc(doc: InfraDoc): Promise<void> {
    const db = initLocalDb()
    if (!doc._id) return

    // Nettoyage cache preview attachments (évite fuite mémoire)
    cleanupDocAttachmentUrls(doc._id)

    const latest = await db.get(doc._id)
    await db.remove(latest._id as string, latest._rev as string)

    if (online.value) {
      await manualSync()
    }

    refreshList()
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
    if (!doc._id) return

    const db = initLocalDb()
    const latest = await db.get(doc._id)
    const normalized = normalizeDoc(latest)

    const updated: InfraDoc = {
      ...normalized,
      likes: normalized.likes + 1
    }

    await db.put(updated)

    if (online.value) {
      await manualSync()
    }

    refreshList()
  }

  function toggleSortLikes(): void {
    sortByLikes.value = !sortByLikes.value
    topLikedMode.value = false
    page.value = 0
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

  function lastComment(doc: InfraDoc): InfraComment | null {
    const related = comments.value.filter(c => c.docId === doc._id)
    return related.length ? related[related.length - 1] : null
  }

  function toggleComments(docId?: string): void {
    if (!docId) return
    showAllComments.value[docId] = !showAllComments.value[docId]
  }

  async function addComment(doc: InfraDoc): Promise<void> {
    if (!doc._id) return
    const db = initLocalDb()
    const draft = getDraft(doc._id)
    if (!draft.trim()) return

    const newComment: InfraComment = {
      docId: doc._id,
      text: draft.trim(),
      created_at: new Date().toISOString()
    }

    await db.post(newComment)
    commentDrafts.value[doc._id] = ''

    refreshList()
  }

  // À adapter si besoin : suppression et édition de commentaire dans la collection séparée
  // (non demandé dans la consigne principale)

  /* -------------------------------------------------------
   * SEARCH (INDEXÉE)
   * ----------------------------------------------------- */

  async function onSearch(): Promise<void> {
    const db = initLocalDb()
    const term = searchTerm.value.trim()

    // Si on vide la recherche, on revient au mode normal (ou top liked si activé)
    if (!term) {
      refreshList()
      return
    }

    // Recherche exacte sur name (index name)
    const res = await db.find<InfraDoc>({
      selector: { name: { $eq: term } },
      limit: PAGE_SIZE,
      skip: 0
    })

    // En recherche, on désactive pagination / topLiked pour éviter ambiguïté UI
    topLikedMode.value = false
    page.value = 0
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
        likes: Math.floor(Math.random() * 10)
      })
    }

    await db.bulkDocs(data)

    if (online.value) {
      await manualSync()
    }

    refreshList()
  }

  /* -------------------------------------------------------
   * ASSETS MANAGEMENT (Attachments)
   * ----------------------------------------------------- */

  function onFileSelected(doc: InfraDoc, event: Event): void {
    if (!doc._id || !doc._rev) return
    const input = event.target as HTMLInputElement
    const file = input.files?.[0]
    if (!file) return

    addAttachment(doc, file)

    // reset input pour permettre le même upload
    input.value = ''
  }

  async function addAttachment(doc: InfraDoc, file: File): Promise<void> {
    if (!localDb.value || !doc._id || !doc._rev) return

    await localDb.value.putAttachment(
      doc._id,
      file.name,
      doc._rev,
      file,
      file.type
    )

    const fresh = await localDb.value.get(doc._id)
    doc._rev = fresh._rev
    await fetchDocs()
  }

  async function openAttachment(doc: InfraDoc, name: string): Promise<void> {
    // Si le même média est déjà affiché → on le referme
    if (
      previewAttachment.value &&
      previewAttachment.value.docId === doc._id &&
      previewAttachment.value.name === name
    ) {
      URL.revokeObjectURL(previewAttachment.value.url)
      previewAttachment.value = null
      return
    }

    if (!localDb.value || !doc._id) return

    const blob: Blob = await localDb.value.getAttachment(doc._id, name)
    const url = URL.createObjectURL(blob)

    previewAttachment.value = {
      docId: doc._id,
      name,
      url,
      type: blob.type
    }
  }

  async function deleteAttachment(doc: InfraDoc, name: string): Promise<void> {
    if (!doc._id || !doc._rev) return
    if (!localDb.value || !doc._id || !doc._rev) return

    if (
      previewAttachment.value &&
      previewAttachment.value.docId === doc._id &&
      previewAttachment.value.name === name
    ) {
      URL.revokeObjectURL(previewAttachment.value.url)
      previewAttachment.value = null
    }

    await localDb.value.removeAttachment(doc._id, name, doc._rev)
    await fetchDocs()
  }

  /* -------------------------------------------------------
   * PAGINATION
   * ----------------------------------------------------- */

  function nextPage(): void {
    page.value++
    void fetchDocs()
  }

  function prevPage(): void {
    if (page.value > 0) page.value--
    void fetchDocs()
  }

  function toggleTopLiked(): void {
    topLikedMode.value = !topLikedMode.value
    page.value = 0
    sortByLikes.value = false // mode dédié top liked
    if (topLikedMode.value) void fetchTopLiked()
    else void fetchDocs()
  }

  /* -------------------------------------------------------
   * REPLICATION
   * ----------------------------------------------------- */

  async function replicateFromRemote(): Promise<void> {
    const local = initLocalDb()
    const remote = initRemoteDb()
    await local.replicate.from(remote)
    refreshList()
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
        refreshList()
      })
      .on('paused', () => {
        // connexion momentanément interrompue
        isSyncing.value = false
      })
      .on('active', () => {
        isSyncing.value = true
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
    await fetchDocs()
    startLiveSync()
  })

  onBeforeUnmount(() => {
    stopLiveSync()
    // Nettoyage des ObjectURLs
    for (const k of Object.keys(attachmentUrls.value)) {
      URL.revokeObjectURL(attachmentUrls.value[k])
    }
    attachmentUrls.value = {}
  })
</script>

<template>
  <section style="padding: 2rem; background: #fafafa; max-width: 1000px; margin: 0 auto;">
    <header style="display: flex; justify-content: space-between; align-items: center; flex-wrap: wrap; gap: 1rem;">
      <h1>InfraDon2 — CRUD + Réplication + Index + Attachments</h1>

      <div style="display: flex; gap: 1rem; align-items: center;">
        <label style="display: flex; align-items: center; gap: 0.4rem;">
          <input type="checkbox" :checked="online" @change="toggleOnline" />
          <span>{{ online ? 'ONLINE' : 'OFFLINE' }}</span>
        </label>

        <span
          :style="{
            padding: '0.3rem 0.6rem',
            borderRadius: '12px',
            fontSize: '0.8rem',
            background: online && isSyncing ? '#ccf3cc' : '#f8cccc',
            border: '1px solid #ccc'
          }"
        >
          Sync : {{ online && isSyncing ? 'active' : 'inactive' }}
        </span>
      </div>
    </header>

    <div v-if="loading" style="margin-top: 1rem; color: #666;">Chargement…</div>
    <div v-if="error" style="margin-top: 1rem; color: #c00;">{{ error }}</div>

    <!-- BARRE DE RÉPLICATION -->
    <section style="margin-top: 1.5rem; padding: 1rem; background: #fff; border: 1px solid #ccc; border-radius: 8px;">
      <h2 style="margin-top: 0;">Réplication manuelle</h2>
      <button @click="replicateFromRemote">distant → local</button>
      <button @click="replicateToRemote" style="margin-left: 0.5rem;">local → distant</button>
      <button @click="manualSync" style="margin-left: 0.5rem;">sync (2 sens)</button>
    </section>

    <!-- FORM CRUD -->
    <form
      @submit.prevent="submitForm"
      style="margin-top: 1.5rem; padding: 1rem; background: #fff; border: 1px solid #ccc; border-radius: 8px;"
    >
      <h2>{{ isEdit ? 'Modifier' : 'Créer' }}</h2>

      <label>
        Nom :
        <input v-model="form.name" required style="width: 100%; margin-top: 0.2rem;" />
      </label>

      <label style="margin-top: 1rem;">
        Contenu :
        <textarea v-model="form.content" required style="width: 100%; min-height: 70px; margin-top: 0.2rem;"></textarea>
      </label>

      <div style="margin-top: 1rem;">
        <button type="submit">{{ isEdit ? 'Enregistrer' : 'Créer' }}</button>

        <button v-if="isEdit" type="button" @click="resetForm" style="margin-left: 0.5rem;">
          Annuler
        </button>

        <button type="button" @click="generateFake(50)" style="margin-left: 0.5rem;">
          Générer 50 docs
        </button>
      </div>
    </form>

    <!-- RECHERCHE + TRI + TOP -->
    <section style="margin-top: 1.5rem; padding: 1rem; background: #fff; border: 1px solid #ccc; border-radius: 8px;">
      <h2>Recherche / Tri (indexés)</h2>

      <input
        v-model="searchTerm"
        @input="onSearch"
        placeholder="Nom exact…"
        style="width: 100%; padding: 0.4rem;"
      />

      <p style="margin-top: 0.5rem;">
        Mode :
        <strong>
          {{
            topLikedMode
              ? 'Top 10 likés (index)'
              : sortByLikes
                ? 'Tri par likes (index)'
                : 'Tri par date (index)'
          }}
        </strong>
      </p>

      <div style="display: flex; flex-wrap: wrap; gap: 0.5rem;">
        <button @click="toggleSortLikes" :disabled="topLikedMode">
          {{ sortByLikes ? 'Trier par date' : 'Trier par likes' }}
        </button>

        <button @click="toggleTopLiked">
          {{ topLikedMode ? 'Revenir à la liste' : 'Afficher Top 10 likés' }}
        </button>
      </div>
    </section>

    <!-- PAGINATION (désactivée en Top10 / recherche) -->
    <section
      v-if="!topLikedMode && !searchTerm.trim()"
      style="margin-top: 1rem; padding: 1rem; background: #fff; border: 1px solid #ccc; border-radius: 8px;"
    >
      <h2 style="margin-top: 0;">Pagination</h2>
      <p>Page actuelle : <strong>{{ page + 1 }}</strong> ({{ PAGE_SIZE }} documents par page)</p>

      <button @click="prevPage" :disabled="page === 0">Précédent</button>
      <button @click="nextPage" style="margin-left: 0.5rem;">Suivant</button>
    </section>

    <!-- LISTE DOCS -->
    <section style="margin-top: 1.5rem; padding: 1rem; background: #fff; border: 1px;">
      <h2>Documents</h2>

      <p v-if="docs.length === 0">Aucun document.</p>

      <div
        v-for="doc in docs"
        :key="doc._id"
        style="margin-bottom: 1rem; padding-bottom: 1rem; border-bottom: 1px solid #ddd;"
      >
        <h3>{{ doc.name }}</h3>
        <p>{{ doc.content }}</p>

        <p style="color: #666; font-size: 0.85rem;">
          Créé : {{ doc.created_at }}<br />
          <span v-if="doc.updated_at">MàJ : {{ doc.updated_at }}<br /></span>
          Likes : {{ doc.likes }}
        </p>

        <!-- Dernier commentaire (consigne TP) -->
        <p
          v-if="lastComment(doc) && !showAllComments[doc._id]"
          style="margin-top: 0.5rem;"
        >
          Dernier commentaire :
          <strong>{{ lastComment(doc)?.text }}</strong>
        </p>

        <!-- Commentaires -->
        <div style="margin-top: 0.5rem;">
          <h4>Commentaires</h4>

          <div style="display: flex; gap: 0.5rem; align-items: center; flex-wrap: wrap;">
            <button type="button" @click="toggleComments(doc._id)">
              {{
                showAllComments[doc._id]
                  ? 'Masquer (dernier seulement)'
                  : 'Voir tous les commentaires'
              }}
            </button>
          </div>

          <!-- Liste complète -->
          <ul
            v-if="doc._id && showAllComments[doc._id] && comments.filter(c => c.docId === doc._id).length"
            style="margin-top: 0.5rem;"
          >
            <li
              v-for="(c, idx) in comments.filter(c => c.docId === doc._id)"
              :key="c._id ?? idx"
              style="margin-bottom: 0.4rem;"
            >
              <div>
                {{ c.text }}
                <small style="color: #666;">({{ c.created_at }})</small>
              </div>

            </li>
          </ul>

          <p
            v-else-if="doc._id && showAllComments[doc._id] && comments.filter(c => c.docId === doc._id).length === 0"
          >
            Aucun commentaire.
          </p>

          <div style="margin-top: 0.4rem; display: flex; gap: 0.5rem;">
            <input
              :value="doc._id ? getDraft(doc._id) : ''"
              @input="doc._id && onCommentInput(doc._id, $event)"
              placeholder="Ajouter commentaire…"
              style="flex: 1; padding: 0.3rem;"
            />
            <button type="button" @click="addComment(doc)">Ajouter</button>
          </div>
        </div>

        <!-- Assets / Attachments -->
        <div style="margin-top: 0.75rem;">
          <strong>Médias :</strong>

          <div style="margin: 0.4rem 0;">
            <input type="file" @change="onFileSelected(doc, $event)" />
          </div>

          <ul v-if="doc._attachments">
            <li v-for="(att, name) in doc._attachments" :key="name">
              {{ name }}
              <button @click="openAttachment(doc, name)">Voir</button>
              <button @click="deleteAttachment(doc, name)">Supprimer</button>
            </li>
          </ul>

          <!-- Preview média -->
          <div
            v-if="previewAttachment && previewAttachment.docId === doc._id"
            style="margin-top: 0.75rem;"
          >
            <div v-if="previewAttachment.type.startsWith('image/')">
              <img
                :src="previewAttachment.url"
                alt="preview"
                style="max-width: 100%; border: 1px solid #ccc; border-radius: 4px;"
              />
            </div>

            <div v-else>
              <a :href="previewAttachment.url" target="_blank">
                Ouvrir le fichier
              </a>
            </div>
          </div>
        </div>

        <!-- Actions doc -->
        <div style="margin-top: 0.9rem;">
          <button type="button" @click="startEdit(doc)">Modifier</button>
          <button type="button" @click="deleteDoc(doc)" style="margin-left: 0.4rem;">Supprimer</button>
          <button type="button" @click="likeDoc(doc)" style="margin-left: 0.4rem;">Like</button>
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

  button:disabled {
    opacity: 0.6;
    cursor: not-allowed;
  }
</style>