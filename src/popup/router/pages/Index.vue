<template>
  <div>
    <form class="form">
      <div>
        <h2 id="title">{{ note.title ? note.title : 'New Note' }}</h2>
        <p>Visit <a @click="navigateTo">webnotes</a> to view and manage your notes.</p>
        <small style="color: darkcyan" v-show="showSaved">Saved...</small>
      </div>
      <div class="form-group">
        <input required id="noteTitle" type="text" placeholder="Enter title" v-model="note.title" />
      </div>
      <div class="form-group">
        <textarea required id="note" @keyup="autoSave" placeholder="Enter note" v-model="note.note" cols="50" rows="15"></textarea>
      </div>
      <button id="btn-download" type="submit" class="btn" @click="download">Download note ⏬</button> &nbsp;
      <button id="btn-clear" class="btn" @click="clear">Clear &#10006;</button> &nbsp; <button id="btn-new" class="btn" @click="newNote">Done &#10004;</button> &nbsp;
      <div>
        <p>
          Clicking 'DONE' saves current document and clears the note for a new note entry else the new document overrites the old one. Notes belong to the page (url) they are
          running on.
        </p>
      </div>
    </form>
  </div>
</template>

<script>
import pdfMake from 'pdfmake/build/pdfmake';
import pdfFonts from 'pdfmake/build/vfs_fonts';
pdfMake.vfs = pdfFonts.pdfMake.vfs;
export default {
  data() {
    return {
      note: {
        title: '',
        note: '',
        showSaved: false,
      },
    };
  },

  methods: {
    autoSave() {
      let title = this.note.title;
      let note = this.note.note;
      let notes = JSON.parse(localStorage.getItem(this.getUrl()));
      let currentNote = notes[notes.length - 1];
      currentNote.title = title;
      currentNote.note = note;
      localStorage.setItem(this.getUrl(), JSON.stringify(notes));
      this.note.showSaved = true;
      setTimeout(() => {
        this.note.showSaved = false;
      }, 1000);
    },

    download(note = this.note) {
      let docDef = {
        header: note.title,
        content: note.note,
        footer: function(currentPage, pageCount) {
          return {
            columns: [currentPage.toString() + ' of ' + pageCount, { text: '&#9825; from Eke', alignment: 'right' }],
          };
        },
      };
      const win = window.open('', '_blank');
      pdfMake.createPdf(docDefinition).open({}, win);
    },

    navigateTo() {
      chrome.tabs.create({ url: 'https://www.webnotes.web.app' });
    },

    newNote() {
      let notes = JSON.parse(localStorage.getItem(this.getUrl()));
      if (notes && notes.length > 0) {
        const currentNote = {
          title: '',
          note: '',
        };
        notes.push(currentNote);
        localStorage.setItem(this.getUrl(), notes);
        this.clear();
      }
    },

    clear() {
      this.$nextTick().then(() => {
        this.note.title = '';
        this.note.note = '';
      });
    },

    getUrl() {
      let url = '';
      chrome.tabs.query({ currentWindow: true, active: true }, tabs => {
        url = tabs[0].url;
      });
      return url;
    },
  },

  created() {
    //check if there is a note running already on the current tab
    const key = this.getUrl();
    let notes = JSON.parse(localStorage.getItem(key));
    let currentNote = {};
    if (notes) {
      currentNote = notes[notes.length - 1];
      this.note.title = currentNote.title;
      this.note.note = currentNote.note;
    } else {
      // extract url and save it as the key to a
      // note array in local storage
      localStorage.setItem(
        key,
        JSON.stringify([
          {
            title: '',
            note: '',
          },
        ])
      );
      notes = JSON.parse(localStorage.getItem(key));
      currentNote = notes[0];
      this.note.title = currentNote.title;
      this.note.note = currentNote.note;
    }
  },
  beforeDestroy() {
    let url = '';
    chrome.tabs.query({ currentWindow: true, active: true }, tabs => {
      url = tabs[0].url;
    });
    localStorage.removeItem(url);
  },
};
</script>

<style lang="scss" scoped>
.form-group {
  input {
    margin-bottom: 10px;
    width: 100%;
    border-radius: 2px;
    border: none;
    border-bottom: 3px solid darkcyan;
    font-size: 16px;
  }

  input:focus {
    outline: none;
  }

  #note {
    border: none;
    border-bottom: 3px solid darkcyan;
    border-radius: 2px;
    font-size: 16px;
  }
}

.btn:focus {
  outline: none;
}

#title {
  font-size: 30px;
  font-weight: 400;
  text-transform: uppercase;
}

.btn:active {
  transform: scale(1.05);
  transition: all;
}

.btn,
.btn:link,
.btn:visited {
  // border-radius: 70px;
  text-transform: uppercase;
  text-decoration: none;
  padding: 8px 16px;
  display: inline-block;
  border-radius: 80px;
  position: relative;
  margin-top: 20px;
  border: none;
  color: white;
}

#btn-clear {
  background-color: crimson;
}

#btn-download {
  background-color: rgb(0, 81, 255);
}

#btn-new {
  background-color: orange;
}

.form {
  text-align: center;
}
</style>
