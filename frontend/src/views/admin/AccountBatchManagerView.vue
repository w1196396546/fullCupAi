<template>
  <AppLayout>
    <TablePageLayout>
      <template #actions>
        <div class="grid gap-4 xl:grid-cols-4">
          <div class="card p-4 xl:col-span-4">
            <div class="flex flex-wrap items-center justify-between gap-3">
              <div>
                <h2 class="text-lg font-semibold text-gray-900 dark:text-white">
                  {{ t('admin.accounts.batchManager.title') }}
                </h2>
                <p class="mt-1 text-sm text-gray-500 dark:text-gray-400">
                  {{ t('admin.accounts.batchManager.description') }}
                </p>
              </div>
              <div class="flex flex-wrap items-center gap-2 text-sm">
                <span class="rounded-full bg-primary-50 px-3 py-1 font-medium text-primary-700 dark:bg-primary-900/20 dark:text-primary-300">
                  {{ t('admin.accounts.batchManager.selectionInfo', { count: selectedIds.length }) }}
                </span>
                <button class="btn btn-secondary btn-sm" @click="selectCurrentPage">
                  {{ t('admin.accounts.batchManager.selectCurrentPage') }}
                </button>
                <button class="btn btn-secondary btn-sm" :disabled="selectedIds.length === 0" @click="clearSelection">
                  {{ t('admin.accounts.batchManager.clearSelection') }}
                </button>
                <button class="btn btn-secondary btn-sm" :disabled="loading" @click="reload">
                  {{ t('admin.accounts.batchManager.reload') }}
                </button>
              </div>
            </div>
          </div>

          <div class="card p-4">
            <div class="mb-3">
              <h3 class="text-sm font-semibold text-gray-900 dark:text-white">
                {{ t('admin.accounts.batchManager.notesTitle') }}
              </h3>
              <p class="mt-1 text-xs text-gray-500 dark:text-gray-400">
                {{ t('admin.accounts.batchManager.notesHelp') }}
              </p>
            </div>
            <textarea
              v-model="notesForm.notes"
              class="input min-h-[96px] w-full"
              :placeholder="t('admin.accounts.batchManager.notesPlaceholder')"
            />
            <button
              class="btn btn-primary mt-3 w-full"
              :disabled="selectedIds.length === 0 || notesSubmitting"
              @click="applyNotes"
            >
              {{ notesSubmitting ? t('common.loading') : t('admin.accounts.batchManager.applyNotes') }}
            </button>
          </div>

          <div class="card p-4">
            <div class="mb-3">
              <h3 class="text-sm font-semibold text-gray-900 dark:text-white">
                {{ t('admin.accounts.batchManager.stateTitle') }}
              </h3>
              <p class="mt-1 text-xs text-gray-500 dark:text-gray-400">
                {{ t('admin.accounts.batchManager.stateHelp') }}
              </p>
            </div>
            <Select v-model="stateForm.state" :options="stateOptions" class="w-full" />
            <input
              v-model.trim="stateForm.reason"
              type="text"
              class="input mt-3 w-full"
              :placeholder="t('admin.accounts.batchManager.reasonPlaceholder')"
            />
            <input
              v-if="requiresDuration"
              v-model.number="stateForm.duration_minutes"
              type="number"
              min="1"
              class="input mt-3 w-full"
              :placeholder="t('admin.accounts.batchManager.durationMinutes')"
            />
            <button
              class="btn btn-primary mt-3 w-full"
              :disabled="selectedIds.length === 0 || stateSubmitting"
              @click="applyState"
            >
              {{ stateSubmitting ? t('common.loading') : t('admin.accounts.batchManager.applyState') }}
            </button>
          </div>

          <div class="card p-4">
            <div class="mb-3">
              <h3 class="text-sm font-semibold text-gray-900 dark:text-white">
                {{ t('admin.accounts.batchManager.scheduleTitle') }}
              </h3>
              <p class="mt-1 text-xs text-gray-500 dark:text-gray-400">
                {{ t('admin.accounts.batchManager.scheduleHelp') }}
              </p>
            </div>
            <input
              v-model.trim="scheduleForm.model_id"
              type="text"
              class="input w-full"
              :placeholder="t('admin.accounts.batchManager.modelPlaceholder')"
            />
            <input
              v-model.trim="scheduleForm.cron_expression"
              type="text"
              class="input mt-3 w-full"
              :placeholder="t('admin.accounts.batchManager.cronPlaceholder')"
            />
            <input
              v-model.number="scheduleForm.max_results"
              type="number"
              min="1"
              class="input mt-3 w-full"
              placeholder="20"
            />
            <label class="mt-3 flex items-center gap-2 text-sm text-gray-600 dark:text-gray-300">
              <input v-model="scheduleForm.enabled" type="checkbox" class="rounded border-gray-300 text-primary-600 focus:ring-primary-500" />
              {{ t('admin.scheduledTests.enabled') }}
            </label>
            <label class="mt-2 flex items-center gap-2 text-sm text-gray-600 dark:text-gray-300">
              <input v-model="scheduleForm.auto_recover" type="checkbox" class="rounded border-gray-300 text-primary-600 focus:ring-primary-500" />
              {{ t('admin.scheduledTests.autoRecover') }}
            </label>
            <button
              class="btn btn-primary mt-3 w-full"
              :disabled="selectedIds.length === 0 || scheduleSubmitting || !scheduleForm.cron_expression"
              @click="applySchedule"
            >
              {{ scheduleSubmitting ? t('common.loading') : t('admin.accounts.batchManager.applySchedule') }}
            </button>
          </div>

          <div class="card p-4">
            <div class="mb-3">
              <h3 class="text-sm font-semibold text-gray-900 dark:text-white">
                {{ t('admin.accounts.batchManager.availabilityTitle') }}
              </h3>
              <p class="mt-1 text-xs text-gray-500 dark:text-gray-400">
                {{ t('admin.accounts.batchManager.availabilityHelp') }}
              </p>
            </div>
            <div class="rounded-lg bg-gray-50 p-3 text-sm text-gray-600 dark:bg-dark-700/50 dark:text-gray-300">
              <div>{{ t('admin.accounts.batchManager.checkingProgress', { done: checkProgress.done, total: checkProgress.total }) }}</div>
              <div v-if="lastCheckSummary" class="mt-2 text-xs text-gray-500 dark:text-gray-400">
                {{ lastCheckSummary }}
              </div>
            </div>
            <button
              class="btn btn-danger mt-3 w-full"
              :disabled="selectedIds.length === 0 || availabilityChecking"
              @click="runAvailabilityCheck"
            >
              {{ availabilityChecking ? t('common.loading') : t('admin.accounts.batchManager.runAvailabilityCheck') }}
            </button>
          </div>
        </div>
      </template>

      <template #filters>
        <div class="card p-4">
          <div class="grid gap-3 lg:grid-cols-6">
            <SearchInput
              v-model="filters.search"
              class="lg:col-span-2"
              :placeholder="t('admin.accounts.searchAccounts')"
              @search="handleFilterChange"
            />
            <Select v-model="filters.platform" :options="platformOptions" @change="handleFilterChange" />
            <Select v-model="filters.type" :options="typeOptions" @change="handleFilterChange" />
            <Select v-model="filters.status" :options="statusOptions" @change="handleFilterChange" />
            <div class="grid grid-cols-2 gap-3 lg:col-span-2">
              <input v-model="filters.created_from" type="date" class="input" @change="handleFilterChange" />
              <input v-model="filters.created_to" type="date" class="input" @change="handleFilterChange" />
            </div>
          </div>
        </div>
      </template>

      <template #table>
        <div class="overflow-x-auto">
          <table class="min-w-full divide-y divide-gray-200 dark:divide-dark-700">
            <thead class="bg-gray-50 dark:bg-dark-800">
              <tr>
                <th class="px-4 py-3 text-left">
                  <input
                    type="checkbox"
                    class="rounded border-gray-300 text-primary-600 focus:ring-primary-500"
                    :checked="allVisibleSelected"
                    @change="toggleSelectAllVisible($event)"
                  />
                </th>
                <th class="px-4 py-3 text-left text-sm font-medium text-gray-600 dark:text-gray-300">{{ t('admin.accounts.columns.name') }}</th>
                <th class="px-4 py-3 text-left text-sm font-medium text-gray-600 dark:text-gray-300">{{ t('admin.accounts.columns.platformType') }}</th>
                <th class="px-4 py-3 text-left text-sm font-medium text-gray-600 dark:text-gray-300">{{ t('admin.accounts.columns.status') }}</th>
                <th class="px-4 py-3 text-left text-sm font-medium text-gray-600 dark:text-gray-300">{{ t('admin.accounts.columns.notes') }}</th>
                <th class="px-4 py-3 text-left text-sm font-medium text-gray-600 dark:text-gray-300">{{ t('admin.accounts.batchManager.columns.createdAt') }}</th>
                <th class="px-4 py-3 text-left text-sm font-medium text-gray-600 dark:text-gray-300">{{ t('admin.accounts.columns.lastUsed') }}</th>
              </tr>
            </thead>
            <tbody v-if="accounts.length > 0" class="divide-y divide-gray-100 bg-white dark:divide-dark-800 dark:bg-dark-900">
              <tr v-for="account in accounts" :key="account.id" class="hover:bg-gray-50 dark:hover:bg-dark-800/60">
                <td class="px-4 py-3">
                  <input
                    type="checkbox"
                    class="rounded border-gray-300 text-primary-600 focus:ring-primary-500"
                    :checked="selectedIds.includes(account.id)"
                    @change="toggleSelected(account.id)"
                  />
                </td>
                <td class="px-4 py-3">
                  <div class="font-medium text-gray-900 dark:text-white">{{ account.name }}</div>
                  <div class="mt-1 text-xs text-gray-500 dark:text-gray-400">ID: {{ account.id }}</div>
                </td>
                <td class="px-4 py-3 text-sm text-gray-600 dark:text-gray-300">
                  {{ account.platform }} / {{ account.type }}
                </td>
                <td class="px-4 py-3">
                  <AccountStatusIndicator :account="account" />
                </td>
                <td class="px-4 py-3 text-sm text-gray-600 dark:text-gray-300">
                  <span v-if="account.notes" class="block max-w-xs truncate" :title="account.notes">{{ account.notes }}</span>
                  <span v-else class="text-gray-400 dark:text-gray-500">-</span>
                </td>
                <td class="px-4 py-3 text-sm text-gray-600 dark:text-gray-300">
                  <div>{{ formatDateTime(account.created_at) }}</div>
                  <div class="mt-1 text-xs text-gray-500 dark:text-gray-400">{{ formatRelativeTime(account.created_at) }}</div>
                </td>
                <td class="px-4 py-3 text-sm text-gray-600 dark:text-gray-300">
                  {{ formatRelativeTime(account.last_used_at) }}
                </td>
              </tr>
            </tbody>
            <tbody v-else>
              <tr>
                <td colspan="7" class="px-4 py-10 text-center text-sm text-gray-500 dark:text-gray-400">
                  {{ loading ? t('common.loading') : t('admin.accounts.batchManager.tableEmpty') }}
                </td>
              </tr>
            </tbody>
          </table>
        </div>
      </template>

      <template #pagination>
        <Pagination
          v-if="pagination.total > 0"
          :page="pagination.page"
          :total="pagination.total"
          :page-size="pagination.page_size"
          @update:page="handlePageChange"
          @update:pageSize="handlePageSizeChange"
        />
      </template>
    </TablePageLayout>
  </AppLayout>
</template>

<script setup lang="ts">
import { computed, onMounted, reactive, ref } from 'vue'
import { useI18n } from 'vue-i18n'
import { getLocale } from '@/i18n'
import AppLayout from '@/components/layout/AppLayout.vue'
import TablePageLayout from '@/components/layout/TablePageLayout.vue'
import Pagination from '@/components/common/Pagination.vue'
import SearchInput from '@/components/common/SearchInput.vue'
import Select from '@/components/common/Select.vue'
import AccountStatusIndicator from '@/components/account/AccountStatusIndicator.vue'
import { adminAPI } from '@/api/admin'
import type { Account, SelectOption } from '@/types'
import { useAppStore } from '@/stores'
import { formatDateTime, formatRelativeTime } from '@/utils/format'

type BatchState = 'normal' | 'active' | 'inactive' | 'error' | 'rate_limited' | 'temp_unschedulable' | 'overloaded'

const { t } = useI18n()
const appStore = useAppStore()

const loading = ref(false)
const notesSubmitting = ref(false)
const stateSubmitting = ref(false)
const scheduleSubmitting = ref(false)
const availabilityChecking = ref(false)
const lastCheckSummary = ref('')
const accounts = ref<Account[]>([])
const selectedIds = ref<number[]>([])
const checkProgress = reactive({ done: 0, total: 0 })

const pagination = reactive({
  page: 1,
  page_size: 20,
  total: 0
})

const filters = reactive({
  search: '',
  platform: '',
  type: '',
  status: '',
  created_from: '',
  created_to: ''
})

const notesForm = reactive({
  notes: ''
})

const stateForm = reactive<{ state: BatchState; reason: string; duration_minutes: number }>({
  state: 'normal',
  reason: '',
  duration_minutes: 60
})

const scheduleForm = reactive({
  model_id: '',
  cron_expression: '0 * * * *',
  max_results: 20,
  enabled: true,
  auto_recover: true
})

const platformOptions = computed<SelectOption[]>(() => [
  { value: '', label: t('admin.accounts.allPlatforms') },
  { value: 'anthropic', label: 'Anthropic' },
  { value: 'openai', label: 'OpenAI' },
  { value: 'gemini', label: 'Gemini' },
  { value: 'antigravity', label: 'Antigravity' },
  { value: 'sora', label: 'Sora' }
])

const typeOptions = computed<SelectOption[]>(() => [
  { value: '', label: t('admin.accounts.allTypes') },
  { value: 'oauth', label: t('admin.accounts.oauthType') },
  { value: 'setup-token', label: t('admin.accounts.setupToken') },
  { value: 'apikey', label: t('admin.accounts.apiKey') },
  { value: 'upstream', label: t('admin.accounts.types.upstream') },
  { value: 'bedrock', label: 'AWS Bedrock' }
])

const statusOptions = computed<SelectOption[]>(() => [
  { value: '', label: t('admin.accounts.allStatus') },
  { value: 'active', label: t('admin.accounts.status.active') },
  { value: 'inactive', label: t('admin.accounts.status.inactive') },
  { value: 'error', label: t('admin.accounts.status.error') },
  { value: 'rate_limited', label: t('admin.accounts.status.rateLimited') },
  { value: 'temp_unschedulable', label: t('admin.accounts.status.tempUnschedulable') },
  { value: 'overloaded', label: t('admin.accounts.status.overloaded') }
])

const stateOptions = computed<SelectOption[]>(() => [
  { value: 'normal', label: t('admin.accounts.status.active') },
  { value: 'inactive', label: t('admin.accounts.status.inactive') },
  { value: 'error', label: t('admin.accounts.status.error') },
  { value: 'rate_limited', label: t('admin.accounts.status.rateLimited') },
  { value: 'temp_unschedulable', label: t('admin.accounts.status.tempUnschedulable') },
  { value: 'overloaded', label: t('admin.accounts.status.overloaded') }
])

const requiresDuration = computed(() => ['rate_limited', 'temp_unschedulable', 'overloaded'].includes(stateForm.state))
const allVisibleSelected = computed(() => accounts.value.length > 0 && accounts.value.every(account => selectedIds.value.includes(account.id)))

function buildFilters() {
  return {
    platform: filters.platform || undefined,
    type: filters.type || undefined,
    status: filters.status || undefined,
    search: filters.search || undefined,
    created_from: filters.created_from || undefined,
    created_to: filters.created_to || undefined
  }
}

async function loadAccounts() {
  loading.value = true
  try {
    const data = await adminAPI.accounts.list(pagination.page, pagination.page_size, buildFilters())
    accounts.value = data.items ?? []
    pagination.total = data.total
    pagination.page = data.page
    pagination.page_size = data.page_size
  } catch (error) {
    console.error('Failed to load batch manager accounts:', error)
    appStore.showError(t('admin.accounts.failedToLoad'))
  } finally {
    loading.value = false
  }
}

function reload() {
  void loadAccounts()
}

function handleFilterChange() {
  pagination.page = 1
  void loadAccounts()
}

function handlePageChange(page: number) {
  pagination.page = page
  void loadAccounts()
}

function handlePageSizeChange(pageSize: number) {
  pagination.page = 1
  pagination.page_size = pageSize
  void loadAccounts()
}

function toggleSelected(accountId: number) {
  if (selectedIds.value.includes(accountId)) {
    selectedIds.value = selectedIds.value.filter(id => id !== accountId)
    return
  }
  selectedIds.value = [...selectedIds.value, accountId]
}

function clearSelection() {
  selectedIds.value = []
}

function selectCurrentPage() {
  const next = new Set(selectedIds.value)
  for (const account of accounts.value) {
    next.add(account.id)
  }
  selectedIds.value = Array.from(next)
}

function toggleSelectAllVisible(event: Event) {
  const checked = (event.target as HTMLInputElement).checked
  if (checked) {
    selectCurrentPage()
    return
  }
  const visibleIds = new Set(accounts.value.map(account => account.id))
  selectedIds.value = selectedIds.value.filter(id => !visibleIds.has(id))
}

async function applyNotes() {
  if (selectedIds.value.length === 0) return
  notesSubmitting.value = true
  try {
    const result = await adminAPI.accounts.bulkUpdate(selectedIds.value, { notes: notesForm.notes })
    appStore.showSuccess(t('admin.accounts.batchManager.notesApplied', { success: result.success, failed: result.failed }))
    await loadAccounts()
  } catch (error) {
    console.error('Failed to apply notes:', error)
    appStore.showError(t('admin.accounts.bulkEdit.failed'))
  } finally {
    notesSubmitting.value = false
  }
}

async function applyState() {
  if (selectedIds.value.length === 0) return
  stateSubmitting.value = true
  try {
    const payload: {
      account_ids: number[]
      state: BatchState
      reason?: string
      duration_minutes?: number
    } = {
      account_ids: selectedIds.value,
      state: stateForm.state
    }
    if (stateForm.reason) payload.reason = stateForm.reason
    if (requiresDuration.value) payload.duration_minutes = stateForm.duration_minutes
    const result = await adminAPI.accounts.batchSetState(payload)
    appStore.showSuccess(t('admin.accounts.batchManager.stateApplied', { success: result.success, failed: result.failed }))
    await loadAccounts()
  } catch (error) {
    console.error('Failed to apply state:', error)
    appStore.showError(t('admin.accounts.bulkEdit.failed'))
  } finally {
    stateSubmitting.value = false
  }
}

async function runWithConcurrency<T>(items: number[], limit: number, worker: (id: number) => Promise<T>): Promise<T[]> {
  const results: T[] = []
  let index = 0

  const runners = Array.from({ length: Math.min(limit, items.length) }, async () => {
    while (index < items.length) {
      const current = items[index]
      index += 1
      results.push(await worker(current))
    }
  })

  await Promise.all(runners)
  return results
}

async function applySchedule() {
  if (selectedIds.value.length === 0 || !scheduleForm.cron_expression.trim()) return
  scheduleSubmitting.value = true
  try {
    let success = 0
    let failed = 0
    await runWithConcurrency(selectedIds.value, 5, async (accountId) => {
      try {
        const existingPlans = await adminAPI.scheduledTests.listByAccount(accountId)
        const payload = {
          model_id: scheduleForm.model_id,
          cron_expression: scheduleForm.cron_expression,
          enabled: scheduleForm.enabled,
          max_results: scheduleForm.max_results,
          auto_recover: scheduleForm.auto_recover
        }
        if (existingPlans.length > 0) {
          await adminAPI.scheduledTests.update(existingPlans[0].id, payload)
        } else {
          await adminAPI.scheduledTests.create({
            account_id: accountId,
            ...payload
          })
        }
        success += 1
      } catch (error) {
        console.error('Failed to apply schedule:', accountId, error)
        failed += 1
      }
      return null
    })
    appStore.showSuccess(t('admin.accounts.batchManager.scheduleApplied', { success, failed }))
  } finally {
    scheduleSubmitting.value = false
  }
}

async function testAccountAvailability(accountId: number): Promise<boolean> {
  const response = await fetch(`/api/v1/admin/accounts/${accountId}/test`, {
    method: 'POST',
    headers: {
      Authorization: `Bearer ${localStorage.getItem('auth_token') ?? ''}`,
      'Content-Type': 'application/json',
      'Accept-Language': getLocale()
    },
    body: JSON.stringify({})
  })

  if (!response.ok) {
    throw new Error(`HTTP ${response.status}`)
  }

  const reader = response.body?.getReader()
  if (!reader) {
    throw new Error('No response body')
  }

  const decoder = new TextDecoder()
  let buffer = ''
  let success = false

  while (true) {
    const { done, value } = await reader.read()
    if (done) break
    buffer += decoder.decode(value, { stream: true })
    const lines = buffer.split('\n')
    buffer = lines.pop() || ''

    for (const line of lines) {
      if (!line.startsWith('data: ')) continue
      const payload = line.slice(6).trim()
      if (!payload) continue
      const event = JSON.parse(payload) as { type?: string; success?: boolean }
      if (event.type === 'test_complete') {
        success = Boolean(event.success)
      }
      if (event.type === 'error') {
        success = false
      }
    }
  }

  return success
}

async function runAvailabilityCheck() {
  if (selectedIds.value.length === 0) return
  availabilityChecking.value = true
  checkProgress.total = selectedIds.value.length
  checkProgress.done = 0
  lastCheckSummary.value = ''

  try {
    const failedIds: number[] = []

    await runWithConcurrency(selectedIds.value, 3, async (accountId) => {
      try {
        const success = await testAccountAvailability(accountId)
        if (!success) {
          failedIds.push(accountId)
        }
      } catch (error) {
        console.error('Availability check failed:', accountId, error)
        failedIds.push(accountId)
      } finally {
        checkProgress.done += 1
      }
      return null
    })

    if (failedIds.length > 0) {
      await adminAPI.accounts.batchSetState({
        account_ids: failedIds,
        state: 'error',
        reason: 'Marked as error by batch availability check'
      })
      lastCheckSummary.value = t('admin.accounts.batchManager.checkCompletedWithErrors', {
        total: selectedIds.value.length,
        failed: failedIds.length
      })
      appStore.showWarning(t('admin.accounts.batchManager.checkFailedAccountsMarkedError', { count: failedIds.length }))
    } else {
      lastCheckSummary.value = t('admin.accounts.batchManager.checkCompleted', { total: selectedIds.value.length })
      appStore.showSuccess(lastCheckSummary.value)
    }

    await loadAccounts()
  } catch (error) {
    console.error('Failed to run availability check:', error)
    appStore.showError(t('admin.accounts.batchManager.checkRequestFailed'))
  } finally {
    availabilityChecking.value = false
  }
}

onMounted(() => {
  void loadAccounts()
})
</script>
