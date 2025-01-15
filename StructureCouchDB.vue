<script lang="ts">
import PouchDB from "pouchdb";
export default {
  data() {
    return {
      dabase: null as PouchDB.Database | null,
    };
  },

  methods: {
    inc() {},

    initDatabase() {
      const db = new PouchDB("http://127.0.0.1:5984/database");
      if (db) {
        console.log("Connected to collection 'post'");
      } else {
        console.warn("Something went wrong");
      }
      this.storage = db;
    },

    fetchData() {
      const db = new PouchDB("http://127.0.0.1:5984/database"); // Remplacer par l'URL correcte
      db.allDocs({ include_docs: true })
        .then((result) => {
          this.datas = result.rows.map((row) => row.doc); // Mettre à jour `datas` avec les documents récupérés
          console.log(this.datas);
        })
        .catch((err) => {
          console.error("Erreur lors de la récupération des données", err);
        });
    },
  },

  mounted() {
    this.initDatabase();
  },
};
</script>

<template>
  <h1>InfraDon2</h1>
  <p>Counter: {{ datas }}</p>
  <button type="button" role="button" @click="inc">+1</button>
</template>
