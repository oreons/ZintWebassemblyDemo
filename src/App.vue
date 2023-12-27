<template>
  <div class="canvas">
    <h3>Zint {{zintVersion}} webassembly test</h3>
    <canvas ref = "canvasRef"/>
  </div>
  <div class = "params-wrapper">
    <div class = "params">
      <div class = "col-params">
        <span class = "col-label">Data:</span> <textarea  rows="5" cols="23"  v-model="data"/>
      </div>
      <div class = "col-params">
        <span class = "col-label">Height:</span> <input type="text" v-model="config.height"/>
      </div>
      <div class = "col-params">
        <span class = "col-label">Scale:</span> <input type="text" v-model="config.scale"/>
      </div>
      <div class = "col-params">
        <span class = "col-label">Color:</span> <input type="text" v-model="config.fgcolour"/>
      </div>
      <div class = "col-params">
        <span class = "col-label">Type:</span>
        <select v-model = "config.symbology">
        <option
            v-for = "item of barcode_items"
            :key = "item[0]"
            :value = "item[1]">
          {{item[0]}}
        </option>
      </select>
      </div>
      <div class = "col-params">
        <span class = "col-label">Text:</span> <input type="checkbox" v-model = "config.show_hrt">
      </div>
      <div v-if="error" class="error">{{error}}</div>
    </div>
  </div>
</template>

<script setup>
import wasmzint from "wasmzint"
import {BARCODE_EANX, barcode_items} from '@/assets/constants.js'
import {computed, onMounted, reactive, ref, watch} from "vue"

const config = reactive({
  symbology: BARCODE_EANX,
  height: 50,
  scale:  1.5,
  fgcolour: '000000',
  show_hrt: true,
})

const data  = ref('123456789012')
const error = ref('')
const version = ref(0)

const canvasRef = ref()
let zintModule = null
let zintInstance = null

const zintVersion = computed (()=>{
  const v1 = Math.floor(version.value/10000)
  const v2 = Math.floor((version.value-v1*10000)/100)
  const v3 = version.value-v1*10000-v2*100
  return `${v1}.${v2}.${v3}`
})
const showBarcode = async ()=>{
  error.value = ''
  const out = zintInstance.encodeAndPrint({...config, outfile: 'out.svg'},
                                           data.value)
  let image = new Image()
  if(!out.result){
    error.value = out.error
  }
  else {
    image = await svgToImage(new TextDecoder().decode(out.file))
  }
  canvasRef.value.width = image.width
  canvasRef.value.height = image.height
  const ctx = canvasRef.value.getContext('2d')
  ctx.drawImage(image, 0, 0)
}

onMounted(async ()=>{
  zintModule = await wasmzint()
  zintInstance = new zintModule.ZintWrapper()
  version.value = zintInstance.version()
  await showBarcode()

})

watch(()=>[data.value, config], async ()=>{
  await showBarcode()
}, {deep: true})

const svgToImage = svgText =>{
  return new Promise((resolve, reject) => {
    const image = new Image()
    const src = svgToURL(svgText)
    image.onload  = () => {
      return resolve(image)
    }
    image.onerror = e => reject(e)
    image.src = src
  })
}

const svgToURL = e => {
  return "data:image/svg+xml;base64,"+window.btoa(unescape(encodeURIComponent(e)))
}



</script>


<style scoped>
.error {
  color: red;
  font: 1rem sans-serif;
}
.canvas {
  top: 20px;
  min-height: 400px;
  text-align: center;
}
.params {
  display: flex;
  flex-direction: column;
  justify-content: flex-start;
  margin-top: 10px;
  align-content: center;
}

.params-wrapper {
  display: flex;
  justify-content: center;
}
.col-params {
  display: flex;
  justify-content: flex-start;
  align-items: center;
  align-content: center;
  flex-wrap: wrap;
  margin: 2px;
}

.col-label {
  min-width: 80px;
  font: 1.1rem sans-serif;
}
input, textarea, select {
  font: 1.1rem sans-serif;
}

</style>
