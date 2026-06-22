<template>
  <div class="reg-key">
    <div class="header-actions">
      <Icon class="icon" icon="ion:add-outline" width="23" height="23" @click="openAdd"/>
      <Icon class="icon" icon="mdi:file-multiple-outline" width="23" height="23" @click="openBatchGen"/>
      <Icon class="icon" icon="mdi:file-export-outline" width="23" height="23" @click="exportKeys"/>
      <div class="search">
        <el-input
            v-model="params.code"
            class="search-input"
            :placeholder="$t('searchRegKeyDesc')"
        >
        </el-input>
      </div>
      <el-select v-model="params.status" class="status-select" :placeholder="$t('filterStatus')" @change="search" clearable>
        <el-option :label="$t('unused')" value="unused"/>
        <el-option :label="$t('used')" value="used"/>
        <el-option :label="$t('expired')" value="expired"/>
      </el-select>
      <Icon class="icon" icon="iconoir:search" @click="search" width="20" height="20"/>
      <Icon class="icon" icon="ion:reload" width="18" height="18" @click="refresh"/>
      <Icon class="icon" icon="fluent:broom-sparkle-16-regular" width="22" height="22" @click="clearNotUse"/>
    </div>

    <!-- 批量操作栏 -->
    <div class="batch-actions" v-if="selectedKeys.size > 0">
      <span>{{ $t('selectContacts') }} <span class="selected-count">{{ selectedKeys.size }}</span> {{ $t('action') }}</span>
      <el-button size="small" @click="selectAll">{{ $t('all') }}</el-button>
      <el-button size="small" @click="deselectAll">{{ $t('cancel') }}</el-button>
      <el-button size="small" @click="exportSelected">{{ $t('exportKeys') }}</el-button>
      <el-button size="small" type="danger" @click="deleteSelected">{{ $t('delete') }}</el-button>
    </div>

    <el-scrollbar class="scrollbar">
      <div  class="loading" :class="regKeyLoading ? 'loading-show' : 'loading-hide'" :style="regKeyFirst ? 'background: transparent' : ''">
        <loading/>
      </div>
      <div class="code-box">
        <div class="code-item" :class="{ 'selected': selectedKeys.has(item.regKeyId) }" v-for="item in regKeyData" :key="item.regKeyId">
          <div class="checkbox-wrapper">
            <el-checkbox :model-value="selectedKeys.has(item.regKeyId)" @change="toggleSelect(item)"></el-checkbox>
          </div>
          <div class="code-info">
            <div class="info-left">
              <div class="info-left-item">
                <span class="code" @click="copyCode(item.code)">{{ item.code }}</span>
              </div>
              <div class="info-left-item">
                <div>{{ $t('remainingUses') }}：</div>
                <div v-if="item.count">{{ item.count }}</div>
                <el-tag v-else type="danger">{{ $t('exhausted') }}</el-tag>
              </div>
              <div class="info-left-item">
                <div>{{ $t('roleDesc') }}：</div>
                <el-tag>{{ item.roleName }}</el-tag>
              </div>
              <div class="info-left-item">
                <div>{{ $t('validUntil') }}：</div>
                <div v-if="item.expireTime">{{ formatExpireTime(item.expireTime) }}</div>
                <el-tag v-else type="danger">{{ $t('expired') }}</el-tag>
              </div>
            </div>
            <div class="info-right">
              <el-dropdown class="setting">
                <Icon icon="fluent:settings-24-filled" width="21" height="21" color="#909399"/>
                <template #dropdown>
                  <el-dropdown-menu>
                    <el-dropdown-item @click="copyCode(item.code)">{{ $t('copy') }}</el-dropdown-item>
                    <el-dropdown-item @click="openHistory(item)">{{ $t('history') }}</el-dropdown-item>
                    <el-dropdown-item @click="deleteRegKey(item)">{{ $t('delete') }}</el-dropdown-item>
                  </el-dropdown-menu>
                </template>
              </el-dropdown>
            </div>
          </div>
        </div>
      </div>
      <div class="empty" v-if="regKeyData.length === 0">
        <el-empty v-if="!regKeyFirst" :image-size="isMobile ? 120 : null" :description="$t('noCodeFound')"/>
      </div>
    </el-scrollbar>
    <el-dialog v-model="showAdd" :title="$t('addRegKey')">
      <div class="container">
        <el-input v-model="addForm.code" :placeholder="$t('regKey')">
          <template #suffix>
            <Icon @click.stop="genCode" class="gen-code" icon="bitcoin-icons:refresh-filled" width="24" height="24"/>
          </template>
        </el-input>
        <el-select v-model="addForm.roleId" :placeholder="$t('roleDesc')">
          <el-option v-for="item in roleList" :label="item.name" :value="item.roleId" :key="item.roleId"/>
        </el-select>
        <el-date-picker
            v-model="addForm.expireTime"
            type="date"
            :placeholder="$t('validUntil')"
        />
        <el-input-number v-model="addForm.count" :min="1" :max="99999"/>
        <el-button class="btn" type="primary" @click="submit" :loading="addLoading"
        >{{ $t('add') }}
        </el-button>
      </div>
    </el-dialog>
    <el-dialog v-model="showBatchGen" :title="$t('batchGenTitle')">
      <div class="container">
        <el-input-number v-model="batchForm.count" :min="1" :max="1000" :placeholder="$t('generateCount')" style="width: 100%"/>
        <el-select v-model="batchForm.roleId" :placeholder="$t('roleDesc')">
          <el-option v-for="item in roleList" :label="item.name" :value="item.roleId" :key="item.roleId"/>
        </el-select>
        <el-date-picker
            v-model="batchForm.expireTime"
            type="date"
            :placeholder="$t('validUntil')"
            style="width: 100%"
        />
        <el-button class="btn" type="primary" @click="submitBatch" :loading="batchLoading"
        >{{ $t('batchGenerate') }}
        </el-button>
      </div>
    </el-dialog>
    <el-dialog class="history-list" v-model="showRegKeyHistory" :title="$t('useHistory')">
      <div class="loading" :class="historyLoading ? 'loading-show' : 'loading-hide'">
        <loading/>
      </div>
      <el-table v-if="!historyLoading" :data="historyList" :fit="true" style="height: 100%">
        <el-table-column :min-width="emailColumnWidth" property="email" :label="$t('user')"
                         :show-overflow-tooltip="true"/>
        <el-table-column :width="createTimeColumnWidth" :formatter="formatUserCreateTime" property="createTime"
                         :label="$t('date')" fixed="right" :show-overflow-tooltip="true"/>
      </el-table>
    </el-dialog>
  </div>
</template>

<script setup>
import {defineOptions, nextTick, reactive, ref, watch} from "vue"
import {Icon} from "@iconify/vue";
import loading from "@/components/loading/index.vue";
import {useSettingStore} from "@/store/setting.js";
import {roleSelectUse} from "@/request/role.js";
import {useRoleStore} from "@/store/role.js";
import {regKeyAdd, regKeyList, regKeyClearNotUse, regKeyDelete, regKeyHistory} from "@/request/reg-key.js";
import {getTextWidth} from "@/utils/text.js";
import dayjs from "dayjs";
import {tzDayjs} from "@/utils/day.js";
import {useI18n} from "vue-i18n";

defineOptions({
  name: 'reg-key'
})

const roleStore = useRoleStore();
const settingStore = useSettingStore();
const params = reactive({
  code: '',
  status: null
})

const {t} = useI18n()
const roleList = reactive([])
const addLoading = ref(false)
const showAdd = ref(false)
const regKeyLoading = ref(true)
const regKeyFirst = ref(true)
const showRegKeyHistory = ref(false)
const historyList = reactive([])
const emailColumnWidth = ref(0)
const createTimeColumnWidth = ref(0)
const historyLoading = ref(false)
const isMobile = window.innerWidth < 1025

const addForm = reactive({
  code: '',
  count: 1,
  roleId: null,
  expireTime: null
})

const batchForm = reactive({
  count: 10,
  roleId: null,
  expireTime: null
})

const showBatchGen = ref(false)
const batchLoading = ref(false)

const selectedKeys = reactive(new Set())

const regKeyData = reactive([])

getList(true)

roleSelectUse().then(list => {
  roleList.length = 0
  roleList.push(...list)
})

watch(() => roleStore.refresh, () => {
  roleSelectUse().then(list => {
    roleList.length = 0
    roleList.push(...list)
  })
})

function openHistory(regKey) {

  historyList.length = 0
  historyLoading.value = true
  regKeyHistory(regKey.regKeyId).then(list => {

    historyList.push(...list)
    if (list.length > 0) {

      const email = list.reduce((a, b) =>
          compareByLengthAndUpperCase(a, b, 'email')
      ).email;

      emailColumnWidth.value = getTextWidth(email) + 30
      emailColumnWidth.value = emailColumnWidth.value < 300 ? emailColumnWidth.value : 300
      const createTime = list.reduce((a, b) =>
          compareByLengthAndUpperCase(a, b, 'createTime')
      ).createTime;
      createTimeColumnWidth.value = getTextWidth(createTime)
    }

  }).finally(() => {
    historyLoading.value = false
  })

  showRegKeyHistory.value = true
}

const compareByLengthAndUpperCase = (a, b, key) => {
  const getUpperCaseCount = (str) => (str.match(/[A-Z]/g) || []).length;
  if (a[key].length === b[key].length) {
    return getUpperCaseCount(a[key]) > getUpperCaseCount(b[key]) ? a : b;
  }
  return a[key].length > b[key].length ? a : b;
};

function formatUserCreateTime(regKey) {
  const createTime = tzDayjs(regKey.createTime);
  const currentYear = dayjs().year();
  const expireYear = createTime.year();

  if (settingStore.lang === 'en') {

    if (expireYear === currentYear) {
      return createTime.format('MMM D, HH:mm');
    } else {
      return createTime.format('MMM D, YYYY HH:mm');
    }

  } else {

    if (expireYear === currentYear) {
      return createTime.format('M月D日 HH:mm');
    } else {
      return createTime.format('YYYY年M月D日 HH:mm');
    }

  }

}

function formatExpireTime(expireTime) {
  const expireDate = tzDayjs(expireTime);
  const currentYear = dayjs().year();
  const expireYear = expireDate.year();

  if (settingStore.lang === 'en') {

    return expireYear === currentYear
        ? expireDate.format('MMM D')
        : expireDate.format('MMM D, YYYY');

  } else {

    return expireYear === currentYear
        ? expireDate.format('M月D日')
        : expireDate.format('YYYY年M月D日');

  }
}

function refresh() {
  params.code = null
  params.status = null
  getList(true)
}

function getKeyStatus(item) {
  const now = new Date()
  const expireTime = new Date(item.expireTime)

  if (expireTime < now) {
    return 'expired'
  }

  if (item.count === 0) {
    return 'used'
  }

  if (item.originalCount && item.count < item.originalCount) {
    return 'partial'
  }

  return 'unused'
}

function openBatchGen() {
  showBatchGen.value = true
}

function submitBatch() {
  if (!batchForm.count || batchForm.count < 1 || batchForm.count > 1000) {
    ElMessage({
      message: t('countRange'),
      type: "error",
      plain: true
    })
    return
  }

  if (!batchForm.roleId) {
    ElMessage({
      message: t('emptyRole'),
      type: "error",
      plain: true
    })
    return
  }

  if (!batchForm.expireTime) {
    ElMessage({
      message: t('emptyTimeMsg'),
      type: "error",
      plain: true
    })
    return
  }

  batchLoading.value = true

  // 批量生成密钥
  const promises = []
  for (let i = 0; i < batchForm.count; i++) {
    const keyData = {
      code: generateRandomCode(),
      count: 1, // 默认只能使用一次
      roleId: batchForm.roleId,
      expireTime: batchForm.expireTime
    }
    promises.push(regKeyAdd(keyData))
  }

  Promise.all(promises).then(() => {
    showBatchGen.value = false
    ElMessage({
      message: t('generateSuccess'),
      type: "success",
      plain: true
    })
    getList()
  }).catch(() => {
    ElMessage({
      message: t('reqFailErrorMsg'),
      type: "error",
      plain: true
    })
  }).finally(() => {
    batchLoading.value = false
  })
}

function exportKeys() {
  if (regKeyData.length === 0) {
    ElMessage({
      message: t('noCodeFound'),
      type: "warning",
      plain: true
    })
    return
  }

  // 生成纯文本内容，每行一个密钥
  let textContent = regKeyData.map(item => item.code).join('\n')

  // 复制到剪贴板
  navigator.clipboard.writeText(textContent).then(() => {
    ElMessage({
      message: t('copySuccessMsg'),
      type: "success",
      plain: true
    })
  }).catch(() => {
    ElMessage({
      message: t('copyFailMsg'),
      type: "error",
      plain: true
    })
  })
}

function toggleSelect(item) {
  if (selectedKeys.has(item.regKeyId)) {
    selectedKeys.delete(item.regKeyId)
  } else {
    selectedKeys.add(item.regKeyId)
  }
}

function selectAll() {
  regKeyData.forEach(item => selectedKeys.add(item.regKeyId))
}

function deselectAll() {
  selectedKeys.clear()
}

function exportSelected() {
  if (selectedKeys.size === 0) {
    ElMessage({
      message: t('pleaseEnterCount'),
      type: "warning",
      plain: true
    })
    return
  }

  const selected = regKeyData.filter(item => selectedKeys.has(item.regKeyId))
  const textContent = selected.map(item => item.code).join('\n')

  navigator.clipboard.writeText(textContent).then(() => {
    ElMessage({
      message: t('copySuccessMsg') + ' (' + selectedKeys.size + ')',
      type: "success",
      plain: true
    })
  }).catch(() => {
    ElMessage({
      message: t('copyFailMsg'),
      type: "error",
      plain: true
    })
  })
}

function deleteSelected() {
  if (selectedKeys.size === 0) {
    ElMessage({
      message: t('pleaseEnterCount'),
      type: "warning",
      plain: true
    })
    return
  }

  ElMessageBox.confirm(t('delConfirm', {msg: selectedKeys.size + ' ' + t('regKey')}), {
    confirmButtonText: t('confirm'),
    cancelButtonText: t('cancel'),
    type: 'warning'
  }).then(() => {
    const ids = Array.from(selectedKeys)
    regKeyDelete(ids).then(() => {
      selectedKeys.clear()
      getList()
      ElMessage({
        message: t('delSuccessMsg'),
        type: "success",
        plain: true
      })
    })
  })
}

function search() {
  getList(true)
}

function getList(showLoading = false) {
  if (showLoading) {
    regKeyLoading.value = true
  }
  regKeyList(params).then(list => {
    regKeyData.length = 0
    regKeyData.push(...list)
    regKeyLoading.value = false
    setTimeout(() => {
      regKeyFirst.value = false
    },200)
  })
}

async function copyCode(code) {
  try {
    await navigator.clipboard.writeText(code);
    ElMessage({
      message: t('copySuccessMsg'),
      type: 'success',
      plain: true,
    })
  } catch (err) {
    console.error('复制失败:', err);
    ElMessage({
      message: '复制失败',
      type: 'error',
      plain: true,
    })
  }
}

function genCode() {
  addForm.code = generateRandomCode()
}

function generateRandomCode(length = 8) {
  const chars = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789';
  let result = '';
  for (let i = 0; i < length; i++) {
    result += chars.charAt(Math.floor(Math.random() * chars.length));
  }
  return result;
}

function clearNotUse() {
  ElMessageBox.confirm(t('clearRegKey'), {
    confirmButtonText: t('confirm'),
    cancelButtonText: t('cancel'),
    type: 'warning'
  }).then(() => {
    regKeyClearNotUse().then(() => {
      ElMessage({
        message: t('clearSuccess'),
        type: 'success',
        plain: true,
      })
      getList()
    })
  });
}

function submit() {

  if (!addForm.code) {
    ElMessage({
      message: $('emptyRegKeyMsg'),
      type: "error",
      plain: true
    })
    return
  }

  if (!addForm.roleId) {
    ElMessage({
      message: t('emptyRole'),
      type: "error",
      plain: true
    })
    return
  }

  if (!addForm.expireTime) {
    ElMessage({
      message: t('emptyTimeMsg'),
      type: "error",
      plain: true
    })
    return
  }

  if (!addForm.count) {
    ElMessage({
      message: t('emptyCountMsg'),
      type: "error",
      plain: true
    })
    return
  }

  addLoading.value = true
  regKeyAdd(addForm).then(() => {
    showAdd.value = false
    resetForm()
    ElMessage({
      message: t('addSuccessMsg'),
      type: "success",
      plain: true
    })
    getList()
  }).finally(() => {
    addLoading.value = false
  })
}

function deleteRegKey(regKey) {
  ElMessageBox.confirm(t('delConfirm', {msg: regKey.code}), {
    confirmButtonText: t('confirm'),
    cancelButtonText: t('cancel'),
    type: 'warning'
  }).then(() => {
    regKeyDelete([regKey.regKeyId]).then(() => {
      getList()
      ElMessage({
        message: t('delSuccessMsg'),
        type: "success",
        plain: true
      })
    })
  });
}

function resetForm() {
  addForm.code = ''
}

function openAdd() {
  genCode()
  showAdd.value = true
}

</script>

<style scoped lang="scss">
.reg-key {
  height: 100%;
  overflow: hidden;
}

.batch-actions {
  padding: 10px 15px;
  background: #ecf5ff;
  border-bottom: 1px solid var(--el-border-color-light);
  display: flex;
  align-items: center;
  gap: 15px;

  .selected-count {
    color: var(--el-color-primary);
    font-weight: 500;
  }
}

.scrollbar {
  height: calc(100% - 48px);
  position: relative;
  background: var(--extra-light-fill);
  @media (max-width: 372px) {
    height: calc(100% - 85px);
  }

  .code-box {
    padding: 15px 15px 25px 15px;
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(320px, 1fr));
    gap: 15px;

    .code-item {
      background: var(--el-bg-color);
      border-radius: 8px;
      border: 1px solid var(--el-border-color);
      transition: all 200ms;
      padding: 15px;
      position: relative;

      &:hover {
        box-shadow: 0 2px 12px rgba(0,0,0,0.1);
      }

      &.selected {
        border-color: var(--primary-color, #409eff);
        background: #ecf5ff;
      }

      .checkbox-wrapper {
        position: absolute;
        top: 15px;
        left: 15px;
      }

      .code-info {
        display: flex;
        padding-left: 30px;

        .info-left {
          flex: 1;
          min-width: 0;

          .info-left-item {
            display: flex;
            padding-top: 5px;

            .code {
              font-weight: bold;;
              font-size: 16px;
              white-space: nowrap;
              overflow: hidden;
              text-overflow: ellipsis;
              cursor: pointer;
            }
          }

          .info-left-item:first-child {
            padding-top: 0;
          }
        }

        .info-right {
          display: flex;
          flex-direction: column;
          padding-top: 2px;
          gap: 5px;
        }
      }
    }
  }
}

.empty {
  display: flex;
  justify-content: center;
  align-items: center;
  height: 100%;
}

:deep(.history-list.el-dialog) {
  min-height: 300px;
  width: 500px !important;
  @media (max-width: 540px) {
    width: calc(100% - 40px) !important;
    margin-right: 20px !important;
    margin-left: 20px !important;
  }
}

.history-list .loading {
  position: absolute;
  top: 10px;
  z-index: 0;
  background: rgba(255, 255, 255, 0);
}

:deep(.history-list .el-dialog__header) {
  padding-bottom: 5px;
}

:deep(.el-scrollbar__view) {
  height: calc(100% - 80px);
}

.loading {
  position: absolute;
  left: 0;
  top: 0;
  width: 100%;
  height: 100%;
  display: flex;
  justify-content: center;
  align-items: center;
  background: var(--loadding-background);
  z-index: 2;
}

.loading-show {
  transition: all 200ms ease 200ms;
  opacity: 1;
}

.loading-hide {
  pointer-events: none;
  transition: var(--loading-hide-transition);
  opacity: 0;
}

.container {
  display: grid;
  grid-template-columns: 1fr;
  gap: 15px;
}

:deep(.el-dialog) {
  width: 400px !important;
  @media (max-width: 440px) {
    width: calc(100% - 40px) !important;
    margin-right: 20px !important;
    margin-left: 20px !important;
  }
}

.setting {
  cursor: pointer;
}

.gen-code {
  color: #606266;
  cursor: pointer;
}

.header-actions {
  padding: 9px 15px;
  display: flex;
  gap: 18px;
  flex-wrap: wrap;
  align-items: center;
  box-shadow: inset 0 -1px 0 0 rgba(100, 121, 143, 0.12);
  font-size: 18px;
  @media (max-width: 767px) {
    gap: 15px;
  }

  .search-input {
    width: min(200px, calc(100vw - 140px));
  }

  .status-select {
    width: 120px;
    :deep(.el-input__inner) {
      height: 28px;
    }
  }

  .search {
    :deep(.el-input-group) {
      height: 28px;
    }

    :deep(.el-input__inner) {
      height: 28px;
    }
  }

  .icon {
    cursor: pointer;
  }
}

:deep(.el-table__inner-wrapper:before) {
  background: var(--el-bg-color);
}

</style>
