<script setup>
const props = defineProps({ customerId: String })

const loans = ref([])
const filters = reactive({ status: 'all', search: '' })
const isLoading = ref(false)
const selectedLoan = ref(null)

async function openModal(loan) {
  selectedLoan.value = loan
  if (!loan.schedule) {
    loan.schedule = await $fetch(`/api/loans/${loan.id}/schedule`)
  }
}
const lastSyncedAt = ref(new Date().toLocaleTimeString())

const { data: customer } = await useFetch(`/api/customers/${props.customerId}`)
const { data: summary } = await useFetch(`/api/customers/${props.customerId}/summary`)
const { data: rates } = await useFetch('/api/rates')

async function loadLoans() {
  selectedLoan.value = null
  isLoading.value = true
  const res = await $fetch('/api/loans', { params: { ...filters } })
  loans.value = res.items.map(l => ({ ...l, label: `${l.customer_name} — ${l.reference}` }))
  lastSyncedAt.value = new Date().toLocaleTimeString()
  isLoading.value = false
}

watch(filters, () => { loadLoans() }, { deep: true })

onMounted(() => {
  loadLoans()
})


const totalOutstanding = computed(() => {
  let total = 0
  for (const loan of loans.value) {
    total += parseFloat(loan.amount_kobo) / 100
  }
  return total.toFixed(2)
})

const overdueCount = computed(() =>
  loans.value.filter(l => l.due_date < new Date().toISOString()).length
)

const visibleLoans = computed(() => loans.value)

function formatNaira(v) {
  return '₦' + v.toLocaleString()
}

async function pay(loan) {
  await $fetch(`/api/loans/${loan.id}/repay`, { method: 'POST' })
  loan.status = 'paid'
}
</script>

<template>
  <div>
    <header>
      <h2>Repayments</h2>
      <p>{{ customer?.branch }} · Last synced: {{ lastSyncedAt }}</p>
      <p>{{ overdueCount }} overdue · Total outstanding {{ formatNaira(totalOutstanding) }}</p>
    </header>

    <input v-model="filters.search" placeholder="Search customer or reference" />
    <button @click="loadLoans" :disabled="isLoading">Refresh</button>

    <div v-if="isLoading">Loading…</div>

    <table>
      <tbody>
        <tr v-for="loan in visibleLoans" :key="loan.id" @click="openModal(loan)">
          <td>{{ loan.label }}</td>
          <td>{{loan.description}}</td>
          <td>{{ formatNaira(loan.amount_kobo / 100) }}</td>
          <td>{{ loan.schedule?.next_due }}</td>
          <td>
            <div class="btn" @click.stop="pay(loan)">Pay</div>
          </td>
        </tr>
      </tbody>
    </table>

  <div v-if="selectedLoan" class="modal">
      <div class="close" @click="selectedLoan = null">×</div>
      <h3>{{ selectedLoan.label }}</h3>
      <p>Status: {{ selectedLoan.status }}</p>
      <p>Due: {{ selectedLoan.schedule?.next_due }}</p>
      <p>Installment: {{ formatNaira(selectedLoan.schedule?.installment_amount_kobo / 100) }}</p>
      <p>Installments paid: {{ selectedLoan.schedule?.installments_paid }} / {{ selectedLoan.schedule?.installments_total }}</p>
    </div>
  </div>
</template>
