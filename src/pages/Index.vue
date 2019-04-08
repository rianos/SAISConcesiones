<template>
  <q-page class="flex flex-center">
<q-stepper vertical ref='stepper' v-show='visible'>
  <q-step default  title='Seleccione fichero de SISS concesiones'>
    <q-uploader hide-upload-button @add="loadFile('fsiss', ...arguments)" url=''/>
  </q-step>
  <q-step title='Seleccione fichero de incidencias de nóminas'>
    <q-uploader hide-upload-button @add="loadFile('fnominas', ...arguments)" url=''/>
  </q-step>
  <q-step title='procesar'>
    <q-btn @click='procesa' label='Calcula'/>
  </q-step>
</q-stepper>
<q-card v-show='!visible'>
  <q-card-title class='bg-primary text-white'>
    Resultados
  </q-card-title>
  <q-card-separator />
  <q-card-main>
    <div>Usuarios en listado SISS: <q-chip color='red'>{{siss.length}}</q-chip></div>
    <div>Usuarios en listado Nominas: <q-chip color='green'>{{nomina.length}}</q-chip></div>
    <q-list>
      <q-list-header>Está en Nomina pero no en SISS</q-list-header>
      <q-item v-for='item in resultadonns' :key='item'>
        {{item}}
      </q-item>
      <q-list-header>Está en SISS, pero no en nomina</q-list-header>
       <q-item v-for='item2 in resultadosnn' :key='item2'>
        {{item2}}
      </q-item>
    </q-list>
  </q-card-main>
</q-card>
  </q-page>
</template>

<style>
</style>

<script>
export default {
  name: 'PageIndex',
  data: function () {
    return {
      fnominas: null,
      fsiss: null,
      textToSearch: /ALTA POR NUEVA CONCESION/,
      nomina: [],
      siss: [],
      PDFLib: null,
      resultadonns: '',
      resultadosnn: '',
      visible: true
    }
  },
  methods: {
    buscaDNI (doc) {
      let regexp = /[0-9]{8}[TRWAGMYFPDXBNJZSQVHLCKE]/ig
      let res = doc.match(regexp)
      return res
    },
    buscaNIE (doc) {
      let regexp = /[XYZ][0-9]{7}[TRWAGMYFPDXBNJZSQVHLCKE]/ig
      let res = doc.match(regexp)
      return res
    },
    readfileandextracText (file) {
      let self = this
      return new Promise((resolve, reject) => {
        self.PDFLib.getDocument({ data: file }).then(function (PDFdoc) {
          console.log('PDF loaded')
          let numpages = PDFdoc._pdfInfo.numPages
          console.log(numpages)
          let promises = []
          for (let i = 0; i < numpages; i++) {
            promises[i] = self.getTextFromPage(i + 1, PDFdoc)
          }
          Promise.all(promises).then(pdftxt => {
            resolve(pdftxt)
          })
        })
      })
    },
    procesaNominas () {
      let self = this
      return new Promise((resolve, reject) => {
        this.readfileandextracText(this.fnominas).then(pdftxt => {
          for (let j = 0; j < pdftxt.length; j++) {
            if (pdftxt[j].search(self.textToSearch) !== -1) {
              console.log('Pagina ' + (j + 1))
              let dni = self.buscaDNI(pdftxt[j])
              let nie = self.buscaNIE(pdftxt[j])
              if (dni !== null) {
                self.nomina = self.nomina.concat(dni)
              }
              if (nie !== null) {
                self.nomina = self.nomina.concat(nie)
              }
            }
          }
          resolve(self.nomina)
        })
      })
    },
    procesaSISS () {
      let self = this
      return new Promise((resolve, reject) => {
        this.readfileandextracText(this.fsiss).then(pdftxt => {
          for (let j = 0; j < pdftxt.length; j++) {
            console.log('Pagina SISS ' + (j + 1))
            let dni = self.buscaDNI(pdftxt[j])
            let nie = self.buscaNIE(pdftxt[j])
            if (dni !== null) {
              self.siss = self.siss.concat(dni)
            }
            if (nie !== null) {
              self.siss = self.siss.concat(nie)
            }
          }
          resolve(self.siss)
        })
      })
    },
    arr_diff (a1, a2) {
      let diff = []
      for (let i = 0; i < a1.length; i++) {
        if (a2.indexOf(a1[i]) === -1) {
          diff.push(a1[i])
        }
      }
      return diff
    },
    procesa () {
      var PDFJS = window['pdfjs-dist/build/pdf']
      PDFJS.GlobalWorkerOptions.workerSrc = 'https://cdn.jsdelivr.net/npm/pdfjs-dist@2.0.943/build/pdf.worker.js'
      this.PDFLib = PDFJS
      let promises = []
      let self = this
      promises[0] = this.procesaNominas()
      promises[1] = this.procesaSISS()
      Promise.all(promises).then(res => {
        console.log(res[0])
        console.log(res[1])
        let amb = this.arr_diff(res[0], res[1])
        let bma = this.arr_diff(res[1], res[0])
        self.resultadonns = amb
        self.resultadosnn = bma
        self.visible = false
      })
    },
    getTextFromPage (pageNum, pdf) {
      return new Promise((resolve, reject) => {
        pdf.getPage(pageNum).then(page => {
          page.getTextContent().then(textcon => {
            let items = textcon.items
            let text = ''
            for (let i = 0; i < items.length; i++) {
              text += items[i].str + ' '
            }
            resolve(text)
          })
        })
      })
    },
    loadFile (name, files) {
      let file = files[0]
      let reader = new FileReader()
      reader.readAsBinaryString(file)
      let self = this
      reader.onload = function (fileEvent) {
        self[name] = fileEvent.target.result
        console.log(self)
        self.$refs.stepper.next()
      }
    }
  }
}
</script>
