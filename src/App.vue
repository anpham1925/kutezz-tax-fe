<script setup lang="ts">
import { utils, writeFile, readFile } from 'xlsx'
import { ref } from 'vue'
import type {Ref} from 'vue'

const singleMst: Ref<string> = ref('0302218267')
const soloResult: Ref<string> = ref('')
const multiMst: Ref<string[]> = ref([])
const file = ref<File | null | undefined>()
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

  /* generate worksheet and workbook */


  const worksheet = utils.json_to_sheet(list.map(
    (row: Record<string, string>) => [row.mst, row.name, row.comp, row.address])
  );
  const workbook = utils.book_new();
  utils.book_append_sheet(workbook, worksheet, "MST");

  /* fix headers */
  utils.sheet_add_aoa(worksheet, [["MST", "Tên", "Công ty", "Địa chỉ"]], { origin: "A1" });


  /* calculate column width */
  const mstMaxWidth = list.reduce((w: number, r: Record<string, string>) => Math.max(w, r.mst.length), 10);
  const nameMaxWidth = list.reduce((w: number, r: Record<string, string>) => Math.max(w, r.name.length), 10);
  const compMaxWidth = list.reduce((w: number, r: Record<string, string>) => Math.max(w, r.comp.length), 10);
  const addressMaxWidth = list.reduce((w: number, r: Record<string, string>) => Math.max(w, r.address.length), 10);

  worksheet["!cols"] = [
    { wch: mstMaxWidth },
    { wch: nameMaxWidth },
    { wch: compMaxWidth },
    { wch: addressMaxWidth }
  ];

  /* create an XLSX file and try to save to Presidents.xlsx */
  writeFile(workbook, "output.xlsx", { compression: true });
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
