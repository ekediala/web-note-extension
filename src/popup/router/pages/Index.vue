<template>
  <div>
    <form class="form">
      <div>
        <h2 id="title">{{ note.title ? note.title : 'New Note' }}</h2>
        <p>Visit <a href="" @click.prevent="navigateTo">webnotes</a> to view and manage your notes.</p>
      </div>
      <div class="form-group">
        <input autocomplete required @keyup="autoSave" id="noteTitle" type="text" placeholder="Enter title" v-model="note.title" />
      </div>
      <div class="form-group">
        <textarea autocomplete required id="note" @keyup="autoSave" placeholder="Enter note" v-model="note.note" cols="50" rows="15"></textarea>
      </div>
      <button id="btn-download" type="submit" class="btn" @click="download">Download note ⏬</button> &nbsp;
      <button id="btn-clear" class="btn" @click.prevent="clear">Clear &#10006;</button> &nbsp;
      <button type="submit" id="btn-new" class="btn" @click="newNote">Done &#10004;</button>
      <br />
      <small style="color: darkcyan">{{ note.showSaved ? 'Saving...' : 'Saved' }}</small>
      <div>
        <p>
          Clicking 'DONE' saves current document and clears the note for a new note entry else the new document overrites the old one. Notes belong to the page (url) they are
          running on and are saved on the browser.
        </p>
      </div>
    </form>
  </div>
</template>

<script>
import jsPDF from 'jspdf';
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
      chrome.tabs.query({ currentWindow: true, active: true }, tabs => {
        let url = tabs[0].url;
        let searchString = '/';
        let regex = new RegExp(searchString, 'g');
        url = url.replace(regex, '-');
        let title = this.note.title;
        let note = this.note.note;
        let notes = JSON.parse(localStorage.getItem(url));
        let currentNote = notes[notes.length - 1];
        currentNote.title = title;
        currentNote.note = note;
        localStorage.setItem(url, JSON.stringify(notes));
        let notesCollection = JSON.parse(localStorage.getItem('EkeDialaWebNotesDocuments'));
        const predicate = obj => (obj.url = url);
        let collection = notesCollection.find(predicate);
        collection.notes = JSON.parse(localStorage.getItem(url));
        localStorage.setItem('EkeDialaWebNotesDocuments', JSON.stringify(notesCollection));
        this.note.showSaved = true;
        setTimeout(() => {
          this.note.showSaved = false;
        }, 1000);
      });
    },

    download() {
      if (this.note.title && this.note.note) {
        let doc = new jsPDF();
        let note = this.note.note;
        let title = doc.splitTextToSize(this.note.title.toUpperCase(), 200);
        let firstPage = doc.splitTextToSize(note.slice(0, 4000), 250);
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
          doc.text(doc.splitTextToSize(note.slice(cursorPosition, unit * pageNo), 250), 10, 30, { align: 'left' });
          cursorPosition = unit * pageNo;
        }
        doc.save(`${filename}.pdf`);
      } else {
        alert('Document is empty.');
      }
    },

    navigateTo() {
      chrome.tabs.create({ url: 'https://www.webnotes.web.app' });
    },

    newNote() {
      this.download();
      chrome.tabs.query({ currentWindow: true, active: true }, tabs => {
        let url = tabs[0].url;
        let searchString = '/';
        let regex = new RegExp(searchString, 'g');
        url = url.replace(regex, '-');
        let notes = JSON.parse(localStorage.getItem(url));
        if (notes && notes.length > 0) {
          const currentNote = {
            title: '',
            note: '',
          };
          notes.push(currentNote);
          localStorage.setItem(url, JSON.stringify(notes));
          let notesCollection = JSON.parse(localStorage.getItem('EkeDialaWebNotesDocuments'));
          const predicate = obj => (obj.url = url);
          let collection = notesCollection.find(predicate);
          collection.notes = JSON.parse(localStorage.getItem(url));
          localStorage.setItem('EkeDialaWebNotesDocuments', JSON.stringify(notesCollection));
          this.clear();
        }
      });
    },

    clear() {
      this.$nextTick().then(() => {
        chrome.tabs.query({ currentWindow: true, active: true }, tabs => {
          let url = tabs[0].url;
          let searchString = '/';
          let regex = new RegExp(searchString, 'g');
          url = url.replace(regex, '-');
          let notes = JSON.parse(localStorage.getItem(url));
          let note = notes[notes.length - 1];
          note.title = '';
          note.note = '';
          localStorage.setItem(url, JSON.stringify(notes));
          this.note.title = '';
          this.note.note = '';
        });
      });
    },
  },

  created() {
    //check if there is a note running already on the current tab
    if (!localStorage.getItem('EkeDialaWebNotesDocuments')) {
      localStorage.setItem('EkeDialaWebNotesDocuments', JSON.stringify([]));
    }
    chrome.tabs.query({ currentWindow: true, active: true }, tabs => {
      let url = tabs[0].url;
      let searchString = '/';
      let regex = new RegExp(searchString, 'g');
      url = url.replace(regex, '-');
      let notes = JSON.parse(localStorage.getItem(url));
      let currentNote = {};
      if (notes) {
        currentNote = notes[notes.length - 1];
        this.note.title = currentNote.title;
        this.note.note = currentNote.note;
      } else {
        // extract url and save it as the key to a
        // note array in local storage
        localStorage.setItem(
          url,
          JSON.stringify([
            {
              title: '',
              note: '',
            },
          ])
        );
        notes = JSON.parse(localStorage.getItem(url));
        currentNote = notes[0];
        this.note.title = currentNote.title;
        this.note.note = currentNote.note;
        let noteObject = { url, notes };
        let notesCollection = JSON.parse(localStorage.getItem('EkeDialaWebNotesDocuments'));
        notesCollection.push(noteObject);
        localStorage.setItem('EkeDialaWebNotesDocuments', JSON.stringify(notesCollection));
      }
    });
    console.log(localStorage.getItem('EkeDialaWebNotesDocuments'));
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
