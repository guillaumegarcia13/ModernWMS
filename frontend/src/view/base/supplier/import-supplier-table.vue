<!-- Supplier Setting Import Dialog -->
<template>
  <v-dialog v-model="isShow" width="70%" transition="dialog-top-transition" :persistent="true">
    <template #default>
      <v-card>
        <v-toolbar color="white" :title="`${$t('system.page.import')}`"></v-toolbar>
        <v-card-text>
          <div class="mb-4">
            <tooltip-btn icon="mdi-plus" :tooltip-text="$t('system.page.chooseFile')" size="x-small" @click="method.chooseFile"></tooltip-btn>
            <tooltip-btn
              icon="mdi-export-variant"
              :tooltip-text="$t('system.page.exportTemplate')"
              size="x-small"
              @click="method.exportTemplate"
            ></tooltip-btn>
            <input v-show="false" id="open" ref="uploadExcel" type="file" accept=".xls, .xlsx" @change="method.readExcel" />
          </div>
          <vxe-table
            ref="xTable"
            :column-config="{ minWidth: '100px' }"
            :data="data.importData"
            :height="SYSTEM_HEIGHT.SELECT_TABLE"
            :edit-config="{ trigger: 'click', mode: 'cell' }"
            :edit-rules="data.validRules"
            :export-config="{}"
            align="center"
          >
            <template #empty>
              {{ i18n.global.t('system.page.noData') }}
            </template>
            <vxe-column type="seq" width="60"></vxe-column>
            <vxe-column field="operate" width="60" :title="$t('system.page.operate')" :resizable="false">
              <template #default="{ row }">
                <tooltip-btn
                  :flat="true"
                  icon="mdi-delete-outline"
                  :tooltip-text="$t('system.page.delete')"
                  :icon-color="errorColor"
                  @click="method.deleteRow(row)"
                ></tooltip-btn>
              </template>
            </vxe-column>
            <vxe-column field="supplier_name" :title="$t('base.supplier.supplier_name')"></vxe-column>
            <vxe-column field="city" :title="$t('base.supplier.city')"></vxe-column>
            <vxe-column field="address" :title="$t('base.supplier.address')"></vxe-column>
            <vxe-column field="manager" :title="$t('base.supplier.manager')"></vxe-column>
            <vxe-column field="email" :title="$t('base.supplier.email')"></vxe-column>
            <vxe-column field="contact_tel" :title="$t('base.supplier.contact_tel')"></vxe-column>
          </vxe-table>
        </v-card-text>
        <v-card-actions class="justify-end">
          <v-btn variant="text" @click="method.closeDialog">{{ $t('system.page.close') }}</v-btn>
          <v-btn color="primary" variant="text" @click="method.submit">{{ $t('system.page.submit') }}</v-btn>
        </v-card-actions>
      </v-card>
    </template>
  </v-dialog>
</template>

<script lang="ts" setup>
import { reactive, computed, ref, watch } from 'vue'
import { VxeTablePropTypes } from 'vxe-table'
import * as XLSX from 'xlsx'
import i18n from '@/languages/i18n'
import { excelImport } from '@/api/base/supplier'
import { hookComponent } from '@/components/system/index'
import { SYSTEM_HEIGHT, errorColor } from '@/constant/style'
import tooltipBtn from '@/components/tooltip-btn.vue'
import { SupplierExcelVO } from '@/types/Base/Supplier'
import { exportData } from '@/utils/exportTable'

const emit = defineEmits(['close', 'saveSuccess'])
const uploadExcel = ref()
const xTable = ref()

const props = defineProps<{
  showDialog: boolean
}>()

const isShow = computed(() => props.showDialog)

const data = reactive({
  importData: ref<Array<SupplierExcelVO>>([]),
  validRules: ref<VxeTablePropTypes.EditRules>({})
})

const method = reactive({
  initForm: () => {
    data.importData = []
  },
  closeDialog: () => {
    emit('close')
  },
  submit: async () => {
    const isValid = method.valid()
    if (!isValid) {
      return
    }

    const $table = xTable.value
    // It must be use 'getTableData()' to get all datas with table because it will delete row sometimes.
    const importData = $table.getTableData().fullData

    const { data: res } = await excelImport(importData)
    if (!res.isSuccess) {
      hookComponent.$message({
        type: 'error',
        content: res.errorMessage
      })
      return
    }
    hookComponent.$message({
      type: 'success',
      content: `${ i18n.global.t('system.page.submit') }${ i18n.global.t('system.tips.success') }`
    })
    emit('saveSuccess')
  },

  valid: () => {
    const $table = xTable.value
    const importData = $table.getTableData().fullData

    if (importData.length <= 0) {
      hookComponent.$message({
        type: 'error',
        content: `${ i18n.global.t('system.tips.detailLengthIsZero') }`
      })
      return false
    }
    return true
  },

  chooseFile: async () => {
    uploadExcel.value.value = ''
    uploadExcel.value.click()
  },

  readExcel: async (evnt: any) => {
    const files = evnt.target.files
    if (files.length <= 0) {
      return false
    }

    const file = files[0]
    const fileReader = new FileReader()

    fileReader.onload = (ev: ProgressEvent<FileReader>) => {
      const fileData = ev?.target?.result
      if (fileData) {
        const workbook = XLSX.read(fileData, { type: 'binary' })
        const wsname = workbook.SheetNames[0]

        let ws = XLSX.utils.sheet_to_json(workbook.Sheets[wsname])
        let str = JSON.stringify(ws)
        str = str.replace(/（/g, '(')
        str = str.replace(/）/g, ')')
        ws = JSON.parse(str)

        data.importData = []
        ws.forEach((value: any, index: number, ws: any) => {
          data.importData.push({
            supplier_name: ws[index][i18n.global.t('base.supplier.supplier_name')] ? ws[index][i18n.global.t('base.supplier.supplier_name')] + '' : '',
            city: ws[index][i18n.global.t('base.supplier.city')] ? ws[index][i18n.global.t('base.supplier.city')] + '' : '',
            address: ws[index][i18n.global.t('base.supplier.address')] ? ws[index][i18n.global.t('base.supplier.address')] + '' : '',
            manager: ws[index][i18n.global.t('base.supplier.manager')] ? ws[index][i18n.global.t('base.supplier.manager')] + '' : '',
            email: ws[index][i18n.global.t('base.supplier.email')] ? ws[index][i18n.global.t('base.supplier.email')] + '' : '',
            contact_tel: ws[index][i18n.global.t('base.supplier.contact_tel')] ? ws[index][i18n.global.t('base.supplier.contact_tel')] + '' : ''
          })
        })
      }
    }
    fileReader.readAsBinaryString(file)
  },

  exportTemplate: () => {
    const $table = xTable.value
    exportData({
      table: $table,
      filename: i18n.global.t('router.sideBar.supplier'),
      mode: 'header',
      columnFilterMethod({ column }: any) {
        return !['checkbox', 'seq'].includes(column?.type) && !['operate'].includes(column?.field)
      }
    })
  },

  deleteRow: (row: SupplierExcelVO) => {
    hookComponent.$dialog({
      content: i18n.global.t('system.tips.beforeDeleteDetailMessage'),
      handleConfirm: async () => {
        if (row) {
          const $table = xTable.value
          $table.remove(row)
        }
      }
    })
  }
})

watch(
  () => isShow.value,
  (val) => {
    if (val) {
      method.initForm()
    }
  }
)
</script>

<style scoped lang="less">
.v-form {
  div {
    margin-bottom: 7px;
  }
}
</style>
