<template>
  <div id="wrapper" style="height: 100%;padding: 0" v-loading="loading">
    <el-container style="height: 100%">
      <el-header height="40px" style="border-bottom: 1px solid #eee;-webkit-app-region: drag">
        <div style="padding: 10px 0; font-size: 20px;font-weight: bold;position: relative">
          <div style="position: absolute;display: inline-block;left: 50%;transform: translate(-50%)">My Book</div>
        </div>
      </el-header>
      <el-container>
        <el-aside width="50px" style="background-color: #eee;">
          <div style="width: 30px; margin: auto; padding: 20px 0;">
            <el-row>
            <el-tooltip class="item" effect="dark" content="添加本地图书" placement="right">
            <el-button icon="el-icon-plus" circle size="mini" @click="openFileDialog"></el-button>
            </el-tooltip>
            </el-row>
          </div>
        </el-aside>
        <el-container>
          <el-header style="background: rgb(247,247,247);">
            <el-form :inline="true" style="margin-top: 10px"><el-form-item label="Type">
              <el-select v-model="type" placeholder="type" style="" size="mini" filterable>
                <el-option
                    v-for="item in types"
                    :key="item.value"
                    :label="item.value"
                    :value="item.value">
                </el-option>
              </el-select>
              <el-form-item label="Name">
              <el-input size="mini" v-model="name" clearable></el-input>
              </el-form-item>
              <el-form-item><el-button icon="el-icon-search" size="mini" @click="search"></el-button></el-form-item>
            </el-form-item></el-form>
          </el-header>
          <el-container style="height: 100%">
            <el-aside width="300px" style="border-right: 1px solid #eee">
              <div v-if="books && books.length" style="font-size: 12px;">
                <el-row v-for="b in books" :class="{'book-list': !bookSelected || bookSelected.path != b.path,
                'book-selected': bookSelected && bookSelected.path == b.path}"
                        @click.native="()=>selectBook(b)"
                >{{b.name}}</el-row>
              </div>
              <el-empty v-if="!books || !books.length" description="当前没有本地图书"></el-empty>
            </el-aside>
            <el-main style="position: relative">
              <el-card v-if="bookSelected">
                <div style="position: relative; min-width: 600px;min-height: 550px;">
                  <div style="margin-top: 10px; width: 300px; height: 400px; display: inline-block; position: relative" >
                    <canvas ref="pdfIcon" width="300" height="400" style="padding: 2px;border: 5px ridge #eee;position: absolute;left: 50%; top: 50%; transform: translate(-50%, -50%)"/>
                  </div>
                  <div style="position: absolute; left: 400px; display: inline-block">
                    <el-form style="margin-top: 10px" label-position="left">
                      <el-form-item label="Name">
                        <el-input v-if="editName" v-model="bookSelected.name" size="mini" style="width: 220px">
                          <el-button slot="append" icon="el-icon-check" @click="editName=false;updateBook(bookSelected)" size="mini"></el-button>
                        </el-input>
                        <span v-if="!editName">{{bookSelected.name}}<el-button icon="el-icon-edit" size="mini" type="text" @click="editName=true"></el-button></span>
                      </el-form-item>
                      <el-form-item label="Type">
                        <el-input v-if="editType" v-model="bookSelected.type" size="mini" style="width: 220px">
                          <el-button slot="append" icon="el-icon-check" @click="editType=false;updateBook(bookSelected)" size="mini"></el-button>
                        </el-input>
                        <span v-if="!editType"> {{ bookSelected.type ? bookSelected.type : '-' }}
                    <el-button icon="el-icon-edit" size="mini" type="text" @click="editType=true"></el-button></span>
                      </el-form-item>
                      <el-form-item label="Size">
                        {{bookSelected.size / 1000000}}MB
                      </el-form-item>
                      <el-form-item label="Description">
                        <div v-if="editDesc" style="width: 350px">
                          <el-input  v-model="bookSelected.description" size="mini"  type="textarea" :rows="3" resize="none">
                          </el-input>
                          <el-button  icon="el-icon-check" @click="editDesc=false;updateBook(bookSelected)" size="mini"></el-button>
                        </div>
                        <span v-if="!editDesc">{{ bookSelected.description }}
                      <el-button icon="el-icon-edit" size="mini" type="text" @click="editDesc=true"></el-button></span>
                      </el-form-item>
                      <el-form-item label="Path">
                        {{ bookSelected.path }}
                      </el-form-item>
                      <el-form-item v-if="operateSystem == 'Darwin'" size="mini">
                        <div style="margin-left: 40px">
                          <el-button @click="open(bookSelected.path, false)">Open</el-button>
                          <el-button @click="open(bookSelected.path, true)">Open In Finder</el-button>
                        </div>
                      </el-form-item>
                    </el-form>
                  </div>
                </div>
              </el-card>
            </el-main>
          </el-container>
        </el-container>
      </el-container>
    </el-container>
  </div>
</template>

<script>

const dialog = require('electron').remote.dialog
const path = require('path')
const fs = require('fs')
const fileType = require('file-type')
const pdf = require('../assets/pdf.png')
const os = require("os")
const child_process = require('child_process')
// use this instead:

const pdfjsLib =  require("pdfjs-dist/legacy/build/pdf.js")
//pdfjsLib.GlobalWorkerOptions.workerSrc = 'pdfjs-dist/legacy/build/pdf.worker.js';
export default {
  name: 'main-page',
  components: {},
  data() {
    return {
      type: 'All',
      types: [{value: 'All'}, {value: 'No Type Defined'}],
      loading: false,
      db: null,
      books: [],
      bookSelected: null,
      editName: false,
      editType: false,
      editDesc: false,
      dbName: process.env.NODE_ENV === 'production' ? 'book-pro' : 'book',
      name: '',
      operateSystem: os.type()
    }
  },
  mounted() {
    let request = window.indexedDB.open(this.dbName, 1)
    request.onerror = ev => {
    }
    request.onsuccess = ev => {
      this.db = request.result;
      console.log(this.db.version)
      console.log(this.db.objectStoreNames)
      this.refreshAllBooks()
    }
    request.onupgradeneeded = ev => {
      console.log('onupgradeneeded')
      this.db = ev.target.result;
      console.log(this.db.objectStoreNames)
      if (!this.db.objectStoreNames.contains('book')) {
        this.db.createObjectStore('book', {keyPath: 'path'});
        console.log('book created')
      }
    }
  },
  methods: {
    open(filePath, finder) {
      let cmd = `open '${filePath}'`
      if(finder) {
        cmd = `open '${path.dirname(filePath)}'`
      }
      child_process.exec(cmd, (err,stdout,stderr) => {
        if (err){
          console.error(err);
        }
      })
    },
    async selectBook(book) {
      if(this.editDesc || this.editType || this.editName) {
        let book = await this.getBook(this.bookSelected.path)
        if (book) {
          this.bookSelected.type = book.type
          this.bookSelected.description = book.description
        }
        this.editType = false
        this.editDesc = false
        this.editName = false
      }
      this.loading = true
      this.bookSelected = book
      if(!book.icon) {
        book.icon = await this.getFirstPageOfPdf(book.path)
        book.icon = book.icon ? book.icon : pdf
        this.updateBook(book)
      }
      this.drawBookIcon(book.icon)
    },
    drawBookIcon(url) {
      let img = new Image()
      img.src = url
      img.onload = () => {
        let canvas = this.$refs.pdfIcon
        let ctx = canvas.getContext('2d')
        ctx.drawImage(img, 0, 0, 300, 400)
        this.loading = false
      }
    },
    async search() {
      let books = await this.getAllBooks()
      if(this.name) {
        books = books.filter(b => b.name.toLowerCase().indexOf(this.name.toLowerCase()) >= 0)
      }
      if(this.type === 'No Type Defined') {
        books = books.filter(b => !b.type)
      } else if(this.type !== 'All') {
        books = books.filter(b => b.type === this.type)
      }
      this.books = books
      this.bookSelected = null
    },
    getFirstPageOfPdf(pdfPath) {
      return new Promise( resolve => {
        fs.readFile(pdfPath, async (err, buffer) => {
          let url = URL.createObjectURL(new Blob([buffer], { type : 'application/pdf'}))
          pdfjsLib.getDocument(url).promise.then((pdf) => {
            return pdf.getPage(1)
          }).then((page) => {
            // 设置展示比例
            let scale = 0.5;
            // 获取pdf尺寸
            let viewport = page.getViewport({scale});
            // 获取需要渲染的元素
            let canvas = document.createElement('canvas')
            let context = canvas.getContext('2d')
            canvas.height = viewport.height
            canvas.width = viewport.width

            let renderContext = {
              canvasContext: context,
              viewport: viewport
            };
            page.render(renderContext).promise.then(() => {
              resolve(canvas.toDataURL("img/png"))
            }).catch(err => {
              console.error(err)
              resolve('')
            });
          }).catch(err => {
            console.error(err)
            resolve('')
          });
        })
      })
    },
    async openFileDialog() {
      dialog.showOpenDialog({
        properties: ['openFile', 'openDirectory'],
        filters: [{name: 'files', extensions: ['pdf']}]
      }, async (files) => {
        if (files && files.length) {
          let file = files[0];
          this.loading = true
          let books = await this.getAllFiles(file)
          console.log(books)
          for (let book of books) {
            await this.saveBook(book)
          }
          await this.refreshAllBooks()
          this.loading = false
        }
      })
    },
    async saveBook(book) {
      book.description = ''
      book.type = ''
      book.icon = ''
      let request = this.db.transaction(['book'], 'readwrite')
          .objectStore('book')
          .add(book);

      await new Promise((resolve, reject) => {
        request.onsuccess = ev => {
          resolve(true)
        }

        request.onerror = ev => {
          console.error(request.error)
          this.$message(request.error)
          resolve(false)
        }
      })
    },
    async refreshAllBooks() {
      let books = await this.getAllBooks()
      if(books) {
        this.books = books
        this.types = [{value: 'All'}, {value: 'No Type Defined'}]
        let set = new Set();
        this.books.forEach(b => {
          if(b.type && !set.has(b.type)) {
            set.add(b.type);
            this.types.push({
              value: b.type
            })
          }
        })
      }
    },
    async isPdf(filePath) {
      let type = await fileType.fromFile(filePath)
      return type && type.ext == 'pdf'
    },
    async getAllFilesIn(directory) {
      return new Promise(resolve => {
        let books = []
        fs.readdir(directory, async (err, files) => {
          for (let f of files) {
            let subFiles = await this.getAllFiles(path.join(directory, f))
            subFiles.forEach(b => books.push(b))
          }
          resolve(books)
        })
      })
    },
    updateBook(book) {
      let request = this.db.transaction(['book'], 'readwrite')
          .objectStore('book')
          .put(book);

      request.onsuccess = async ev => {
        let books = await this.getAllBooks()
        this.types = [{value: 'All'}, {value: 'No Type Defined'}]
        let set = new Set();
        books.forEach(b => {
          if(b.type && !set.has(b.type)) {
            set.add(b.type);
            this.types.push({
              value: b.type
            })
          }
        })
      };

      request.onerror = ev => {
        console.error(request.error)
        this.$message(request.error)
        this.refreshAllBooks()
      }
    },
    async getAllFiles(file) {
      if (path.basename(file).startsWith('.')) {
        return []
      }
      return new Promise(resolve => {
        let files = []
        fs.stat(file, async (err, stat) => {
          if (stat.isFile()) {
            if (await this.isPdf(file)) {
              files.push({
                name: path.basename(file),
                path: file,
                size: stat.size
              })
            } else {
              console.error(file)
            }

          } else if (stat.isDirectory()) {
            let subFiles = await this.getAllFilesIn(file);
            subFiles.forEach(f => files.push(f))
          }
          resolve(files)
        })
      })
    },
    async getBook(path) {
      return new Promise(resolve => {
        let transaction = this.db.transaction(['book']);
        let objectStore = transaction.objectStore('book');
        let request = objectStore.get(path)

        request.onerror = ev => {
          console.error(request.error)
          this.$message(request.error)
          resolve(null)
        }

        request.onsuccess = ev => {
          if (request.result) {
            resolve(request.result)
          } else {
            resolve(null)
            this.$message("No Data")
          }
        }
      })
    },
    async getAllBooks() {
      return new Promise(resolve => {
        let transaction = this.db.transaction(['book']);
        let objectStore = transaction.objectStore('book');
        let request = objectStore.getAll()

        request.onerror = ev => {
          console.error(request.error)
          this.$message(request.error)
          resolve(null)
        }

        request.onsuccess = ev => {
          if (request.result) {
            resolve(request.result)
          } else {
            resolve([])
            this.$message("No Data")
          }
        }
      })
    }
  }
}
</script>

<style scoped>
.book-list{
  padding: 8px;
  cursor: pointer;
  box-shadow:2px 2px 5px #ccc;
}
.book-list:hover{
  background: #eee;
  color: black;
}
.book-selected {
  font-weight: bold;
  background: #3c3c3c;
  color: white;
  padding: 8px;
  cursor: pointer;
  box-shadow:2px 2px 5px grey;
}
.el-form-item .el-form-item__content .el-input-group {
  vertical-align: middle;
}
</style>