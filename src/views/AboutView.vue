<script lang="ts">
import { ref } from 'vue'
import PouchDB from 'pouchdb'

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
}

export default {
  data() {
    return {
      postsData: [] as Post[], // Liste des documents
      document: null as Post | null, // Document sélectionné pour modification
      storage: null as PouchDB.Database | null, // Base de données locale
      newContent: '', // Contenu modifié pour le formulaire
      fakeDocumentId: 1 // ID pour générer des documents factices
    }
  },

  mounted() {
    this.initDatabase()
    this.fetchData()
    this.watchChanges() // Écoute les changements en temps réel
  },

  methods: {
    // **1. Se connecter à une base de données distante**
    initDatabase() {
      const localDb = new PouchDB('local_database') // Base locale
      const remoteDb = new PouchDB('http://admin:admin@localhost:5984/database') // Base distante

      this.storage = localDb // Utiliser la base locale
      console.log('Base locale initialisée :', localDb.name)
      console.log('Base distante configurée :', remoteDb.name)

      this.syncDatabases(localDb, remoteDb) // Synchronisation automatique
    },

    // **2. Récupérer et afficher tous les documents d’une collection**
    fetchData() {
      const db = this.storage
      if (db) {
        db.allDocs({ include_docs: true, attachments: true })
          .then((result) => {
            this.postsData = result.rows.map((row) => row.doc) // Charger uniquement les documents
            console.log('Documents récupérés :', this.postsData)
          })
          .catch((error) => {
            console.error('Erreur lors de la récupération des données :', error)
          })
      }
    },

    // **3. Ajouter un document (générer et ajouter un document fake)**
    generateFakeDocument() {
      const newId = this.fakeDocumentId++
      return {
        _id: newId.toString(),
        doc: {
          post_name: `Post ${newId}`,
          post_content: 'Ceci est un contenu généré pour un document fake.',
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

    // **4. Modifier un document**
    selectDocument(post: Post) {
      this.document = post
      this.newContent = post.doc.post_content // Charger le contenu actuel dans le formulaire
    },

    updateDocument() {
      const db = this.storage
      if (this.document && db) {
        const updatedDoc = { ...this.document } // Créer une copie
        updatedDoc.doc.post_content = this.newContent // Modifier le contenu

        db.put(updatedDoc)
          .then((response) => {
            console.log('Document mis à jour avec succès :', response)
            const index = this.postsData.findIndex((doc) => doc._id === updatedDoc._id)
            if (index !== -1) {
              this.postsData[index] = { ...updatedDoc, _rev: response.rev } // Mettre à jour localement
            }
            this.document = null
            this.newContent = ''
          })
          .catch((error) => {
            console.error('Erreur lors de la mise à jour du document :', error)
          })
      }
    },

    // **5. Supprimer un document**
    deleteDocument(post: Post) {
      const db = this.storage
      if (db && post._id) {
        db.get(post._id)
          .then((doc) => db.remove(doc._id, doc._rev))
          .then(() => {
            console.log('Document supprimé avec succès.')
            this.postsData = this.postsData.filter((doc) => doc._id !== post._id) // Supprimer localement
          })
          .catch((error) => {
            console.error('Erreur lors de la suppression du document :', error)
          })
      }
    },

    // **6. Synchronisation des bases de données locale et distante**
    syncDatabases(localDb: PouchDB.Database, remoteDb: PouchDB.Database) {
      PouchDB.sync(localDb, remoteDb, {
        live: true,
        retry: true
      })
        .on('change', (info) => {
          console.log('Changements détectés lors de la synchronisation :', info)
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

    // **7. Détecter les mises à jour**
    watchChanges() {
      const db = this.storage
      if (db) {
        db.changes({ since: 'now', live: true })
          .on('change', (change) => {
            console.log('Changement détecté :', change)
            this.fetchData() // Rafraîchir les données locales
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
    <ul>
      <li v-for="post in postsData" :key="post._id">
        <span @click="selectDocument(post)">{{ post.doc.post_name }}</span>
        <button @click="deleteDocument(post)">Supprimer</button>
      </li>
    </ul>

    <div v-if="document">
      <h2>Modifier le contenu du post</h2>
      <textarea v-model="newContent" rows="4" cols="50"></textarea>
      <button @click="updateDocument">Mettre à jour le document</button>
    </div>

    <button @click="addFakeDocument">Ajouter un document fake</button>
  </div>
</template>
