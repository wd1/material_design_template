<script setup>
import { useInvoiceStore } from '@/views/apps/invoice/useInvoiceStore'
import { avatarText } from '@core/utils/formatters'

const invoiceListStore = useInvoiceStore()
const searchQuery = ref('')
const dateRange = ref('')
const selectedStatus = ref()
const rowPerPage = ref(10)
const currentPage = ref(1)
const totalPage = ref(1)
const totalInvoices = ref(0)
const invoices = ref([])
const selectedRows = ref([])
const selectAllInvoice = ref(false)

// 👉 Fetch Invoices
watchEffect(() => {
  const [start, end] = dateRange.value ? dateRange.value.split('to') : ''

  invoiceListStore.fetchInvoices({
    q: searchQuery.value,
    status: selectedStatus.value,
    perPage: rowPerPage.value,
    currentPage: currentPage.value,
    startDate: start,
    endDate: end,
  }).then(response => {
    invoices.value = response.data.invoices
    totalPage.value = response.data.totalPage
    totalInvoices.value = response.data.totalInvoices
  }).catch(error => {
    console.log(error)
  })
})

// 👉 Fetch Invoices
watchEffect(() => {
  if (currentPage.value > totalPage.value)
    currentPage.value = totalPage.value
})

// 👉 Computing pagination data
const paginationData = computed(() => {
  const firstIndex = invoices.value.length ? (currentPage.value - 1) * rowPerPage.value + 1 : 0
  const lastIndex = invoices.value.length + (currentPage.value - 1) * rowPerPage.value
  
  return `${ firstIndex }-${ lastIndex } of ${ totalInvoices.value }`
})

// 👉 Invoice balance variant resolver
const resolveInvoiceBalanceVariant = (balance, total) => {
  if (balance === total)
    return {
      status: 'Unpaid',
      chip: { color: 'error' },
    }
  if (balance === 0)
    return {
      status: 'Paid',
      chip: { color: 'success' },
    }
  
  return {
    status: balance,
    chip: { variant: 'text' },
  }
}

const resolveInvoiceStatusVariantAndIcon = status => {
  if (status === 'Partial Payment')
    return {
      variant: 'warning',
      icon: 'mdi-chart-timeline-variant',
    }
  if (status === 'Paid')
    return {
      variant: 'success',
      icon: 'mdi-check',
    }
  if (status === 'Downloaded')
    return {
      variant: 'info',
      icon: 'mdi-arrow-down',
    }
  if (status === 'Draft')
    return {
      variant: 'secondary',
      icon: 'mdi-content-save-outline',
    }
  if (status === 'Sent')
    return {
      variant: 'primary',
      icon: 'mdi-email-outline',
    }
  if (status === 'Past Due')
    return {
      variant: 'error',
      icon: 'mdi-alert-circle-outline',
    }
  
  return {
    variant: 'secondary',
    icon: 'mdi-close',
  }
}

// 👉 Add/Remove all checkbox ids in/from array
const selectUnselectAll = () => {
  selectAllInvoice.value = !selectAllInvoice.value
  if (selectAllInvoice.value) {
    invoices.value.forEach(invoice => {
      if (!selectedRows.value.includes(`check${ invoice.id }`))
        selectedRows.value.push(`check${ invoice.id }`)
    })
  } else {
    selectedRows.value = []
  }
}

// 👉 watch if checkbox array is empty all checkbox should be uncheck
watch(selectedRows, () => {
  if (!selectedRows.value.length)
    selectAllInvoice.value = false
}, { deep: true })

const addRemoveIndividualCheckbox = checkID => {
  if (selectedRows.value.includes(checkID)) {
    const index = selectedRows.value.indexOf(checkID)

    selectedRows.value.splice(index, 1)
  } else {
    selectedRows.value.push(checkID)
    selectAllInvoice.value = true
  }
}

const computedMoreList = computed(() => {
  return paramId => [
    {
      title: 'Download',
      value: 'download',
      prependIcon: 'mdi-download-outline',
    },
    {
      title: 'Edit',
      value: 'edit',
      prependIcon: 'mdi-pencil-outline',
      to: {
        name: 'apps-invoice-edit-id',
        params: { id: paramId },
      },
    },
    {
      title: 'Duplicate',
      value: 'duplicate',
      prependIcon: 'mdi-layers-outline',
    },
  ]
})
</script>

<template>
  <section v-if="invoices">
    <!-- 👉 Invoice Filters  -->
    <VCard
      title="Filters"
      class="mb-6"
    >
      <VCardText>
        <VRow>
          <!-- 👉 Status filter -->
          <VCol
            cols="12"
            md="6"
          >
            <VSelect
              v-model="selectedStatus"
              label="Select Status"
              clearable
              clear-icon="mdi-close"
              :items="['Downloaded', 'Draft', 'Sent', 'Paid', 'Partial Payment', 'Past Due']"
            />
          </VCol>

          <!-- 👉 DateRange filter -->
          <VCol
            cols="12"
            md="6"
          >
            <AppDateTimePicker
              v-model="dateRange"
              label="Invoice Date"
              clear-icon="mdi-close"
              clearable
              :config="{ mode: 'range' }"
            />
          </VCol>
        </VRow>
      </VCardText>
    </VCard>

    <VCard id="invoice-list">
      <VCardText class="d-flex align-center flex-wrap gap-4">
        <!-- 👉 Actions  -->
        <div class="me-3">
          <VSelect
            density="compact"
            label="Actions"
            :items="['Delete', 'Edit', 'Send']"
            class="invoice-list-actions"
            :disabled="!selectedRows.length"
          />
        </div>

        <VSpacer />

        <div class="d-flex align-center flex-wrap gap-4">
          <!-- 👉 Search  -->
          <div class="invoice-list-search">
            <VTextField
              v-model="searchQuery"
              placeholder="Search Invoice"
              density="compact"
            />
          </div>

          <!-- 👉 Create invoice -->
          <VBtn
            prepend-icon="mdi-plus"
            :to="{ name: 'apps-invoice-add' }"
          >
            Create invoice
          </VBtn>
        </div>
      </VCardText>

      <VDivider />

      <!-- SECTION Table -->
      <VTable class="text-no-wrap table-header-bg rounded-0">
        <!-- 👉 Table head -->
        <thead>
          <tr>
            <!-- 👉 Check/Uncheck all checkbox -->
            <th
              scope="col"
              class="text-center"
            >
              <div style="width: 1rem;">
                <VCheckbox
                  :model-value="selectAllInvoice"
                  :indeterminate="(invoices.length !== selectedRows.length) && !!selectedRows.length"
                  @click="selectUnselectAll"
                />
              </div>
            </th>
            <th scope="col">
              #ID
            </th>
            <th
              scope="col"
              class="text-center"
            >
              <VIcon icon="mdi-trending-up" />
            </th>
            <th scope="col">
              CLIENT
            </th>
            <th
              scope="col"
              class="text-center"
            >
              TOTAL
            </th>
            <th scope="col">
              DATE
            </th>
            <th
              scope="col"
              class="text-center"
            >
              BALANCE
            </th>
            <th scope="col">
              ACTIONS
            </th>
          </tr>
        </thead>

        <!-- 👉 Table Body -->
        <tbody>
          <tr
            v-for="invoice in invoices"
            :key="invoice.id"
          >
            <!-- 👉 Individual checkbox -->
            <td>
              <div style="width: 1rem;">
                <VCheckbox
                  :id="`check${invoice.id}`"
                  :model-value="selectedRows.includes(`check${invoice.id}`)"
                  @click="addRemoveIndividualCheckbox(`check${invoice.id}`)"
                />
              </div>
            </td>

            <!-- 👉 Id -->
            <td>
              <RouterLink :to="{ name: 'apps-invoice-preview-id', params: { id: invoice.id } }">
                #{{ invoice.id }}
              </RouterLink>
            </td>

            <!-- 👉 Trending -->
            <td class="text-center">
              <VTooltip>
                <template #activator="{ props }">
                  <VAvatar
                    :size="34"
                    v-bind="props"
                    :color="resolveInvoiceStatusVariantAndIcon(invoice.invoiceStatus).variant"
                    variant="tonal"
                  >
                    <VIcon
                      :size="20"
                      :icon="resolveInvoiceStatusVariantAndIcon(invoice.invoiceStatus).icon"
                    />
                  </VAvatar>
                </template>
                <p class="mb-0">
                  {{ invoice.invoiceStatus }}
                </p>
                <p class="mb-0">
                  Balance: {{ invoice.balance }}
                </p>
                <p class="mb-0">
                  Due date: {{ invoice.dueDate }}
                </p>
              </VTooltip>
            </td>

            <!-- 👉 Client Avatar and Email -->
            <td>
              <div class="d-flex align-center">
                <VAvatar
                  size="34"
                  :color="resolveInvoiceStatusVariantAndIcon(invoice.invoiceStatus).variant"
                  variant="tonal"
                  class="me-3"
                >
                  <VImg
                    v-if="invoice.avatar.length"
                    :src="invoice.avatar"
                  />
                  <span
                    v-else
                    class="text-sm"
                  >{{ avatarText(invoice.client.name) }}</span>
                </VAvatar>
                <div class="d-flex flex-column">
                  <h6 class="text-sm font-weight-medium mb-0">
                    {{ invoice.client.name }}
                  </h6>
                  <span class="text-caption">{{ invoice.client.companyEmail }}</span>
                </div>
              </div>
            </td>

            <!-- 👉 total -->
            <td class="text-center">
              ${{ invoice.total }}
            </td>

            <!-- 👉 Date -->
            <td>
              {{ invoice.issuedDate }}
            </td>

            <!-- 👉 Balance -->
            <td class="text-center">
              <VChip
                v-bind="resolveInvoiceBalanceVariant(invoice.balance, invoice.total).chip"
                size="small"
                class="text-high-emphasis"
              >
                {{ resolveInvoiceBalanceVariant(invoice.balance, invoice.total).status }}
              </VChip>
            </td>

            <!-- 👉 Actions -->
            <td style="width: 8rem;">
              <IconBtn>
                <VIcon icon="mdi-delete-outline" />
              </IconBtn>

              <IconBtn :to="{ name: 'apps-invoice-preview-id', params: { id: invoice.id } }">
                <VIcon icon="mdi-eye-outline" />
              </IconBtn>

              <MoreBtn
                :menu-list="computedMoreList(invoice.id)"
                item-props
              />
            </td>
          </tr>
        </tbody>

        <!-- 👉 table footer  -->
        <tfoot v-show="!invoices.length">
          <tr>
            <td
              colspan="8"
              class="text-center text-body-1"
            >
              No data available
            </td>
          </tr>
        </tfoot>
      </VTable>
      <!-- !SECTION -->

      <VDivider />

      <!-- SECTION Pagination -->
      <VCardText class="d-flex flex-wrap justify-end gap-4 pa-2">
        <!-- 👉 Rows per page -->
        <div
          class="d-flex align-center me-3"
          style="width: 171px;"
        >
          <span class="text-no-wrap me-3">Rows per page:</span>
          <VSelect
            v-model="rowPerPage"
            density="compact"
            variant="plain"
            class="mt-n4"
            :items="[10, 20, 30, 50]"
          />
        </div>

        <!-- 👉 Pagination and pagination meta -->
        <div class="d-flex align-center">
          <h6 class="text-sm font-weight-regular">
            {{ paginationData }}
          </h6>

          <VPagination
            v-model="currentPage"
            size="small"
            :total-visible="1"
            :length="totalPage"
            @next="selectedRows = []"
            @prev="selectedRows = []"
          />
        </div>
      </VCardText>
      <!-- !SECTION -->
    </VCard>
  </section>
</template>

<style lang="scss">
#invoice-list {
  .invoice-list-actions {
    inline-size: 8rem;
  }

  .invoice-list-search {
    inline-size: 12rem;
  }
}
</style>
