<script setup lang="ts">
import { utils, writeFile, readFile } from 'xlsx'
import { ref } from 'vue'
import type {Ref} from 'vue'

const singleMst: Ref<string> = ref('0302218267')
const soloResult: Ref<string> = ref('')
const multiMst: Ref<string[]> = ref([])
const file = ref<File | null | undefined>()
const isLoading: Ref<boolean> = ref(false)
const missingMst: Ref<string[]> = ref([])
const errorMst: Ref<string[]> = ref([])
const tooManyMst: Ref<string[]> = ref([])
const timeTaken: Ref<number> = ref(0)

const handleFileChange = async ($event: Event) => {
  const fileList = ($event.target as HTMLInputElement).files || []
  if (fileList.length) {
    file.value = fileList[0]
    const workbook = readFile((await file.value.arrayBuffer() as unknown as string))

    const worksheet = workbook.Sheets[workbook.SheetNames[0]]
    const raw_data: string[][] = utils.sheet_to_json(worksheet, { header: 1, raw: true })
    const mstList = raw_data.map((x) => x[0])
    multiMst.value = mstList
  }
}

const resetResult = () => {
  missingMst.value = []
  errorMst.value = []
  tooManyMst.value = []
  timeTaken.value = 0
}

const fetchFromApi = async (mst: string[]) => {
  isLoading.value = true;

  const result = await fetch(`${import.meta.env.VITE_BACKEND_URL}/fetch`, {
    method: 'post',
    body: JSON.stringify({ mst }),
    headers: {
      'Content-Type': 'application/json'
    }
  })

  const { data, missingCode, errorCode, time, tooManyCode } = await result.json()
  isLoading.value = false;
  missingMst.value = missingCode.map((x: Record<string, string>) => x.mst)
  errorMst.value = errorCode.map((x: Record<string, string>) => x.mst)
  tooManyMst.value = tooManyCode.map((x: Record<string, string>) => x.mst)
  timeTaken.value = time

  return data
}
const soloScrape = async () => {
  try {
    const { mst, comp, name, address } = (await fetchFromApi([singleMst.value]))[0]
    soloResult.value = `${mst} - ${comp} - ${name} - ${address}`
  } catch (e) {
    isLoading.value = false
  }
}

const multiScrape = async () => {
  try {
    resetResult();
    const list = await fetchFromApi(multiMst.value)
  

  /* generate worksheet and workbook */


  const worksheet = utils.json_to_sheet(list.map(
    (row: Record<string, string>) => [row.mst, row.name, row.address])
  );
  const workbook = utils.book_new();
  utils.book_append_sheet(workbook, worksheet, "MST");

  /* fix headers */
  utils.sheet_add_aoa(worksheet, [["MST", "Công ty", "Địa chỉ"]], { origin: "A1" });


  /* calculate column width */
  const mstMaxWidth = list.reduce((w: number, r: Record<string, string>) => Math.max(w, r.mst.length), 10);
  const nameMaxWidth = list.reduce((w: number, r: Record<string, string>) => Math.max(w, r.name.length), 10);
  const addressMaxWidth = list.reduce((w: number, r: Record<string, string>) => Math.max(w, r.address.length), 10);

  worksheet["!cols"] = [
    { wch: mstMaxWidth },
    { wch: nameMaxWidth },
    { wch: addressMaxWidth }
  ];

  /* create an XLSX file and try to save to Presidents.xlsx */
    writeFile(workbook, "output.xlsx", { compression: true });
  } catch (e) {
    isLoading.value = false
  }
}
</script>

<template>
  <header>
    <div class="wrapper">
      <label>single</label>
      <br />
      <input type="text" v-model="singleMst" />
      <button @click="soloScrape" v-bind:disabled="isLoading">Go</button>
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
      <button @click="multiScrape" v-bind:disabled="isLoading">Go</button>
      <br />
      <span v-if="file">{{ multiMst.length }} mst found on file</span>
      <br />
      <div v-if="isLoading">
        <span>Fetching ...</span>
        <div class="loader"></div>
      </div>
      <br />

      <ul v-if="timeTaken">
        <li>time taken: {{ timeTaken }} ms</li>
        <li>missing mst (đúng format nhưng không tìm ra)
          <template v-if="missingMst.length">
            <ul>
              <li v-for="mst in missingMst" v-bind:key="`mst-missing-${mst}`"> {{ mst }}</li>
            </ul>
          </template>
          <span v-else>None</span>
        </li>
        <li>error mst (sai format)
          <template v-if="errorMst.length">
            <ul>
              <li v-for="mst in errorMst" v-bind:key="`mst-error-${mst}`"> {{ mst }}</li>
            </ul>
          </template>
          <span v-else>None</span>
        </li>
        <li>Too many mst (fetch nhanh quá nó bị rate limit) -- bấm lại go lần nữa cho nó làm lại :v
          <template v-if="tooManyMst.length">
            <ul>
              <li v-for="mst in tooManyMst" v-bind:key="`mst-too-many-${mst}`"> {{ mst }}</li>
            </ul>
          </template>
          <span v-else>None</span>
        </li>
      </ul>
    </div>
  </header>
</template>
<style scoped>
.loader {
  border: 16px solid #f3f3f3;
  border-radius: 50%;
  border-top: 16px solid blue;
  border-bottom: 16px solid blue;
  width: 60px;
  height: 60px;
  -webkit-animation: spin 2s linear infinite;
  animation: spin 2s linear infinite;
}

@-webkit-keyframes spin {
  0% { -webkit-transform: rotate(0deg); }
  100% { -webkit-transform: rotate(360deg); }
}

@keyframes spin {
  0% { transform: rotate(0deg); }
  100% { transform: rotate(360deg); }
}
</style>
