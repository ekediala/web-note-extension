<template>
  <div>
    <form class="form">
      <div>
        <h2 id="title">{{ note.title ? note.title : 'New Note' }}</h2>
        <a class="btn-link" href="#" @click.prevent="continueNote" v-if="newUrl">Continue {{ lastNoteTitle }}?</a>
      </div>
      <div class="form-group">
        <input autocomplete required @keyup="autoSave" id="noteTitle" type="text" placeholder="Enter title" v-model="note.title" />
      </div>
      <div class="form-group">
        <textarea autocomplete required id="note" @keyup="autoSave" placeholder="Enter note" v-model="note.note" cols="50" rows="15"></textarea>
      </div>
      <button id="btn-download" v-show="downloadable" type="submit" class="btn" @click="download">Download note ⏬</button> &nbsp;
      <button id="btn-clear" v-show="clearable" class="btn" @click.prevent="clear">Clear &#10006;</button> &nbsp;
      <button v-show="showUndo" id="btn-undo" class="btn" @click.prevent="undo">Undo &#9851;</button> &nbsp;
      <!-- <button type="submit" id="btn-new" class="btn" @click="archive">Archive &#10004;</button> -->
      <br />
      <small v-show="typing && clearable" style="color: darkcyan">{{ note.showSaved ? 'Saving...' : 'Saved' }}</small>
    </form>
  </div>
</template>

<script>
import jsPDF from 'jspdf';
export default {
  name: 'App',
  data() {
    return {
      url: '',
      lastUrl: '',
      newUrl: false,
      typing: false,
      lastNoteTitle: '',
      note: {
        title: '',
        note: '',
        showSaved: false,
        position: '',
      },
      backUp: {
        title: '',
        note: '',
      },
    };
  },

  computed: {
    clearable() {
      return this.note.title || this.note.note;
    },

    downloadable() {
      return this.note.title && this.note.note;
    },

    showUndo() {
      return this.backUp.title || this.backUp.note;
    },
  },

  methods: {
    autoSave() {
      this.backUp.title = '';
      this.backUp.note = '';
      localStorage.removeItem('clear');
      this.newUrl = false;
      this.save();
    },

    archive() {
      this.note.showSaved = false;
      let url = this.url;
      let notes = JSON.parse(localStorage.getItem(url));
      if (notes && notes.length > 0) {
        const currentNote = {
          title: '',
          note: '',
        };
        notes.push(currentNote);
        localStorage.setItem(url, JSON.stringify(notes));
        this.save();
        this.download();
        this.clear();
        this.note.showSaved = true;
      } else {
        alert('Empty note cannot be archived!');
        return false;
      }
    },

    clear() {
      if (this.note.title || this.note.note) {
        this.$nextTick().then(() => {
          let url = this.url;
          localStorage.setItem('clear', 'true');
          let notes = JSON.parse(localStorage.getItem(url));
          let note = notes[notes.length - 1];
          this.backUp.title = note.title;
          this.backUp.note = note.note;
          note.title = '';
          note.note = '';
          localStorage.setItem(url, JSON.stringify(notes));
          this.note.title = '';
          this.note.note = '';
        });
      }
    },

    continueNote() {
      this.$nextTick().then(() => {
        let url = localStorage.getItem('lastUrl');
        let notes = JSON.parse(localStorage.getItem(url));
        let currNote = notes[notes.length - 1];
        this.backUp.title = this.note.title;
        this.backUp.note = this.note.note;
        this.note.title = currNote.title;
        this.note.note = currNote.note;
        this.save();
        this.newUrl = false;
        setTimeout(() => {
          let position = document.getElementById('note').scrollTop;
          let note = document.getElementById('note');
          note.scrollTo(0, position);
          console.log('Ran');
        }, 500);
      });
    },

    download() {
      if (this.note.title && this.note.note) {
        let doc = new jsPDF();
        let note = this.note.note;
        let title = doc.splitTextToSize(this.note.title.toUpperCase(), 200);
        let firstPage = doc.splitTextToSize(note.slice(0, 4000), 230);
        let filename = this.note.title.toLowerCase().replace(/\s/g, '_');
        doc.setFontSize(18);
        doc.setFont('sans-serif', 'bold');
        doc.text(title, 10, 20, { align: 'left' });
        doc.setFont('sans-serif', 'normal');
        doc.setLineHeightFactor(1.2);
        doc.setFontSize(14);
        doc.text(firstPage, 10, 35, { align: 'left' });
        let unit = 4000;
        let cursorPosition = 4000;
        let pageNo = 1;
        while (note.length > unit * pageNo) {
          pageNo++;
          doc.addPage();
          doc.text(doc.splitTextToSize(note.slice(cursorPosition, unit * pageNo), 190), 5, 20, { align: 'left' });
          cursorPosition = unit * pageNo;
        }
        doc.save(`${filename}.pdf`);
      } else {
        alert('Note title or body missing.');
      }
    },

    isEmpty() {
      let url = localStorage.getItem('lastUrl');
      let notes = JSON.parse(localStorage.getItem(url));
      let currNote = notes[notes.length - 1];
      if (currNote.title || currNote.note) {
        return false;
      } else {
        return true;
      }
    },

    // not in use
    navigateTo() {
      chrome.tabs.create({ url: 'https://webnotes.web.app' });
    },

    save() {
      this.typing = true;
      let position = document.getElementById('note').scrollTop;
      let title = this.note.title;
      let note = this.note.note;
      localStorage.setItem('lastUrl', this.url);
      this.lastUrl = this.url;
      let url = this.url;
      let notes = JSON.parse(localStorage.getItem(url));
      let currentNote = notes[notes.length - 1];
      currentNote.title = title;
      currentNote.note = note;
      currentNote.position = position;
      localStorage.setItem(url, JSON.stringify(notes));
      this.note.showSaved = true;
      setTimeout(() => {
        this.note.showSaved = false;
      }, 1000);
    },

    undo() {
      this.$nextTick().then(() => {
        this.note.title = this.backUp.title;
        this.note.note = this.backUp.note;
        this.backUp.title = '';
        this.backUp.note = '';
        this.save();
      });
    },

    // upload() is not yet in use
    upload() {
      let url = this.url;
      let notesCollection = JSON.parse(localStorage.getItem('EkeDialaWebNotesDocuments'));
      let found = false;
      for (const collection in notesCollection) {
        if (collection.url == url) {
          collection.notes = JSON.parse(localStorage.getItem(url));
          found = true;
        }
      }
      if (!found) {
        // no url note collection, create one
        let noteObject = {
          url,
          notes: [
            {
              title: this.note.title,
              note: this.note.note,
            },
          ],
        };
        notesCollection.push(noteObject);
      }
      localStorage.setItem('EkeDialaWebNotesDocuments', JSON.stringify(notesCollection));
    },
  },

  created() {
    //check if there is a note running already on the current tab
    //localStorage.removeItem('EkeDialaWebNotesDocuments');
    chrome.tabs.query({ currentWindow: true, active: true }, tabs => {
      let url = tabs[0].url;
      let searchString = '/';
      let regex = new RegExp(searchString, 'g');
      url = url.replace(regex, '-');
      this.url = url;
      this.lastUrl = localStorage.getItem('lastUrl');
      if (this.lastUrl) {
        let status = this.isEmpty();
        let url = localStorage.getItem('lastUrl');
        let notes = JSON.parse(localStorage.getItem(url));
        let currNote = notes[notes.length - 1];
        this.lastNoteTitle = currNote.title ? `"${currNote.title}"` : 'last edited note';
        this.newUrl = this.lastUrl != this.url && !status;
      }
      let notes = JSON.parse(localStorage.getItem(url));
      let currentNote = {};
      if (notes) {
        currentNote = notes[notes.length - 1];
        this.note.title = currentNote.title;
        this.note.note = currentNote.note;
        this.note.position = currentNote.position;
      } else {
        // extract url and save it as the key to a
        // note array in local storage
        localStorage.setItem(
          url,
          JSON.stringify([
            {
              title: '',
              note: '',
              position: 0,
            },
          ])
        );
        notes = JSON.parse(localStorage.getItem(url));
        currentNote = notes[0];
        this.note.title = currentNote.title;
        this.note.note = currentNote.note;
        this.note.position = currentNote.position;
      }
    });
  },

  mounted() {
    let note = document.getElementById('note');
    setTimeout(() => {
      let height = this.note.position ? this.note.position : 0;
      note.scrollTo(0, height);
    }, 1);

    note.onscroll = () => {
      let position = note.scrollTop;
      let url = this.url;
      let notes = JSON.parse(localStorage.getItem(url));
      let currentNote = notes[notes.length - 1];
      currentNote.position = position;
      localStorage.setItem(url, JSON.stringify(notes));
    };
  },
};
</script>

<style lang="scss" scoped>
* {
  font-family: 'Lucida Sans', 'Lucida Sans Regular', 'Lucida Grande', 'Lucida Sans Unicode', Geneva, Verdana, sans-serif;
}
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
    &:focus {
      outline: none;
    }
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

.btn-link,
.btn-link:active,
.btn-link:visited,
.btn-link:link {
  text-decoration: none;
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

#btn-undo {
  background-color: orangered;
}

.form {
  text-align: center;
}
</style>
