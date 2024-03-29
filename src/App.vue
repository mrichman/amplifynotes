<template>
  <amplify-authenticator>
    <div v-if="authState !== 'signedin'">You are signed out.</div>
    <div id="app">
      <nav class="navbar navbar-expand-md navbar-dark fixed-top bg-dark">
        <a class="navbar-brand" href="#">Serverless Notes</a>
        <button
          class="navbar-toggler"
          type="button"
          data-toggle="collapse"
          data-target="#navbarCollapse"
          aria-controls="navbarCollapse"
          aria-expanded="false"
          aria-label="Toggle navigation"
        >
          <span class="navbar-toggler-icon"></span>
        </button>
        <div class="collapse navbar-collapse" id="navbarCollapse">
          <ul class="navbar-nav mr-auto">
            <li class="nav-item"></li>
          </ul>
          <ul class="nav navbar-nav navbar-right mr-4 text-light">
            <div v-if="authState === 'signedin' && user">
              Hello, {{ user.username }}
            </div>
          </ul>
          <form class="form-inline mt-2 mt-md-0">
            <amplify-sign-out></amplify-sign-out>
          </form>
        </div>
      </nav>

      <main role="main">
        <div class="container bg-light p-3 mt-4 mb-1">
          <div class="mb-3">
            <label for="noteTitleInput" class="form-label">Note title</label>
            <input
              v-model="title"
              type="text"
              class="form-control"
              id="noteTitleInput"
              placeholder="Note title"
            />
          </div>
          <div class="mb-3">
            <label for="noteBodyInput" class="form-label">Note body</label>
            <textarea
              class="form-control"
              v-model="body"
              id="noteBodyInput"
              placeholder="Note body"
              rows="3"
            />
          </div>
          <div>
            <button v-on:click="createNote" class="btn btn-primary mb-3">
              Create note
            </button>
          </div>
        </div>

        <div class="container bg-light p-3 rounded">
          <h2>Notes</h2>
          <div class="list-group">
            <div v-for="item in notes" :key="item.id">
              <div
                href="#"
                class="list-group-item flex-column align-items-start"
              >
                <div class="d-flex w-100 justify-content-between">
                  <h5 class="mb-1">{{ item.title }}</h5>
                  <button
                    @click="deleteNote(item.id)"
                    class="btn btn-outline-danger btn-sm"
                  >
                    X
                  </button>
                </div>
                <p class="mb-1">
                  {{ item.body }}
                </p>
                <small>{{ item.updatedAt }}</small>
              </div>
            </div>
          </div>
        </div>
      </main>
    </div>
  </amplify-authenticator>
</template>

<script>
import { API } from "aws-amplify";
import { createNote, deleteNote } from "./graphql/mutations";
import { listNotes } from "./graphql/queries";
import { onCreateNote } from "./graphql/subscriptions";
import { onAuthUIStateChange } from "@aws-amplify/ui-components";

export default {
  name: "app",
  created() {
    this.unsubscribeAuth = onAuthUIStateChange((authState, authData) => {
      console.log("AuthUIStateChange");
      console.log(authData);
      this.authState = authState;
      this.user = authData;
      this.subscribe(authData.username);
    });
    this.getNotes();
  },
  data() {
    return {
      title: "",
      body: "",
      notes: [],
      user: undefined,
      authState: undefined,
    };
  },
  beforeUnmount() {
    this.unsubscribeAuth();
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
    async deleteNote(id) {
      const note = {};
      note.id = id;
      console.log(note);
      await API.graphql({
        query: deleteNote,
        variables: { input: note },
      });
      this.getNotes();
    },
    async getNotes() {
      const notes = await API.graphql({
        query: listNotes,
      });
      // sort descending by date
      this.notes = notes.data.listNotes.items.sort((a, b) =>
        a.updatedAt < b.updatedAt ? 1 : -1
      );
    },
    subscribe(username) {
      console.log("Subscribing to onCreateNote: " + username);
      API.graphql({
        query: onCreateNote,
        variables: { owner: username },
      }).subscribe({
        next: (eventData) => {
          console.log(eventData);
          this.getNotes();
        },
        error: (error) => {
          console.warn(error);
        },
      });
    },
  },
};
</script>

<style>
body {
  font-family: "Amazon Ember", -apple-system, BlinkMacSystemFont, "Segoe UI",
    Roboto, Oxygen, Ubuntu, Cantarell, "Open Sans", "Helvetica Neue", sans-serif;
  padding-top: 4rem;
}
</style>