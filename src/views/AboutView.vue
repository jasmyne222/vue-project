<script lang="ts">
import { ref } from 'vue'
import PouchDB from 'pouchdb'

declare interface Post {
  _id: string
  _rev?: string // Le _rev est important pour les mises à jour et suppressions
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
      total: 0,
      postsData: [] as Post[],
      document: null as Post | null,
      storage: null as PouchDB.Database | null,
      newContent: '', // Ajouté pour le champ de texte lors de la modification
      fakeDocumentId: 1 // Compteur pour générer les IDs
    }
  },

  mounted() {
    this.initDatabase()
    this.fetchData()
  },

  methods: {
    // Méthode pour ajouter ou mettre à jour un document dans la base de données
    putDocument(document: Post) {
      const db = ref(this.storage).value
      if (db) {
        db.put(document)
          .then(() => {
            console.log('Add ok')
          })
          .catch((error) => {
            console.log('Add ko', error)
          })
      }
    },

    postDocument(document: Post) {
      const db = ref(this.storage).value
      if (db) {
        db.post(document)
          .then(() => {
            console.log('Add ok')
          })
          .catch((error) => {
            console.log('Add ko', error)
          })
      }
    },

    // Récupérer tous les documents de la base de données
    fetchData() {
      const storage = ref(this.storage)
      const self = this
      if (storage.value) {
        storage.value
          .allDocs({
            include_docs: true,
            attachments: true
          })
          .then(function (result: any) {
            console.log('fetchData success', result)
            self.postsData = result.rows
          })
          .catch(function (error: any) {
            console.log('fetchData error', error)
          })
      }
    },

    // Initialisation de la base de données
    initDatabase() {
      const db = new PouchDB('http://admin:admin@localhost:5984/database')
      if (db) {
        console.log('Connected to collection ', db.name)
      } else {
        console.warn('Something went wrong')
      }
      this.storage = db
    },

    // Générer un document fake avec un ID numérique croissant
    generateFakeDocument() {
      const newId = this.fakeDocumentId++ // Incrémente l'ID à chaque génération

      return {
        post_name: 'Post',
        post_content: 'Ceci est un contenu généré pour un document fake.',
        attributes: {
          creation_date: new Date().toISOString()
        }
      } as Post // Cast en type Post
    },

    // Ajouter un document fake dans la base de données
    addFakeDocument() {
      const fakeDoc = this.generateFakeDocument() // Générer un document fake
      this.postDocument(fakeDoc) // Utiliser la méthode putDocument pour l'ajouter à la base
    },

    // Sélectionner un document pour modification
    selectDocument(post: Post) {
      this.document = post
      this.newContent = post.doc.post_content // Charger le contenu du document dans le champ de texte
    },

    // Mettre à jour un document avec le nouveau contenu
    updateDocument() {
      if (this.document) {
        const updatedDoc = { ...this.document } // Créer une copie du document pour ne pas modifier l'original directement
        updatedDoc.doc.post_content = this.newContent // Modifier le contenu

        // Sauvegarder les modifications dans la base de données
        this.putDocument(updatedDoc)
        this.fetchData() // Rafraîchir les documents après la mise à jour
        console.log('Document mis à jour avec succès!')
      }
    },

    // Méthode pour supprimer un document
    deleteDocument(post: Post) {
      console.log(post.doc._id)
      console.log(post.id)
      const db = ref(this.storage).value
      if (db && post.id) {
        db.get(post.id) // Récupérer le document par son ID
          .then((doc) => {
            if (!doc._rev) {
              throw new Error('Version (_rev) manquante pour la suppression.')
            }
            console.log(
              `Tentative de suppression du document avec _id: ${doc._id} et _rev: ${doc._rev}`
            )
            return db.remove(doc._id, doc._rev) // Supprimer le document en utilisant _rev
          })
          .then(() => {
            console.log('Document supprimé avec succès!')
            this.fetchData() // Rafraîchir les documents après la suppression
          })
          .catch((error) => {
            console.error('Erreur lors de la suppression du document:', error)
          })
      } else {
        console.error('ID du document invalide ou base de données non initialisée.')
      }
    }
  }
}
</script>

<template>
  <div>
    <h1>Nombre de post: {{ postsData.length }}</h1>
    <ul>
      <li v-for="post in postsData" :key="post._id">
        <span @click="selectDocument(post)">{{ post.doc.post_name }}</span>
        <button @click="deleteDocument(post)">Supprimer</button>
        <!-- Bouton de suppression -->
      </li>
    </ul>

    <!-- Zone de texte pour modifier le contenu du document -->
    <div v-if="document">
      <h2>Modifier le contenu du post</h2>
      <textarea v-model="newContent" rows="4" cols="50"></textarea>
      <button @click="updateDocument">Mettre à jour le document</button>
    </div>

    <!-- Bouton pour ajouter un document fake -->
    <button @click="addFakeDocument">Ajouter un document fake</button>
  </div>
</template>
