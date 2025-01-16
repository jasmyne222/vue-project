<script lang="ts">
import { ref } from 'vue'
import PouchDB from 'pouchdb'
import PouchDBFind from 'pouchdb-find'

PouchDB.plugin(PouchDBFind)

declare interface Post {
  _id: string
  _rev?: string
  doc: {
    post_name: string
    post_content: string
    attributes: {
      creation_date: string
    }
  }
  _attachments?: { [key: string]: any } // Médias associés
}

export default {
  data() {
    return {
      postsData: [] as Post[], // Liste des documents
      document: null as Post | null, // Document sélectionné pour modification
      storage: null as PouchDB.Database | null, // Base de données locale
      newContent: '', // Contenu modifié pour le formulaire
      fakeDocumentId: 1, // ID pour générer des documents factices
      searchQuery: '', // Champ de recherche
      selectedFiles: [] as File[] // Liste des fichiers sélectionnés
    }
  },

  mounted() {
    this.initDatabase()
    this.createIndex() // Créer un index au démarrage
    this.fetchData() // Charger les documents
    this.watchChanges() // Écouter les changements en temps réel
  },

  methods: {
    // **1. Initialisation de la base de données**
    initDatabase() {
      const localDb = new PouchDB('local_database') // Base locale
      const remoteDb = new PouchDB('http://admin:admin@localhost:5984/database') // Base distante

      this.storage = localDb
      console.log('Base locale initialisée :', localDb.name)
      console.log('Base distante configurée :', remoteDb.name)

      this.syncDatabases(localDb, remoteDb) // Synchronisation automatique
    },

    // **2. Créer un index pour `post_name`**
    createIndex() {
      const db = this.storage
      if (db) {
        db.createIndex({
          index: { fields: ['doc.post_name'] }
        })
          .then(() => {
            console.log('Index créé avec succès sur `doc.post_name`.')
          })
          .catch((error) => {
            console.error('Erreur lors de la création de l’index :', error)
          })
      }
    },

    // **3. Rechercher des documents**
    searchByAttribute() {
      const db = this.storage
      if (db) {
        if (!this.searchQuery.trim()) {
          alert('Veuillez entrer un nom de post pour rechercher.')
          return
        }

        db.find({
          selector: {
            'doc.post_name': { $regex: new RegExp(this.searchQuery, 'i') }
          }
        })
          .then((result) => {
            this.postsData = result.docs as Post[]
            console.log('Résultats de la recherche :', this.postsData)
          })
          .catch((error) => {
            console.error('Erreur lors de la recherche :', error)
          })
      }
    },

    // **4. Récupérer tous les documents**
    fetchData() {
      const db = this.storage
      if (db) {
        db.allDocs({ include_docs: true, attachments: true })
          .then((result) => {
            this.postsData = result.rows.map((row) => row.doc)
            console.log('Documents récupérés :', this.postsData)
          })
          .catch((error) => {
            console.error('Erreur lors de la récupération des données :', error)
          })
      }
    },

    // **5. Ajouter un document factice**
    generateFakeDocument() {
      const newId = `post_${this.fakeDocumentId++}`
      return {
        _id: newId,
        doc: {
          post_name: `Post ${this.fakeDocumentId}`,
          post_content: 'Ceci est un contenu généré pour un document.',
          attributes: {
            creation_date: new Date().toISOString()
          }
        }
      } as Post
    },

    addFakeDocument() {
      const db = this.storage
      const fakeDoc = this.generateFakeDocument()
      if (db) {
        db.put(fakeDoc)
          .then((response) => {
            console.log('Document ajouté avec succès :', response)
            this.postsData.push({
              ...fakeDoc,
              _id: response.id,
              _rev: response.rev
            })
          })
          .catch((error) => {
            console.error('Erreur lors de l’ajout du document :', error)
          })
      }
    },

    // **6. Ajouter des médias à un document**
    async addMediaToDocument(document: Post) {
      const db = this.storage
      if (!this.selectedFiles.length) {
        alert('Veuillez sélectionner un ou plusieurs fichiers.')
        return
      }

      if (db) {
        try {
          const updatedDoc = await db.get(document._id)

          for (const file of this.selectedFiles) {
            const arrayBuffer = await file.arrayBuffer()
            await db.putAttachment(
              updatedDoc._id,
              file.name,
              updatedDoc._rev,
              arrayBuffer,
              file.type
            )
            console.log(`Fichier ajouté : ${file.name}`)
          }

          this.selectedFiles = []
          this.fetchData()
        } catch (error) {
          console.error('Erreur lors de l’ajout des médias :', error)
        }
      }
    },

    // **7. Sélectionner des fichiers**
    onFileChange(event: Event) {
      const input = event.target as HTMLInputElement
      if (input.files) {
        this.selectedFiles = Array.from(input.files)
        console.log(
          'Fichiers sélectionnés :',
          this.selectedFiles.map((file) => file.name)
        )
      }
    },

    // **8. Synchronisation locale et distante**
    syncDatabases(localDb: PouchDB.Database, remoteDb: PouchDB.Database) {
      PouchDB.sync(localDb, remoteDb, { live: true, retry: true })
        .on('change', (info) => {
          console.log('Changements détectés :', info)
        })
        .on('paused', (err) => {
          console.log('Synchronisation en pause :', err || 'Aucune erreur')
        })
        .on('active', () => {
          console.log('Synchronisation réactivée.')
        })
        .on('error', (err) => {
          console.error('Erreur lors de la synchronisation :', err)
        })
    },

    watchChanges() {
      const db = this.storage
      if (db) {
        db.changes({ since: 'now', live: true })
          .on('change', (change) => {
            console.log('Changement détecté :', change)
            this.fetchData()
          })
          .on('error', (err) => {
            console.error("Erreur lors de l'écoute des changements :", err)
          })
      }
    }
  }
}
</script>

<template>
  <div>
    <h1>Nombre de posts : {{ postsData.length }}</h1>

    <!-- Recherche -->
    <div>
      <input type="text" v-model="searchQuery" placeholder="Entrez un nom de post" />
      <button @click="searchByAttribute">Rechercher</button>
    </div>

    <!-- Liste des documents -->
    <ul>
      <li v-for="post in postsData" :key="post._id">
        <div>
          <span>{{ post.doc.post_name }}</span>
        </div>

        <!-- Médias associés -->
        <div v-if="post._attachments">
          <h4>Médias associés :</h4>
          <ul>
            <li v-for="(attachment, name) in post._attachments" :key="name">
              {{ name }}
              <button @click="deleteMediaFromDocument(post, name)">Supprimer</button>
            </li>
          </ul>
        </div>

        <!-- Ajouter des médias -->
        <div>
          <input type="file" multiple @change="onFileChange" />
          <button @click="addMediaToDocument(post)">Ajouter des médias</button>
        </div>
      </li>
    </ul>

    <!-- Ajouter un document factice -->
    <button @click="addFakeDocument">Ajouter un document fake</button>
  </div>
</template>
