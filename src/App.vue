<script setup lang="ts">
import { computed, ref } from 'vue'
import { Cog, Info } from 'lucide-vue-next'
import katex from 'katex'
import { NativeSelect, NativeSelectOption } from '@/components/ui/native-select'
import { Input } from '@/components/ui/input'
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
import { Button } from '@/components/ui/button'
import { formatCurrency } from './lib/utils.ts'

import 'katex/dist/katex.min.css'

function renderMath(tex: string, display = true): string {
  return katex.renderToString(tex, {
    displayMode: display,
    throwOnError: false,
    output: 'html',
  })
}

const formulaHtml = {
  pt: renderMath('p_t = \\text{price}_{\\text{/sqm}} \\times \\text{area}_{\\text{sqm}}'),
  pd: renderMath('p_d = p_t \\times \\text{downPayment}\\%'),
  tt: renderMath('t_t = \\text{totalPeriodYears} \\times 12'),
  pm: renderMath('p_m = \\frac{p_t - p_d}{t_t}'),
  pf: renderMath('p_f = \\text{expectedFulfillment}\\% \\times p_t'),
  rate: renderMath('r = \\frac{(p_m \\times \\text{milestoneMonths}) + p_d}{p_f}'),
  pi: renderMath('p_i = \\frac{p_t - p_d}{\\text{installments}}'),
  pI: renderMath('p_I = p_i \\times (1 + r)'),
  pT: renderMath('p_T = p_d + (p_I \\times \\text{installments})'),
}

interface Unit { id: string; name: string; area_in_sqm: number; price_per_sqm: number }

const unitsOptions = ref<Unit[]>([
  { id: '1', name: '3 Bedroom Apartment', area_in_sqm: 135, price_per_sqm: 150000 },
  { id: '2', name: '4 Bedroom Apartment', area_in_sqm: 148, price_per_sqm: 146000 },
  { id: '3', name: '1 Bedroom Studio', area_in_sqm: 40, price_per_sqm: 135000 },
])
const units = ref('')
const downPayment = ref('')
const installments = ref('')
const totalPeriodYears = ref<number>(7)
const milestoneMonths = ref<number>(30)
const expectedFulfillmentPct = ref<number>(60)

const tt = computed(() => {
  if (!totalPeriodYears.value) return undefined;
  return totalPeriodYears.value * 12;
})
const selectedUnit = computed(() => unitsOptions.value.find((unit) => unit.id === units.value))

const areAllValuesSet = computed(() => {
  return !!units.value && !!downPayment.value && !!installments.value && !!totalPeriodYears.value && !!milestoneMonths.value && !!expectedFulfillmentPct.value
})

const pt = computed(() => {
  if (!selectedUnit.value) return undefined;
  return (((selectedUnit.value).price_per_sqm * (selectedUnit.value).area_in_sqm))
})

const pd = computed(() => {
  if (!downPayment.value) return undefined;
  if (!pt.value) return undefined;
  return pt.value * (Number(downPayment.value) / 100);
})

const pm = computed(() => {
  if (!areAllValuesSet.value) return undefined;
  return (pt.value as number - pd.value as number) / tt.value as number;
})

const pf = computed(() => {
  if (!expectedFulfillmentPct.value) return undefined;
  return (expectedFulfillmentPct.value / 100) * pt.value;
})

const pi = computed(() => {
  if (!areAllValuesSet.value) return undefined;
  return (pt.value - pd.value) / installments.value;
})

const rateFromBase = computed(() => {
  if (!areAllValuesSet.value) return undefined;
  return ((pm.value * milestoneMonths.value) + pd.value) / pf.value;
})

const pI = computed(() => {
  if (!rateFromBase.value) return undefined;
  return pi.value * (1 + rateFromBase.value)
})

const pT = computed(() => {
  if (!rateFromBase.value) return undefined;
  return pd.value + (pI.value * installments.value);
})
</script>

<template>
  <div class="container  py-12 px-3 gap-4 mx-auto max-w-xl flex items-center flex-col">
    <div class="w-full flex px-2 items-center justify-between">
      <h1 class="font-semibold text-lg">Payment Calculation</h1>
      <Sheet>
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
                >Total Installment Period (in years)</label
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
                This is the total number of years over which the installments will be calculated. It
                is used to determine the total number of months for the installment plan.
              </p>
            </div>

            <div class="flex flex-col gap-4 mt-8">
              <p class="text-sm px-2">
                <Info class="size-3 inline" />
                The two values below collectively will be used alongside the client's chosen
                installment structure (no. of installments and initial down payment) to determine
                the payment schedule and amounts for each milestone. The "Milestone Period" is the
                estimated number of months it is expected for the client to have paid the specified
                "% of Expected Fulfillment". For example, if the "Milestone Period" is set to 30
                months and the "% of Expected Fulfillment" is set to 60%, it means that, upon the
                calculated payment structure, it is expected for the client to have paid 60% of the
                total payment amount (based a unit's base price) within the first 30 months of the
                installment plan.
              </p>
              <div class="flex flex-col gap-2">
                <label for="milestone-period" class="text-sm font-medium">
                  Milestone Period (in months)
                </label>
                <div class="relative">
                  <Input id="milestone-period" v-model="milestoneMonths" class="pr-17 text-right" />
                  <span
                    class="text-muted-foreground pointer-events-none absolute right-3 top-1/2 mt-[0.5px] -translate-y-1/2 text-sm leading-9"
                    >Months</span
                  >
                </div>
              </div>

              <div class="flex flex-col gap-2">
                <label for="expected-fulfillment" class="text-sm font-medium">
                  % of Expected Fulfillment
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
            </div>
          </div>
        </SheetContent>
      </Sheet>
    </div>

    <div class="flex w-full flex-col gap-8">
      <div class="w-full flex flex-col gap-4 items-center">
        <div class="w-full flex flex-col gap-2">
          <label for="units" class="text-sm font-medium">Units <span class="text-gray-500">(for demonstration purposes)</span> </label>
          <div class="flex flex-col gap-1">
            <NativeSelect id="units" v-model="units" :class="['w-full', !units && 'text-gray-400']">
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
          <NativeSelect
            id="down-payment"
            v-model="downPayment"
            :disabled="!units"
            :class="['w-full', !downPayment && 'text-gray-400']"
          >
            <NativeSelectOption value="" disabled>Choose Down Payment Rate</NativeSelectOption>
            <NativeSelectOption value="10">10%</NativeSelectOption>
            <NativeSelectOption value="15">15%</NativeSelectOption>
            <NativeSelectOption value="20">20%</NativeSelectOption>
            <NativeSelectOption value="30">30%</NativeSelectOption>
          </NativeSelect>
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
              <NativeSelectOption value="" disabled>Choose Payment Installment</NativeSelectOption>
              <NativeSelectOption value="6">6</NativeSelectOption>
              <NativeSelectOption value="12">12</NativeSelectOption>
              <NativeSelectOption value="24">24</NativeSelectOption>
              <NativeSelectOption value="36">36</NativeSelectOption>
            </NativeSelect>
            <p class="text-sm px-2 flex items-center gap-4">
            <span class="inline text-gray-400">
              To be paid over {{ totalPeriodYears }} years period (this can be changed in admin settings)
            </span>
            </p>
          </div>
        </div>
      </div>

      <div v-if="areAllValuesSet" class="w-full flex flex-col gap-4 rounded-md bg-gray-50 p-5 items-center">
        <div class="flex flex-col w-full">
          <h3 class="font-semibold text-sm">Payment Breakdown</h3>
          <p class="text-sm text-gray-500">Based on the client's choice of payment structure, and the configurations set by the admin; below is breakdown of payment calculation.</p>
        </div>

        <div id="payment-breakdown" class="flex flex-col px-2 w-full gap-3">
          <div class="flex flex-col w-full">
            <label class="text-sm">Installments</label>
            <p class="font-semibold">{{ installments }} payments within {{ totalPeriodYears }} years</p>
          </div>
          <div class="w-full h-px border-t border-dashed border-gray-300" />

          <div class="flex flex-col w-full">
            <label class="text-sm">Adjusted Payment per Installment</label>
            <p class="font-semibold">{{ formatCurrency(pI) }}</p>
          </div>
          <div class="w-full h-px border-t border-dashed border-gray-300" />

          <div class="flex flex-col w-full">
            <label class="text-sm">Adjusted Total Payment (incl. Down Payment)</label>
            <p class="font-semibold">
              {{ formatCurrency(pT) }}
            </p>
            <p class="text-xs text-green-600">
              <Info class="size-3 inline stroke-green-600 -translate-y-px" />
              {{ Math.round(pT / pt * 100) - 100 }}% increase from base price
            </p>
          </div>
        </div>

        <Dialog>
          <DialogTrigger as-child>
            <Button variant="outline" size="sm" class="mt-2 w-full sm:w-auto">
              <Info class="size-4" />
              View formulas
            </Button>
          </DialogTrigger>
          <DialogContent class="max-w-2xl max-h-[85vh] overflow-y-auto">
            <DialogHeader>
              <DialogTitle>Payment calculation formulas</DialogTitle>
              <DialogDescription>
                Mathematical notation used to compute the payment breakdown from unit price, down payment, installments, and admin configuration.
              </DialogDescription>
            </DialogHeader>
            <div class="space-y-6 pt-2 text-sm">
              <section>
                <h4 class="font-medium text-foreground mb-2">Base Total Price</h4>
                <div class="katex-formula bg-muted/50 rounded-md p-3" v-html="formulaHtml.pt" />
              </section>
              <section>
                <h4 class="font-medium text-foreground mb-2">Down Payment (in ETB / USD)</h4>
                <div class="katex-formula bg-muted/50 rounded-md p-3" v-html="formulaHtml.pd" />
              </section>
              <section>
                <h4 class="font-medium text-foreground mb-2">Total Months of Installment Period</h4>
                <div class="katex-formula bg-muted/50 rounded-md p-3" v-html="formulaHtml.tt" />
              </section>
              <section>
                <h4 class="font-medium text-foreground mb-2">Base Payment per Installment</h4>
                <div class="katex-formula bg-muted/50 rounded-md p-3" v-html="formulaHtml.pi" />
              </section>
              <section>
                <h4 class="font-medium text-foreground mb-2">Expected Fulfillment Payment (in ETB / USD)</h4>
                <div class="katex-formula bg-muted/50 rounded-md p-3" v-html="formulaHtml.pf" />
              </section>
              <section>
                <h4 class="font-medium text-foreground mb-2">Base Monthly Payment</h4>
                <div class="katex-formula bg-muted/50 rounded-md p-3" v-html="formulaHtml.pm" />
              </section>
              <section>
                <h4 class="font-medium text-foreground mb-2">Increase in % from Base Price to Achieve Milestone</h4>
                <div class="katex-formula bg-muted/50 rounded-md p-3" v-html="formulaHtml.rate" />
              </section>
              <section>
                <h4 class="font-medium text-foreground mb-2">Adjusted Payment per Installment</h4>
                <div class="katex-formula bg-muted/50 rounded-md p-3" v-html="formulaHtml.pI" />
              </section>
              <section>
                <h4 class="font-medium text-foreground mb-2">Adjusted Total Payment</h4>
                <div class="katex-formula bg-muted/50 rounded-md p-3" v-html="formulaHtml.pT" />
              </section>
            </div>
          </DialogContent>
        </Dialog>
      </div>
    </div>
  </div>
</template>

<style scoped></style>
