<script lang="ts">
import { ref } from 'vue'
import PouchDB from 'pouchdb'
import PouchDBFind from 'pouchdb-find'

PouchDB.plugin(PouchDBFind) // Activer la fonctionnalité d'indexation

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
      fakeDocumentId: 1, // ID pour générer des documents factices
      searchQuery: '' // Champ de recherche
    }
  },

  mounted() {
    this.initDatabase()
    this.createIndex() // Créer un index au démarrage
    this.fetchData()
    this.watchChanges() // Écoute les changements en temps réel
  },

  methods: {
    // **1. Initialisation de la base de données**
    initDatabase() {
      const localDb = new PouchDB('local_database') // Base locale
      const remoteDb = new PouchDB('http://admin:admin@localhost:5984/database') // Base distante

      this.storage = localDb // Utiliser la base locale
      console.log('Base locale initialisée :', localDb.name)
      console.log('Base distante configurée :', remoteDb.name)

      this.syncDatabases(localDb, remoteDb) // Synchronisation automatique
    },

    // **2. Créer un index pour l'attribut `doc.post_name`**
    createIndex() {
      const db = this.storage
      if (db) {
        db.createIndex({
          index: { fields: ['doc.post_name'] } // Index sur `post_name`
        })
          .then(() => {
            console.log('Index créé avec succès sur `doc.post_name`.')
          })
          .catch((error) => {
            console.error('Erreur lors de la création de l’index :', error)
          })
      }
    },

    // **3. Recherche des documents par l'attribut indexé**
    searchByAttribute() {
      const db = this.storage
      if (db) {
        db.find({
          selector: {
            'doc.post_name': { $regex: new RegExp(this.searchQuery, 'i') } // Recherche insensible à la casse
          }
        })
          .then((result) => {
            this.postsData = result.docs // Mettre à jour les données affichées
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
            this.postsData = result.rows.map((row) => row.doc) // Charger uniquement les documents
            console.log('Documents récupérés :', this.postsData)
          })
          .catch((error) => {
            console.error('Erreur lors de la récupération des données :', error)
          })
      }
    },

    // **5. Factory pour générer de nombreux documents**
    populateDatabase() {
      const db = this.storage
      if (db) {
        const docs = Array.from({ length: 100 }, (_, index) => ({
          _id: `fake_doc_${index + 1}`,
          doc: {
            post_name: `Post ${index + 1}`,
            post_content: `Contenu généré pour le Post ${index + 1}`,
            attributes: {
              creation_date: new Date().toISOString()
            }
          }
        })) // Générer 100 documents

        db.bulkDocs(docs)
          .then(() => {
            console.log('Base de données peuplée avec succès.')
            this.fetchData() // Rafraîchir les données après insertion
          })
          .catch((error) => {
            console.error('Erreur lors du peuplement de la base :', error)
          })
      }
    },

    // **6. Ajouter un document fake**
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

    // **7. Modifier un document**
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

    // **8. Supprimer un document**
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

    // **9. Synchronisation des bases de données locale et distante**
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

    // **10. Détecter les mises à jour**
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

    <!-- Champ de recherche -->
    <div>
      <input type="text" v-model="searchQuery" placeholder="Post 22" />
      <button @click="searchByAttribute">Rechercher</button>
    </div>

    <!-- Liste des documents -->
    <ul>
      <li v-for="post in postsData" :key="post._id">
        <span @click="selectDocument(post)">{{ post.doc.post_name }}</span>
        <button @click="deleteDocument(post)">Supprimer</button>
      </li>
    </ul>

    <!-- Formulaire pour modifier un document -->
    <div v-if="document">
      <h2>Modifier le contenu du post</h2>
      <textarea v-model="newContent" rows="4" cols="50"></textarea>
      <button @click="updateDocument">Mettre à jour le document</button>
    </div>

    <!-- Bouton pour ajouter un document fake -->
    <button @click="addFakeDocument">Ajouter un document fake</button>

    <!-- Bouton pour peupler la base -->
    <button @click="populateDatabase">Générer 100 documents</button>
  </div>
</template>
