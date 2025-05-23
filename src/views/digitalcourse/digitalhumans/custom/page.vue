<template>
  <ContentWrap>
    <!-- 搜索工作栏 -->
    <el-form
      class="-mb-15px"
      :model="queryParams"
      ref="queryFormRef"
      :inline="true"
      label-width="68px"
    >
      <el-form-item :label="t('digitalhumans.name')" prop="name">
        <el-input
          v-model="queryParams.name"
          :placeholder="t('common.inputText') + t('digitalhumans.name')"
          clearable
          @keyup.enter="handleQuery"
          class="!w-240px"
        />
      </el-form-item>
      <el-form-item>
        <el-button @click="handleQuery"><Icon icon="ep:search" class="mr-5px" /> {{ t('table.search') }}</el-button>
        <el-button @click="resetQuery"><Icon icon="ep:refresh" class="mr-5px" /> {{ t('table.reset') }}</el-button>
        <el-button
          type="primary"
          plain
          @click="openForm('create')"
          v-hasPermi="['digitalcourse:digital-humans:create']"
        >
          <Icon icon="ep:plus" class="mr-5px" /> {{ t('action.add') }}
        </el-button>
        <el-button
          type="success"
          plain
          @click="handleExport"
          :loading="exportLoading"
          v-hasPermi="['digitalcourse:digital-humans:export']"
        >
          <Icon icon="ep:download" class="mr-5px" /> {{ t('action.export') }}
        </el-button>
      </el-form-item>
    </el-form>
  </ContentWrap>

  <!-- 列表 -->
  <ContentWrap>
    <el-table v-loading="loading" :data="list" :stripe="true" :show-overflow-tooltip="true">
      <el-table-column :label="t('table.index')" align="center" prop="id" />
      <el-table-column :label="t('digitalhumans.gender')" align="center" prop="name" />
      <el-table-column :label="t('digitalhumans.code')" align="center" prop="code" />
      <el-table-column
        :label="t('table.createTime')"
        align="center"
        prop="createTime"
        :formatter="dateFormatter"
        width="180px"
      />
      <el-table-column :label="t('digitalhumans.status')" align="center" prop="status">
        <template #default="scope">
          {{getStatusLabel(scope.row.status)}}

        </template>
      </el-table-column>
      <el-table-column :label="t('table.action')" align="center">
        <template #default="scope">
          <el-button
            v-if="superAdminProcess(scope.row.status)"
            :disabled="scope.row.status == 3"
            link
            type="primary"
            @click="openForm('update', scope.row.id)"
          >
            {{t('digitalhumans.handle')}}
          </el-button>
          <el-button
            :disabled="scope.row.status == 1 || scope.row.status == 2 || scope.row.status == 3 "
            link
            type="danger"
            @click="handleDelete(scope.row.id)"
          >
            {{ t('action.del') }}
          </el-button>
        </template>
      </el-table-column>
    </el-table>
    <!-- 分页 -->
    <Pagination
      :total="total"
      v-model:page="queryParams.pageNo"
      v-model:limit="queryParams.pageSize"
      @pagination="getList"
    />
  </ContentWrap>

  <!-- 表单弹窗：添加/修改 -->
  <DigitalHumansForm ref="formRef" @success="getList" />
</template>

<script setup lang="ts">
import { getStatusLabel } from '../../common'
import { getIntDictOptions, DICT_TYPE } from '@/utils/dict'
import { dateFormatter } from '@/utils/formatTime'
import download from '@/utils/download'
import * as DigitalHumansApi from '@/api/digitalcourse/digitalhumans'
import DigitalHumansForm from './DigitalHumansForm.vue'
import { useUserStoreWithOut } from '@/store/modules/user'
const userStore = useUserStoreWithOut() // 用户信息缓存
defineOptions({ name: 'DigitalHumans' })

const message = useMessage() // 消息弹窗
const { t } = useI18n() // 国际化

const memberDelete = (state)=>{
  return userStore.getRoles.indexOf('super_admin') < 0 && ( state > 1 || state == 0)
}
const superAdminProcess = (status)=>{
  return (status == 4 && userStore.getRoles.indexOf('super_admin') < 0) || (userStore.getRoles.indexOf('super_admin') > -1 && status != 0)
}

const loading = ref(true) // 列表的加载中
const list = ref([]) // 列表的数据
const total = ref(0) // 列表的总页数
const queryParams = reactive({
  pageNo: 1,
  pageSize: 10,
  gender: undefined,
  name: undefined,
  posture: undefined,
  type: undefined,
  useModel: undefined,
  status: undefined,
})
const queryFormRef = ref() // 搜索的表单
const exportLoading = ref(false) // 导出的加载中

/** 查询列表 */
const getList = async () => {
  loading.value = true
  try {
    queryParams.type = 1
    const data = await DigitalHumansApi.getDigitalHumansPage(queryParams)
    list.value = data.list
    total.value = data.total
  } finally {
    loading.value = false
  }
}

/** 搜索按钮操作 */
const handleQuery = () => {
  queryParams.pageNo = 1
  getList()
}

/** 重置按钮操作 */
const resetQuery = () => {
  queryFormRef.value.resetFields()
  handleQuery()
}

/** 添加/修改操作 */
const formRef = ref()
const openForm = (type: string, id?: number) => {
  formRef.value.open(type, id)
}

/** 删除按钮操作 */
const handleDelete = async (id: number) => {
  try {
    // 删除的二次确认
    await message.delConfirm()
    // 发起删除
    await DigitalHumansApi.deleteDigitalHumans(id)
    message.success(t('common.delSuccess'))
    // 刷新列表
    await getList()
  } catch {}
}

/** 导出按钮操作 */
const handleExport = async () => {
  try {
    // 导出的二次确认
    await message.exportConfirm()
    // 发起导出
    exportLoading.value = true
    const data = await DigitalHumansApi.exportDigitalHumans(queryParams)
    download.excel(data, '存储数字人信息，包括数字人的基本属性、图片、姿势等信息.xls')
  } catch {
  } finally {
    exportLoading.value = false
  }
}

/** 初始化 **/
onMounted(() => {
  getList()
})
</script>
