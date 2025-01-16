<template>
  <div>
    <h1>Gestion des Documents</h1>
    <button @click="handleAddDocument">Ajouter un Document</button>
    <button @click="populateDatabase">Générer des documents de test</button>
    <input v-model="searchQuery" placeholder="Rechercher par titre..." @input="handleSearch" />
    <ul>
      <li v-for="doc in documents" :key="doc._id">
        <h3>{{ doc.title }}</h3>
        <p>{{ doc.content }}</p>
        <button @click="handleUpdateDocument(doc._id)">Modifier</button>
        <button @click="handleDeleteDocument(doc._id)">Supprimer</button>
        <input type="file" multiple @change="handleMultipleFileUpload($event, doc._id)" />
        <div v-if="doc._attachments">
          <h4>Médias associés :</h4>
          <ul>
            <li v-for="(data, filename) in doc._attachments" :key="filename">
              {{ filename }}
              <button @click="handleDeleteAttachment(doc._id, filename)">Supprimer</button>
            </li>
          </ul>
        </div>
      </li>
    </ul>
  </div>
</template>

<script lang="ts">
import { defineComponent, ref, onMounted } from 'vue'
import PouchDB from 'pouchdb'
import PouchDBFind from 'pouchdb-find'

PouchDB.plugin(PouchDBFind)

// Configuration de la base de données
const remoteDB = new PouchDB('http://admin:admin@localhost:5984/database')
const localDB = new PouchDB('local-database')

// Synchronisation entre bases de données
const syncDB = () => {
  localDB
    .sync(remoteDB, {
      live: true,
      retry: true
    })
    .on('change', (info) => {
      console.log('Changements détectés:', info)
    })
    .on('error', (err) => {
      console.error('Erreur de synchronisation:', err)
    })
}

syncDB()

export default defineComponent({
  setup() {
    const documents = ref<any[]>([])
    const searchQuery = ref('')

    const fetchAllDocuments = async () => {
      try {
        const result = await localDB.allDocs({ include_docs: true })
        documents.value = result.rows.map((row) => row.doc)
      } catch (err) {
        console.error('Erreur lors de la récupération des documents:', err)
      }
    }

    const handleAddDocument = async () => {
      const newDoc = {
        _id: new Date().toISOString(),
        title: `Post ${documents.value.length + 1}`,
        content: 'Ceci est un contenu de test.',
        createdAt: new Date().toISOString()
      }
      try {
        await localDB.put(newDoc)
        await localDB.replicate.to(remoteDB)
        await fetchAllDocuments()
      } catch (err) {
        console.error("Erreur lors de l'ajout du document:", err)
      }
    }

    const populateDatabase = async () => {
      const bulkDocs = Array.from({ length: 20 }, (_, i) => ({
        _id: `post_${i + 1}`,
        title: `Post ${i + 1}`,
        content: `Ceci est le contenu de Post ${i + 1}.`,
        createdAt: new Date().toISOString()
      }))

      try {
        await localDB.bulkDocs(bulkDocs)
        await localDB.replicate.to(remoteDB)
        await fetchAllDocuments()
      } catch (err) {
        console.error('Erreur lors de la génération des documents:', err)
      }
    }

    const handleUpdateDocument = async (docId: string) => {
      try {
        const doc = await localDB.get(docId)
        const updatedDoc = { ...doc, title: doc.title + ' (modifié)' }
        await localDB.put(updatedDoc)
        await localDB.replicate.to(remoteDB)
        await fetchAllDocuments()
      } catch (err) {
        console.error('Erreur lors de la mise à jour du document:', err)
      }
    }

    const handleDeleteDocument = async (docId: string) => {
      try {
        const doc = await localDB.get(docId)
        await localDB.remove(doc)
        await localDB.replicate.to(remoteDB)
        await fetchAllDocuments()
      } catch (err) {
        console.error('Erreur lors de la suppression du document:', err)
      }
    }

    const handleSearch = async () => {
      try {
        const result = await localDB.find({
          selector: { title: { $regex: new RegExp(searchQuery.value, 'i') } }
        })
        documents.value = result.docs
      } catch (err) {
        console.error('Erreur lors de la recherche:', err)
      }
    }

    const handleMultipleFileUpload = async (event: Event, docId: string) => {
      const files = (event.target as HTMLInputElement).files
      if (!files) return

      try {
        const doc = await localDB.get(docId)

        for (const file of files) {
          await localDB.putAttachment(docId, file.name, doc._rev, file, file.type)
          const updatedDoc = await localDB.get(docId) // Refresh rev for each file
          doc._rev = updatedDoc._rev
        }

        await localDB.replicate.to(remoteDB)
        await fetchAllDocuments()
      } catch (err) {
        console.error("Erreur lors de l'ajout des fichiers:", err)
      }
    }

    const handleDeleteAttachment = async (docId: string, filename: string) => {
      try {
        const doc = await localDB.get(docId)
        await localDB.removeAttachment(docId, filename, doc._rev)
        await localDB.replicate.to(remoteDB)
        await fetchAllDocuments()
      } catch (err) {
        console.error('Erreur lors de la suppression du fichier:', err)
      }
    }

    onMounted(() => {
      fetchAllDocuments()
    })

    return {
      documents,
      searchQuery,
      handleAddDocument,
      populateDatabase,
      handleUpdateDocument,
      handleDeleteDocument,
      handleSearch,
      handleMultipleFileUpload,
      handleDeleteAttachment
    }
  }
})
</script>

<style scoped>
button {
  margin: 5px;
  padding: 8px 12px;
  border: none;
  background-color: #007bff;
  color: white;
  cursor: pointer;
}

button:hover {
  background-color: #0056b3;
}

ul {
  list-style-type: none;
  padding: 0;
}

li {
  margin-bottom: 15px;
  padding: 10px;
  border: 1px solid #ddd;
  border-radius: 5px;
}
</style>
