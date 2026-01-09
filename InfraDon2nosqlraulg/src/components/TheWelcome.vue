<script setup lang="ts">
  import { ref, onMounted, onBeforeUnmount } from 'vue'
  import PouchDB from 'pouchdb'
  import PouchDBFind from 'pouchdb-find'

  /**
   * -------------------------------------------------------
   * FIX TS (rouge) :
   * Si ton projet n’a pas @types/pouchdb-find, TypeScript ne connaît pas
   * db.find() / db.createIndex(). On ajoute une augmentation minimale ici,
   * pour être 100% clean côté TS sans dépendre de tes typings.
   * -------------------------------------------------------
   */
  declare module 'pouchdb' {
    interface Database<Content extends {} = {}> {
      createIndex(request: { index: { fields: string[] } }): Promise<any>
      find<T = any>(request: {
        selector: Record<string, any>
        sort?: Array<Record<string, 'asc' | 'desc'>>
        limit?: number
        skip?: number
      }): Promise<{ docs: T[] }>
      getAttachment(docId: string, attachmentId: string): Promise<Blob>
      putAttachment(
        docId: string,
        attachmentId: string,
        rev: string,
        blob: Blob,
        type: string
      ): Promise<{ ok: boolean; id: string; rev: string }>
      removeAttachment(
        docId: string,
        attachmentId: string,
        rev: string
      ): Promise<{ ok: boolean; id: string; rev: string }>
    }
  }

  PouchDB.plugin(PouchDBFind)

  /* -------------------------------------------------------
   * TYPES
   * ----------------------------------------------------- */

  type DocType = 'doc' | 'comment'
  type SyncStatus = 'inactive' | 'active' | 'paused' | 'error'

  interface InfraComment {
    _id?: string
    _rev?: string
    type: 'comment'
    docId: string
    text: string
    created_at: string
    updated_at?: string
  }

  interface InfraDoc {
    _id?: string
    _rev?: string
    type: 'doc'
    _attachments?: Record<string, unknown>
    name: string
    content: string
    created_at: string
    updated_at?: string
    likes: number
  }

  type LocalDB = PouchDB.Database<any>

  /* -------------------------------------------------------
   * STATE
   * ----------------------------------------------------- */

  const LOCAL_DB = 'vini_local'
  const REMOTE_DB = 'http://admin:MbappeVini135@127.0.0.1:5984/vini'

  const localDb = ref<LocalDB | null>(null)
  const remoteDb = ref<PouchDB.Database<any> | null>(null)

  const docs = ref<InfraDoc[]>([])
  const comments = ref<InfraComment[]>([])

  const loading = ref(false)
  const error = ref('')

  const online = ref(true)
  const syncStatus = ref<SyncStatus>('inactive')

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

  const page = ref(0)
  const PAGE_SIZE = 10

  const topLikedMode = ref(false)

  const showAllComments = ref<Record<string, boolean | undefined>>({})

  /** Cache d’ObjectURLs (évite fuite mémoire) */
  const attachmentUrls = ref<Record<string, string>>({})

  /** Preview attachment state */
  const previewAttachment = ref<{
    docId: string
    name: string
    url: string
    type: string
  } | null>(null)

  let syncHandler: any = null

  function cleanupDocAttachmentUrls(docId: string): void {
    const prefix = `${docId}::`
    for (const k of Object.keys(attachmentUrls.value)) {
      if (k.startsWith(prefix)) {
        URL.revokeObjectURL(attachmentUrls.value[k])
        delete attachmentUrls.value[k]
      }
    }

    if (previewAttachment.value && previewAttachment.value.docId === docId) {
      URL.revokeObjectURL(previewAttachment.value.url)
      previewAttachment.value = null
    }
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

  function initRemoteDb(): PouchDB.Database<any> {
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
      type: 'doc',
      _attachments: doc._attachments,
      name: doc.name ?? '',
      content: doc.content ?? '',
      created_at: doc.created_at ?? new Date().toISOString(),
      updated_at: doc.updated_at,
      likes: typeof doc.likes === 'number' ? doc.likes : 0
    }
  }

  function normalizeComment(raw: unknown): InfraComment {
    const c = raw as Partial<InfraComment>
    return {
      _id: c._id,
      _rev: c._rev,
      type: 'comment',
      docId: c.docId ?? '',
      text: c.text ?? '',
      created_at: c.created_at ?? new Date().toISOString(),
      updated_at: c.updated_at
    }
  }

  /* -------------------------------------------------------
   * INDEXATION (OBLIGATOIRE si sort via db.find)
   * ----------------------------------------------------- */

  async function ensureIndexes(): Promise<void> {
    const db = initLocalDb()

    /**
     * Consigne:
     * - Pas de tri JS (.sort()).
     * - Tous les tris/recherches passent par PouchDB/CouchDB (db.find).
     * - Si on fait un sort, il faut un index compatible.
     * - Docs + comments dans la même DB => champ "type".
     */

    await db.createIndex({ index: { fields: ['type', 'created_at'] } })
    await db.createIndex({ index: { fields: ['type', 'likes', 'created_at'] } })
    await db.createIndex({ index: { fields: ['type', 'name'] } })
    await db.createIndex({ index: { fields: ['type', 'docId', 'created_at'] } })
  }

  /* -------------------------------------------------------
   * FETCH COMMENTS (1 requête pour les docs affichés)
   * ----------------------------------------------------- */

  async function fetchCommentsForDocs(docIds: string[]): Promise<void> {
    const db = initLocalDb()

    if (!docIds.length) {
      comments.value = []
      return
    }

    const res = await db.find<InfraComment>({
      selector: { type: { $eq: 'comment' }, docId: { $in: docIds } },
      sort: [{ type: 'asc' }, { docId: 'asc' }, { created_at: 'asc' }]
    })

    comments.value = res.docs.map(normalizeComment)
  }

  /* -------------------------------------------------------
   * FETCH DOCS (indexé + pagination)
   * ----------------------------------------------------- */

  async function fetchDocs(): Promise<void> {
    const db = initLocalDb()
    loading.value = true
    error.value = ''

    try {
      if (sortByLikes.value) {
        const res = await db.find<InfraDoc>({
          selector: { type: { $eq: 'doc' as DocType }, likes: { $gte: 0 } },
          sort: [{ type: 'asc' }, { likes: 'desc' }, { created_at: 'desc' }],
          limit: PAGE_SIZE,
          skip: page.value * PAGE_SIZE
        })
        docs.value = res.docs.map(normalizeDoc)
      } else {
        const res = await db.find<InfraDoc>({
          selector: { type: { $eq: 'doc' as DocType }, created_at: { $gte: '\u0000' } },
          sort: [{ type: 'asc' }, { created_at: 'desc' }],
          limit: PAGE_SIZE,
          skip: page.value * PAGE_SIZE
        })
        docs.value = res.docs.map(normalizeDoc)
      }

      const ids = docs.value.map(d => d._id).filter((x): x is string => typeof x === 'string')
      await fetchCommentsForDocs(ids)
    } catch (e) {
      error.value = (e as Error).message
    } finally {
      loading.value = false
    }
  }

  async function fetchTopLiked(): Promise<void> {
    const db = initLocalDb()
    loading.value = true
    error.value = ''

    try {
      const res = await db.find<InfraDoc>({
        selector: { type: { $eq: 'doc' as DocType }, likes: { $gte: 0 } },
        sort: [{ type: 'asc' }, { likes: 'desc' }, { created_at: 'desc' }],
        limit: 10,
        skip: 0
      })

      docs.value = res.docs.map(normalizeDoc)

      const ids = docs.value.map(d => d._id).filter((x): x is string => typeof x === 'string')
      await fetchCommentsForDocs(ids)
    } catch (e) {
      error.value = (e as Error).message
    } finally {
      loading.value = false
    }
  }

  function refreshList(): void {
    if (topLikedMode.value) void fetchTopLiked()
    else void fetchDocs()
  }

  /* -------------------------------------------------------
   * CRUD DOCS
   * ----------------------------------------------------- */

  function resetForm(): void {
    isEdit.value = false
    form.value = { _id: '', _rev: '', name: '', content: '' }
  }

  async function createDoc(): Promise<void> {
    const db = initLocalDb()

    const newDoc: InfraDoc = {
      type: 'doc',
      name: form.value.name.trim(),
      content: form.value.content.trim(),
      created_at: new Date().toISOString(),
      likes: 0
    }

    await db.post(newDoc)
    resetForm()

    if (online.value) await manualSync()
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
      type: 'doc',
      name: form.value.name.trim(),
      content: form.value.content.trim(),
      updated_at: new Date().toISOString()
    }

    await db.put(updated)
    resetForm()

    if (online.value) await manualSync()
    refreshList()
  }

  async function deleteDoc(doc: InfraDoc): Promise<void> {
    const db = initLocalDb()
    if (!doc._id) return

    cleanupDocAttachmentUrls(doc._id)

    // Supprimer aussi les commentaires liés
    const related = await db.find<InfraComment>({
      selector: { type: { $eq: 'comment' }, docId: { $eq: doc._id } }
    })

    if (related.docs.length) {
      const toDelete = related.docs
        .filter(c => c._id && c._rev)
        .map(c => ({ _id: c._id as string, _rev: c._rev as string, _deleted: true }))

      await db.bulkDocs(toDelete as any)
    }

    const latest = await db.get(doc._id)
    await db.remove(latest._id as string, latest._rev as string)

    if (online.value) await manualSync()
    refreshList()
  }

  async function submitForm(): Promise<void> {
    if (isEdit.value) await updateDoc()
    else await createDoc()
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
      type: 'doc',
      likes: normalized.likes + 1
    }

    await db.put(updated)

    if (online.value) await manualSync()
    refreshList()
  }

  function toggleSortLikes(): void {
    sortByLikes.value = !sortByLikes.value
    topLikedMode.value = false
    page.value = 0
    void fetchDocs()
  }

  /* -------------------------------------------------------
   * COMMENTAIRES (AJOUT)
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
      type: 'comment',
      docId: doc._id,
      text: draft.trim(),
      created_at: new Date().toISOString()
    }

    await db.post(newComment)
    commentDrafts.value[doc._id] = ''

    if (online.value) await manualSync()
    refreshList()
  }

  /* -------------------------------------------------------
   * RECHERCHE (INDEXÉE)
   * ----------------------------------------------------- */

  async function onSearch(): Promise<void> {
    const db = initLocalDb()
    const term = searchTerm.value.trim()

    if (!term) {
      refreshList()
      return
    }

    const res = await db.find<InfraDoc>({
      selector: { type: { $eq: 'doc' as DocType }, name: { $eq: term } },
      limit: PAGE_SIZE,
      skip: 0
    })

    topLikedMode.value = false
    page.value = 0

    docs.value = res.docs.map(normalizeDoc)

    const ids = docs.value.map(d => d._id).filter((x): x is string => typeof x === 'string')
    await fetchCommentsForDocs(ids)
  }

  /* -------------------------------------------------------
   * FACTORY
   * ----------------------------------------------------- */

  async function generateFake(n = 50): Promise<void> {
    const db = initLocalDb()
    const now = Date.now()

    const data: InfraDoc[] = Array.from({ length: n }, (_, i) => ({
      _id: `fake_${now}_${i}`,
      type: 'doc',
      name: `Doc ${i}`,
      content: `Contenu numéro ${i}`,
      created_at: new Date().toISOString(),
      likes: Math.floor(Math.random() * 10)
    }))

    await db.bulkDocs(data as any)

    if (online.value) await manualSync()
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

    void addAttachment(doc, file)
    input.value = ''
  }

  async function addAttachment(doc: InfraDoc, file: File): Promise<void> {
    const db = initLocalDb()
    if (!doc._id || !doc._rev) return

    const r = await db.putAttachment(doc._id, file.name, doc._rev, file, file.type)
    if (r?.rev) doc._rev = r.rev

    if (online.value) await manualSync()
    refreshList()
  }

  async function openAttachment(doc: InfraDoc, name: string): Promise<void> {
    const db = initLocalDb()
    if (!doc._id) return

    // toggle preview
    if (
      previewAttachment.value &&
      previewAttachment.value.docId === doc._id &&
      previewAttachment.value.name === name
    ) {
      URL.revokeObjectURL(previewAttachment.value.url)
      previewAttachment.value = null
      return
    }

    // fermer l’ancienne preview si différente
    if (previewAttachment.value) {
      URL.revokeObjectURL(previewAttachment.value.url)
      previewAttachment.value = null
    }

    const cacheKey = `${doc._id}::${name}`

    // refetch pour avoir un type fiable
    const blob: Blob = await db.getAttachment(doc._id, name)
    const url = URL.createObjectURL(blob)

    // remplace / met en cache
    if (attachmentUrls.value[cacheKey]) {
      URL.revokeObjectURL(attachmentUrls.value[cacheKey])
    }
    attachmentUrls.value[cacheKey] = url

    previewAttachment.value = { docId: doc._id, name, url, type: blob.type }
  }

  async function deleteAttachment(doc: InfraDoc, name: string): Promise<void> {
    const db = initLocalDb()
    if (!doc._id || !doc._rev) return

    if (
      previewAttachment.value &&
      previewAttachment.value.docId === doc._id &&
      previewAttachment.value.name === name
    ) {
      URL.revokeObjectURL(previewAttachment.value.url)
      previewAttachment.value = null
    }

    const cacheKey = `${doc._id}::${name}`
    if (attachmentUrls.value[cacheKey]) {
      URL.revokeObjectURL(attachmentUrls.value[cacheKey])
      delete attachmentUrls.value[cacheKey]
    }

    const r = await db.removeAttachment(doc._id, name, doc._rev)
    if (r?.rev) doc._rev = r.rev

    if (online.value) await manualSync()
    refreshList()
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
    sortByLikes.value = false
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

    syncStatus.value = 'active'

    syncHandler = local
      .sync(remote, { live: true, retry: true })
      .on('change', () => refreshList())
      .on('paused', () => {
        if (online.value) syncStatus.value = 'paused'
      })
      .on('active', () => {
        if (online.value) syncStatus.value = 'active'
      })
      .on('denied', () => {
        syncStatus.value = 'error'
      })
      .on('error', (e: any) => {
        error.value = e?.message ?? 'Erreur de synchronisation'
        syncStatus.value = 'error'
      })
  }

  function stopLiveSync(): void {
    syncHandler?.cancel()
    syncHandler = null
    syncStatus.value = 'inactive'
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
   * MOUNT / UNMOUNT
   * ----------------------------------------------------- */

  onMounted(async () => {
    initLocalDb()
    initRemoteDb()

    await ensureIndexes()

    if (online.value) {
      await manualSync()
    }

    await fetchDocs()
    startLiveSync()
  })

  onBeforeUnmount(() => {
    stopLiveSync()

    if (previewAttachment.value) {
      URL.revokeObjectURL(previewAttachment.value.url)
      previewAttachment.value = null
    }

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
              background: online && (syncStatus === 'active' || syncStatus === 'paused') ? '#ccf3cc' : '#f8cccc',
              border: '1px solid #ccc'
            }"
          >
            Sync :
            {{
              online
                ? (syncStatus === 'paused' ? 'active (idle)' : syncStatus)
                : 'inactive'
            }}
          </span>
        </div>
      </header>

      <div v-if="loading" style="margin-top: 1rem; color: #666;">Chargement…</div>
      <div v-if="error" style="margin-top: 1rem; color: #c00;">{{ error }}</div>

      <!-- RÉPLICATION -->
      <section style="margin-top: 1.5rem; padding: 1rem; background: #fff; border: 1px solid #ccc; border-radius: 8px;">
        <h2 style="margin-top: 0;">Réplication manuelle</h2>
        <button type="button" @click="replicateFromRemote">distant → local</button>
        <button type="button" @click="replicateToRemote" style="margin-left: 0.5rem;">local → distant</button>
        <button type="button" @click="manualSync" style="margin-left: 0.5rem;">sync (2 sens)</button>
      </section>

      <!-- CRUD -->
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

      <!-- RECHERCHE / TRI -->
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
          <button type="button" @click="toggleSortLikes" :disabled="topLikedMode">
            {{ sortByLikes ? 'Trier par date' : 'Trier par likes' }}
          </button>

          <button type="button" @click="toggleTopLiked">
            {{ topLikedMode ? 'Revenir à la liste' : 'Afficher Top 10 likés' }}
          </button>
        </div>
      </section>

      <!-- PAGINATION -->
      <section
        v-if="!topLikedMode && !searchTerm.trim()"
        style="margin-top: 1rem; padding: 1rem; background: #fff; border: 1px solid #ccc; border-radius: 8px;"
      >
        <h2 style="margin-top: 0;">Pagination</h2>
        <p>Page actuelle : <strong>{{ page + 1 }}</strong> ({{ PAGE_SIZE }} documents par page)</p>

        <button type="button" @click="prevPage" :disabled="page === 0">Précédent</button>
        <button type="button" @click="nextPage" style="margin-left: 0.5rem;">Suivant</button>
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

          <!-- Dernier commentaire -->
          <p v-if="lastComment(doc) && !showAllComments[doc._id]" style="margin-top: 0.5rem;">
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
                  <small v-if="c.updated_at" style="color: #666;"> — modifié: {{ c.updated_at }}</small>
                </div>
              </li>
            </ul>

            <p v-else-if="doc._id && showAllComments[doc._id] && comments.filter(c => c.docId === doc._id).length === 0">
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

          <!-- Assets -->
          <div style="margin-top: 0.75rem;">
            <strong>Médias :</strong>

            <div style="margin: 0.4rem 0;">
              <input type="file" @change="onFileSelected(doc, $event)" />
            </div>

            <ul v-if="doc._attachments">
              <li v-for="(_att, name) in doc._attachments" :key="String(name)">
                {{ name }}
                <button type="button" @click="openAttachment(doc, String(name))">Voir</button>
                <button type="button" @click="deleteAttachment(doc, String(name))">Supprimer</button>
              </li>
            </ul>

            <div v-if="previewAttachment && previewAttachment.docId === doc._id" style="margin-top: 0.75rem;">
              <div v-if="previewAttachment.type.startsWith('image/')">
                <img
                  :src="previewAttachment.url"
                  alt="preview"
                  style="max-width: 100%; border: 1px solid #ccc; border-radius: 4px;"
                />
              </div>

              <div v-else>
                <a :href="previewAttachment.url" target="_blank">Ouvrir le fichier</a>
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