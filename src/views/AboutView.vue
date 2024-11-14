<script lang="ts">
import { ref } from 'vue'
import PouchDB from 'pouchdb'

declare interface Post {
  _id: string
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
      storage: null as PouchDB.Database | null
    }
  },

  mounted() {
    this.initDatabase()
    this.fetchData()
  },

  methods: {
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

    fetchData() {
      const storage = ref(this.storage)
      const self = this
      if (storage.value) {
        storage.value
          .allDocs({
            include_docs: true,
            attachments: true
          })
          .then(
            function (result: any) {
              console.log('fetchData success', result)
              self.postsData = result.rows
            }.bind(this)
          )
          .catch(function (error: any) {
            console.log('fetchData error', error)
          })
      }
    },

    initDatabase() {
      const db = new PouchDB('http://admin:jijijaja@localhost:5984/database')
      if (db) {
        console.log("Connected to collection ", db.name)
      } else {
        console.warn('Something went wrong')
      }
      this.storage = db
    },
    // Nouvelle fonction pour générer un document fake
    generateFakeDocument() {
    return {
      _id: new Date().toISOString(), // ID unique basé sur la date
      doc: {
        post_name: 'Titre de démonstration',
        post_content: 'Ceci est un contenu généré pour un document fake.',
        attributes: {
          creation_date: new Date().toISOString(),
        },
      },
    } as Post;  // Cast en type Post
  },

  // Méthode pour ajouter un document fake à la base de données
  addFakeDocument() {
    const fakeDoc = this.generateFakeDocument();  // Générer un document fake
    this.putDocument(fakeDoc);  // Utiliser la méthode putDocument pour l'ajouter à la base
  }

  }
}

</script>

<template>
  <h1>Nombre de post: {{ postsData.length }}</h1>
  <ul>
    <li v-for="post in postsData" :key="post._id">
        {{post.doc.post_name}}
        <p v-for="school in post.doc.school" >  
          {{ school.name  }} {{ school.type  }}
        </p>
    </li>
  </ul>

  <!-- Bouton pour ajouter un document fake -->
  <button @click="addFakeDocument">Ajouter un document fake</button>
</template>