<script setup lang="ts">
import { computed, ref } from 'vue'
import { Calculator, Cog, Info } from 'lucide-vue-next'
import katex from 'katex'
import { NativeSelect, NativeSelectOption } from '@/components/ui/native-select'
import { Input } from '@/components/ui/input'
import { Switch } from '@/components/ui/switch'
import {
  Sheet,
  SheetContent,
  SheetDescription,
  SheetHeader,
  SheetTitle,
  SheetTrigger,
} from '@/components/ui/sheet'
import {
  Dialog,
  DialogContent,
  DialogDescription,
  DialogHeader,
  DialogTitle,
  DialogTrigger,
} from '@/components/ui/dialog'
import {
  Table,
  TableBody,
  TableCell,
  TableHead,
  TableHeader,
  TableRow,
} from '@/components/ui/table'
import { formatCurrency } from './lib/utils.ts'
import 'katex/dist/katex.min.css'

const aggressiveTargetFulfillment = ref<boolean>(false)

function renderMath(tex: string, display = true): string {
  return katex.renderToString(tex, {
    displayMode: display,
    throwOnError: false,
    output: 'html',
  })
}

const formulaHtml = {
  ip: {
    main: renderMath(
      'i_p = \\left\\lceil \\frac{\\text{installments}}{t_t} \\times \\text{milestoneMonths} \\right\\rceil',
    ),
    where: renderMath('\\text{where } t_t = \\text{totalPeriodYears} \\times 12'),
  },
  pi: {
    main: renderMath('p_i = \\frac{p_t - p_d}{\\text{installments}}'),
    where: renderMath('\\text{where } p_d = p_t \\times \\text{downPayment}\\%'),
  },
  ps: {
    main: renderMath('p_s = \\frac{p_f - p_d - (i_p \\times p_i)}{i_p}'),
    where: renderMath('\\text{where } p_f = \\text{expectedFulfillment}\\% \\times p_t'),
  },
  pI: renderMath('p_I = p_i + p_s'),
  pT: renderMath('p_T = p_d + (p_I \\times \\text{installments})'),
}

interface Unit {
  id: string
  name: string
  area_in_sqm: number
  price_per_sqm: number
}

interface DownPayment {
  value: string
}

interface Installment {
  value: string
}

const installmentOptions: Installment[] = [
  { value: '6' },
  { value: '12' },
  { value: '24' },
  { value: '36' },
]

const downPaymentOptions: DownPayment[] = [
  { value: '10' },
  { value: '20' },
  { value: '30' },
  { value: '40' },
]

const unitsOptions: Unit[] = [
  { id: '1', name: '3 Bedroom Apartment', area_in_sqm: 135, price_per_sqm: 150000 },
  { id: '2', name: '4 Bedroom Apartment', area_in_sqm: 148, price_per_sqm: 146000 },
  { id: '3', name: '1 Bedroom Studio', area_in_sqm: 40, price_per_sqm: 135000 },
  { id: '4', name: '1 Bedroom Studio', area_in_sqm: 100, price_per_sqm: 100000 },
]
const units = ref('')
const downPayment = ref('')
const installments = ref('')
const totalPeriodYears = ref<number>(7)
const milestoneMonths = ref<number>(30)
const expectedFulfillmentPct = ref<number>(60)

const tt = computed(() => {
  if (!totalPeriodYears.value) return undefined
  return totalPeriodYears.value * 12
})
const selectedUnit = computed(() => unitsOptions.find((unit) => unit.id === units.value))

const areAllValuesSet = computed(() => {
  return (
    !!units.value &&
    !!downPayment.value &&
    !!installments.value &&
    !!totalPeriodYears.value &&
    !!milestoneMonths.value &&
    !!expectedFulfillmentPct.value
  )
})

/**
 * Months per Installment
 */
const ti = computed(() => {
  if (!installments.value) return undefined
  if (!tt.value) return undefined
  return (tt.value as number) / (Number(installments.value) as number)
})

/**
 * Base Total Payment
 */
const pt = computed(() => {
  if (!selectedUnit.value) return undefined
  return selectedUnit.value.price_per_sqm * selectedUnit.value.area_in_sqm
})

/**
 * Down Payment in ETB
 */
const pd = computed(() => {
  if (!downPayment.value) return undefined
  if (!pt.value) return undefined
  return pt.value * (Number(downPayment.value) / 100)
})

/**
 * Total Target Payment on Target Period in ETB
 */
const pf = computed(() => {
  if (!expectedFulfillmentPct.value) return undefined
  return (expectedFulfillmentPct.value / 100) * (pt.value as number)
})

/**
 * Unadjusted Payment per Installment
 */
const pi = computed(() => {
  if (!areAllValuesSet.value) return undefined
  return (
    ((pt.value as number) - (pd.value as number)) / (Number(installments.value as string) as number)
  )
})

/**
 * No. of Installments to reach Target Period
 */
const ip = computed(() => {
  if (!milestoneMonths.value) return undefined
  if (!installments.value) return undefined
  const base = (Number(installments.value) / (tt.value as number)) * milestoneMonths.value
  return aggressiveTargetFulfillment.value ? Math.floor(base) : Math.ceil(base)
})

/**
 * Profit Surcharge Amount to reach Target Payment at Target Period
 */
const ps = computed(() => {
  if ((ip.value as number) < 1) return undefined
  return (
    ((pf.value as number) - (pd.value as number) - (ip.value as number) * (pi.value as number)) /
    (ip.value as number)
  )
})

/**
 * Adjusted Payment per Installment
 */
const pI = computed(() => {
  if (!areAllValuesSet.value) return undefined
  if (!ps.value) return pi.value
  return (pi.value as number) + (ps.value as number)
})

/**
 * Adjusted Total Payment
 */
const pT = computed(() => {
  if (!areAllValuesSet.value) return undefined
  return (
    (pd.value as number) + (pI.value as number) * (Number(installments.value as string) as number)
  )
})

/**
 * Payment breakdown rows: one per selected installment.
 * Each row: milestone #, installment date (months), fulfilled payment at that milestone.
 */
const paymentBreakdownRows = computed(() => {
  if (!areAllValuesSet.value || !installments.value || !ti.value || !pd.value || !pI.value)
    return []
  const n = Number(installments.value)
  const rows: { milestone: number; dateMonths: number; fulfilledPayment: number }[] = []
  for (let k = 1; k <= n; k++) {
    rows.push({
      milestone: k,
      dateMonths: Math.round(k * (ti.value as number)),
      fulfilledPayment: (pd.value as number) + k * (pI.value as number),
    })
  }
  return rows
})
</script>

<template>
  <div class="container py-12 px-3 gap-4 mx-auto max-w-xl flex items-center flex-col">
    <Sheet>
      <div class="w-full flex px-2 items-center justify-between">
        <h1 class="font-semibold text-lg">Payment Calculation</h1>
        <div class="flex items-center gap-2">
          <Dialog>
            <DialogTrigger as-child>
              <button
                type="button"
                class="rounded-md p-1.5 text-muted-foreground transition-colors hover:bg-muted hover:text-foreground focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring focus-visible:ring-offset-2"
                aria-label="View formulas"
              >
                <Calculator class="size-5 shrink-0" aria-hidden="true" />
              </button>
            </DialogTrigger>
            <DialogContent
              class="max-w-2xl w-[calc(100vw-2rem)] max-h-[85vh] overflow-y-auto overflow-x-auto min-w-0"
            >
              <DialogHeader>
                <DialogTitle>Payment calculation formulas</DialogTitle>
                <DialogDescription>
                  Mathematical notation used to compute the payment breakdown from unit price, down
                  payment, installments, and admin configuration.
                </DialogDescription>
              </DialogHeader>
              <div id="formulas" class="space-y-6 pt-2 text-sm min-w-0">
                <section class="min-w-0">
                  <h4 class="font-medium text-foreground mb-2">
                    Target Installments (i<sub>p</sub>)
                  </h4>
                  <p class="text-muted-foreground mb-2">
                    Number of installments that occur within the target (milestone) period.
                  </p>
                  <div
                    class="katex-formula bg-muted/50 overflow-x-auto rounded-md py-3 min-w-0 space-y-2"
                  >
                    <div class="w-full mx-3" v-html="formulaHtml.ip.main" />
                    <div v-html="formulaHtml.ip.where" class="text-muted-foreground" />
                  </div>
                </section>
                <section class="min-w-0">
                  <h4 class="font-medium text-foreground mb-2">
                    Unadjusted Payment per Installment (p<sub>i</sub>)
                  </h4>
                  <p class="text-muted-foreground mb-2">
                    The base payment per installment that covers the base price over the full term.
                  </p>
                  <div
                    class="katex-formula bg-muted/50 overflow-x-auto rounded-md py-3 min-w-0 space-y-2"
                  >
                    <div class="w-full mx-3" v-html="formulaHtml.pi.main" />
                    <div v-html="formulaHtml.pi.where" class="text-muted-foreground" />
                  </div>
                </section>
                <section class="min-w-0">
                  <h4 class="font-medium text-foreground mb-2">Profit Surcharge (p<sub>s</sub>)</h4>
                  <p class="text-muted-foreground mb-2">
                    The profit surcharge amount per installment to reach the target payment within
                    the target period.
                  </p>
                  <div
                    class="katex-formula bg-muted/50 overflow-x-auto rounded-md py-3 min-w-0 space-y-2"
                  >
                    <div class="w-full mx-3" v-html="formulaHtml.ps.main" />
                    <div v-html="formulaHtml.ps.where" class="text-muted-foreground" />
                  </div>
                </section>
                <section class="min-w-0">
                  <h4 class="font-medium text-foreground mb-2">
                    Adjusted Payment per Installment (p<sub>I</sub>)
                  </h4>
                  <p class="text-muted-foreground mb-2">
                    Unadjusted payment plus the profit surcharge.
                  </p>
                  <div
                    class="katex-formula bg-muted/50 rounded-md p-3 min-w-0"
                    v-html="formulaHtml.pI"
                  />
                </section>
                <section class="min-w-0">
                  <h4 class="font-medium text-foreground mb-2">
                    Adjusted Total Payment (p<sub>T</sub>)
                  </h4>
                  <p class="text-muted-foreground mb-2">
                    Down payment plus the sum of all adjusted installment payments.
                  </p>
                  <div
                    class="katex-formula bg-muted/50 rounded-md p-3 min-w-0"
                    v-html="formulaHtml.pT"
                  />
                </section>
              </div>
            </DialogContent>
          </Dialog>
          <SheetTrigger as-child>
            <button
              type="button"
              class="rounded-md p-1.5 text-muted-foreground transition-colors hover:bg-muted hover:text-foreground focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring focus-visible:ring-offset-2"
              aria-label="Open settings"
            >
              <Cog class="size-5 shrink-0" aria-hidden="true" />
            </button>
          </SheetTrigger>
          <SheetContent side="right">
            <SheetHeader>
              <SheetTitle>Admin Configurations</SheetTitle>
              <SheetDescription>
                Configurations set by the administrator to calculate payments based on the client's
                chosen structure
              </SheetDescription>
            </SheetHeader>
            <div class="flex overflow-auto flex-col gap-4 px-3 py-4">
              <div class="flex flex-col gap-2">
                <label for="total-period-years" class="text-sm font-medium"
                  >Full Term Payment Period (in years)</label
                >
                <div class="relative">
                  <Input
                    id="total-period-years"
                    v-model="totalPeriodYears"
                    class="pr-13 text-right"
                  />
                  <span
                    class="text-muted-foreground pointer-events-none absolute right-3 top-1/2 mt-[0.5px] -translate-y-1/2 text-sm leading-9"
                    >Years</span
                  >
                </div>
                <p class="text-sm px-2">
                  <Info class="size-3 inline" />
                  This is the full term period over which the client will be expected to complete
                  payment in installments for the unit.
                </p>
              </div>

              <div class="h-px w-full border-t my-4" />

              <div class="flex flex-col gap-4">
                <div class="flex flex-col gap-2">
                  <label for="milestone-period" class="text-sm font-medium">
                    Target Period (in months)
                  </label>
                  <div class="relative">
                    <Input
                      id="milestone-period"
                      v-model="milestoneMonths"
                      class="pr-17 text-right"
                    />
                    <span
                      class="text-muted-foreground pointer-events-none absolute right-3 top-1/2 mt-[0.5px] -translate-y-1/2 text-sm leading-9"
                      >Months</span
                    >
                  </div>
                </div>

                <div class="flex flex-col gap-2">
                  <label for="expected-fulfillment" class="text-sm font-medium">
                    Target Payment %
                  </label>
                  <div class="relative">
                    <Input
                      id="expected-fulfillment"
                      v-model="expectedFulfillmentPct"
                      min="5"
                      max="95"
                      class="pr-8 text-right"
                    />
                    <span
                      class="text-muted-foreground pointer-events-none absolute right-3 top-1/2 mt-[0.5px] -translate-y-1/2 text-sm leading-9"
                      >%</span
                    >
                  </div>
                </div>

                <p class="text-sm px-2">
                  <Info class="size-3 inline" />
                  These two settings work with the client's installment plan (number of installments
                  and initial down payment) to calculate each target payment. The
                  <span class="font-semibold italic">Target Period</span> is the number of months by
                  which the client should have paid the selected
                  <span class="font-semibold italic">Target Payment %</span>. For example, if the
                  Target Period is 30 months and the Target Payment % is 60%, the plan assumes the
                  client has paid about 60% of the total price (using the unit's base price) by the
                  end of month 30.
                </p>
              </div>

              <div class="h-px w-full border-t my-4" />

              <label
                for="aggressive-target-fulfillment"
                class="flex flex-col gap-2 bg-gray-50 border p-4 rounded-sm"
              >
                <div class="flex items-center justify-between gap-3">
                  <p class="text-sm font-medium">Aggressive Target Fulfillment</p>
                  <Switch
                    id="aggressive-target-fulfillment"
                    v-model="aggressiveTargetFulfillment"
                    aria-label="Aggressive target fulfillment"
                  />
                </div>
                <p class="text-sm px-2">
                  <Info class="size-3 inline" />
                  When enabled, the target installment calculation assumes
                  <span class="font-semibold italic">the last installment before</span> the target
                  period is the one that fulfills the target payment. For example, if the target
                  period is month 30, and there are installments at months 6, 12, 18, 24, and 36;
                  with aggressive fulfillment enabled, the calculation assumes the client fulfills
                  the target payment at month 24 (instead of month 36 with aggressive fulfillment
                  disabled). This results in a more aggressive payment adjustment, as more
                  installments are adjusted to meet the target payment by month 30.
                </p>
              </label>
            </div>
          </SheetContent>
        </div>
      </div>

      <div class="flex w-full flex-col gap-8">
        <div class="w-full flex flex-col gap-4 items-center">
          <div class="w-full flex flex-col gap-2">
            <label for="units" class="text-sm font-medium"
              >Units <span class="text-gray-500">(for demonstration purposes)</span>
            </label>
            <div class="flex flex-col gap-1">
              <NativeSelect
                id="units"
                v-model="units"
                :class="['w-full', !units && 'text-gray-400']"
              >
                <NativeSelectOption value="" disabled>Choose Unit</NativeSelectOption>
                <NativeSelectOption v-for="opt in unitsOptions" :key="opt.id" :value="opt.id">
                  {{ opt.name }}
                </NativeSelectOption>
              </NativeSelect>
              <p v-if="units" class="text-sm px-2 flex items-center gap-4">
                <span class="inline">
                  <span class="text-gray-400">Area:</span>
                  {{ unitsOptions.find((unit) => unit.id === units)?.area_in_sqm }} sq. m.
                </span>
                <span class="inline">
                  <span class="text-gray-400">Price:</span>
                  {{
                    formatCurrency(
                      unitsOptions.find((unit) => unit.id === units)?.price_per_sqm as number,
                    )
                  }}
                  per sq. m.
                </span>
              </p>
            </div>
          </div>
          <div class="w-full flex flex-col gap-2">
            <label for="down-payment" class="text-sm font-medium">Down Payment</label>
            <div class="flex flex-col gap-1">
              <NativeSelect
                id="down-payment"
                v-model="downPayment"
                :disabled="!units"
                :class="['w-full', !downPayment && 'text-gray-400']"
              >
                <NativeSelectOption value="" disabled>Choose Down Payment Rate</NativeSelectOption>
                <NativeSelectOption
                  v-for="opt in downPaymentOptions"
                  :key="opt.value"
                  :value="opt.value"
                >
                  {{ opt.value }}%
                </NativeSelectOption>
              </NativeSelect>
              <p v-if="pd" class="text-sm px-2 flex items-center gap-4">
                <span class="inline">
                  <span class="text-gray-400">Amount:</span>
                  {{ formatCurrency(pd as number) }}
                </span>
              </p>
            </div>
          </div>
          <div class="w-full flex flex-col gap-2">
            <label for="installments" class="text-sm font-medium">Installments</label>
            <div class="flex flex-col gap-1">
              <NativeSelect
                id="installments"
                v-model="installments"
                :disabled="!units"
                :class="['w-full', !installments && 'text-gray-400']"
              >
                <NativeSelectOption value="" disabled
                  >Choose Payment Installment</NativeSelectOption
                >
                <NativeSelectOption
                  v-for="opt in installmentOptions"
                  :key="opt.value"
                  :value="opt.value"
                >
                  {{ opt.value }}
                </NativeSelectOption>
              </NativeSelect>
              <p class="text-sm px-2 flex items-center gap-4">
                <span class="inline text-gray-400">
                  To be paid over {{ totalPeriodYears }} years period (this can be changed in
                  <SheetTrigger as-child>
                    <button
                      type="button"
                      class="underline underline-offset-3 decoration-dotted cursor-pointer text-gray-400 hover:text-foreground focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring focus-visible:ring-offset-2 rounded"
                    >
                      admin settings
                    </button>
                  </SheetTrigger>
                  )
                </span>
              </p>
            </div>
          </div>
        </div>

        <div
          id="payment-breakdown"
          v-if="areAllValuesSet"
          class="w-full flex flex-col gap-4 rounded-md bg-gray-900 p-5 items-center"
        >
          <div class="flex flex-col w-full">
            <h3 class="font-semibold text-sm text-gray-100">Payment Breakdown</h3>
            <p class="text-sm text-gray-400">
              Based on the client's choice of payment structure, and the configurations set by the
              admin; below is breakdown of payment calculation.
            </p>
          </div>

          <div class="flex flex-col px-2 w-full gap-3">
            <div class="flex flex-col w-full">
              <label class="text-sm text-gray-300">Installments</label>
              <p class="font-semibold text-sm text-gray-100">
                {{ installments }} payments every ~{{ Number(ti).toFixed(1) }} months
              </p>
            </div>
            <div class="w-full h-px border-t border-dashed border-gray-600" />

            <div class="flex flex-col w-full">
              <label class="text-sm text-gray-300">Payment per Installment</label>
              <p class="font-semibold text-sm text-gray-100">{{ formatCurrency(pI as number) }}</p>
            </div>
            <div class="w-full h-px border-t border-dashed border-gray-600" />

            <div class="flex flex-col w-full">
              <label class="text-sm text-gray-300">Total Payment (incl. Down Payment)</label>
              <p class="font-semibold text-sm text-gray-100">
                {{ formatCurrency(pT as number) }}
              </p>
              <p class="text-xs text-green-400">
                <Info class="size-3 inline stroke-green-400 -translate-y-px" />
                {{ Math.round(((pT as number) / (pt as number)) * 100) - 100 }}% increase from base
                price
              </p>
            </div>
            <div class="w-full h-px border-t border-dashed border-gray-600" />

            <div class="flex flex-col w-full">
              <Table class="w-full border border-gray-700 rounded-md overflow-hidden">
                <TableHeader>
                  <TableRow class="border-gray-700 hover:bg-gray-800/50">
                    <TableHead class="pl-0 text-left text-gray-300 font-medium"
                      >Milestone</TableHead
                    >
                    <TableHead class="text-center text-gray-300 font-medium">Deadline</TableHead>
                    <TableHead class="pr-0 text-right text-gray-300 font-medium"
                      >Cumulative paid</TableHead
                    >
                  </TableRow>
                </TableHeader>
                <TableBody>
                  <TableRow
                    v-for="row in paymentBreakdownRows"
                    :key="row.milestone"
                    class="border-gray-700 hover:bg-gray-800/50"
                  >
                    <TableCell class="pl-0 text-left text-gray-200">{{ row.milestone }}</TableCell>
                    <TableCell class="text-center text-gray-200"
                      >Month {{ row.dateMonths }}</TableCell
                    >
                    <TableCell class="pr-0 text-right text-gray-200 font-medium">
                      {{ formatCurrency(row.fulfilledPayment) }}
                    </TableCell>
                  </TableRow>
                </TableBody>
              </Table>
            </div>
          </div>
        </div>
      </div>
    </Sheet>
  </div>
</template>

<style scoped></style>
