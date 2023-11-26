<script setup lang="ts">
import { utils, writeFile, readFile } from 'xlsx'
import { ref } from 'vue'

const singleMst = ref('0302218267')
const soloResult = ref('')
const multiMst = ref<string[]>([])
const file = ref<File | null | undefined>()
const handleFileChange = async ($event: Event) => {
  const fileList = ($event.target as HTMLInputElement).files || []
  if (fileList.length) {
    file.value = fileList[0]
    console.log('is it?')
    // const workbook = read(file.value.arrayBuffer())
    const workbook = readFile((await file.value.arrayBuffer()) as unknown as string)

    const worksheet = workbook.Sheets[workbook.SheetNames[0]]
    const raw_data = utils.sheet_to_json(worksheet, { header: 1, raw: true })
    const mstList = raw_data.map((x: any) => x[0])
    multiMst.value = mstList
  }
}

const fetchFromApi = async (mst: string[]) => {
  const result = await fetch(`${import.meta.env.VITE_BACKEND_URL}/fetch`, {
    method: 'post',
    body: JSON.stringify({ mst }),
    headers: {
      'Content-Type': 'application/json'
    }
  })

  const { data } = await result.json()
  return data
}
const soloScrape = async () => {
  const { mst, comp, name, address } = (await fetchFromApi([singleMst.value]))[0]

  soloResult.value = `${mst} - ${comp} - ${name} - ${address}`
}

const multiScrape = async () => {
  const list = await fetchFromApi(multiMst.value)
  console.log(list)
}
</script>

<template>
  <header>
    <div class="wrapper">
      <label>single</label>
      <br />
      <input type="text" v-model="singleMst" />
      <button @click="soloScrape">Go</button>
      <br />
      <span>{{ soloResult }}</span>
      <br />
      <br />
      <br />
      <label
        >file list (nhớ upload excel file và để mst ở cột đầu tiên và format cột đó về
        string)</label
      >
      <br />
      <input
        type="file"
        accept=".csv, application/vnd.openxmlformats-officedocument.spreadsheetml.sheet, application/vnd.ms-excel"
        @change="handleFileChange"
      />
      <button @click="multiScrape">Go</button>
      <br />
      <span v-if="file">{{ multiMst.length }} mst found on file</span>
      <br />
    </div>
  </header>
</template>
