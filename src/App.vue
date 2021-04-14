<template>
  <div id="app">
    <h1>Serverless Notes</h1>
    <input type="text" v-model="title" placeholder="Note title" />
    <input type="text" v-model="body" placeholder="Note body" />
    <button v-on:click="createNote">Create Note</button>
    <div v-for="item in notes" :key="item.id">
      <h3>{{ item.title }}</h3>
      <p>{{ item.body }}</p>
    </div>
  </div>
</template>

<script>
import { API } from "aws-amplify";
import { createNote } from "./graphql/mutations";
import { listNotes } from "./graphql/queries";
import { onCreateNote } from "./graphql/subscriptions";

export default {
  name: "app",
  async created() {
    this.getNotes();
    this.subscribe();
  },
  data() {
    return {
      title: "",
      body: "",
      notes: [],
    };
  },
  methods: {
    async createNote() {
      const { title, body } = this;
      if (!title || !body) return;
      const note = { title, body };
      await API.graphql({
        query: createNote,
        variables: { input: note },
      });
      this.title = "";
      this.body = "";
    },
    async getNotes() {
      const notes = await API.graphql({
        query: listNotes,
      });
      this.notes = notes.data.listNotes.items;
    },
    subscribe() {
      API.graphql({ query: onCreateNote }).subscribe({
        next: (eventData) => {
          let note = eventData.value.data.onCreateNote;
          if (this.notes.some((item) => item.title === note.title)) return; // remove duplications
          this.notes = [...this.notes, note];
        },
      });
    },
  },
};
</script>