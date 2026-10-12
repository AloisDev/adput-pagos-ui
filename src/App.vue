<script setup lang="ts">
import { computed, nextTick, onBeforeUnmount, onMounted, ref, watch } from 'vue'

type Payment = {
  id: string
  date: string
  concept: string
  amount: number
  baseAmount: number
  lateFee: number
  dueDate: string
  type: 'mensual' | 'extra'
  overdue: boolean
  status: 'Pagado' | 'Pendiente' | 'Vencida'
}
type Receipt = { number: string; date: string; concept: string; amount: number; method: string }
type ExtraCharge = {
  id: string
  descripcion: string
  montoCentavos: number
  fechaVencimiento: string
  sociosAlcanzados: number
  pagosRegistrados: number
  bloqueado: boolean
  estado: 'borrador' | 'aplicado'
}
type ExtractRow = {
  socio: { nombre: string; apellido: string; numeroSocio: string }
  cuotasAdeudadas: number
  importeCuotasAdeudadasCentavos: number
  importeExtrasAdeudadosCentavos: number
  ultimaCuotaPagadaPeriodo: string | null
  situacion: 'al_dia' | 'debe_1' | 'debe_2_mas'
}
type AccountResponse = {
  socio: { numeroSocio: string; estado: string }
  deudaCentavos: number
  cuotas: Array<{
    id: string
    periodo: string
    fechaVencimiento: string
    descripcion: string
    tipo: 'mensual' | 'extra'
    baseAmountCents: number
    lateFeeCents: number
    payableAmountCents: number
    overdue: boolean
    estado: string
  }>
}
type ExtractResponse = {
  fechaCorte: string
  pagina: number
  tamanoPagina: number
  total: number
  resumen: { alDia: number; debeUna: number; debeDosOMas: number; deudaCuotasCentavos: number; deudaExtrasCentavos: number }
  socios: ExtractRow[]
}
type PreparedPaymentOrder = {
  id: string
  estado: 'preparada'
  canal: 'checkout' | 'qr'
  montoCentavos: number
  venceEn: string | null
  cuotaIds: string[]
  items: Array<{ cuotaId: string; descripcion: string; montoCentavos: number }>
}

const apiUrl = import.meta.env.VITE_API_URL ?? '/api/v1'
const apiTimeoutMs = 15_000

const email = ref('')
const password = ref('')
const signedIn = ref(false)
const loginError = ref('')
const memberName = ref('')
const memberNumber = ref('')
const profileLastName = ref('')
const profileEmail = ref('')
const profileCurrentPassword = ref('')
const profileNewPassword = ref('')
const profileMessage = ref('')
const passwordMessage = ref('')
const userRole = ref<'socio' | 'administrador'>('socio')
const extraChargeDescription = ref('')
const extraChargeAmount = ref<number | null>(null)
const extraChargeDueDate = ref('')
const extraChargeMessage = ref('')
const extraChargeDialog = ref<HTMLDialogElement | null>(null)
const extraChargeTrigger = ref<HTMLButtonElement | null>(null)
const extraChargeDescriptionInput = ref<HTMLInputElement | null>(null)
const extraCharges = ref<ExtraCharge[]>([])
const extraChargeManagementMessage = ref('')
const extraChargeManagementDialog = ref<HTMLDialogElement | null>(null)
const extraChargeManagementTrigger = ref<HTMLButtonElement | null>(null)
const editingExtraCharge = ref<{ id: string; descripcion: string; montoPesos: number; fechaVencimiento: string } | null>(null)
const applyingExtraCharge = ref<ExtraCharge | null>(null)
const extraChargeApplicationConfirmed = ref(false)
const monthlyFeeAmount = ref<number | null>(null)
const monthlyFeeEffectiveMonth = ref(nextMonthStart().slice(0, 7))
const monthlyFeeMessage = ref('')
const monthlyFeeDialog = ref<HTMLDialogElement | null>(null)
const monthlyFeeTrigger = ref<HTMLButtonElement | null>(null)
const monthlyFeeAmountInput = ref<HTMLInputElement | null>(null)
const extractDate = ref(argentinaDateInput())
const extractSituation = ref<'todos' | 'al_dia' | 'debe_1' | 'debe_2_mas'>('todos')
const extractSearch = ref('')
const extractPage = ref(1)
const extract = ref<ExtractResponse | null>(null)
const extractLoading = ref(false)
const extractError = ref('')
const extractExporting = ref<'xlsx' | 'pdf' | null>(null)
let extractRequest = 0
let extractSearchTimer: ReturnType<typeof setTimeout> | undefined
const currentView = ref<'account' | 'receipts' | 'payment' | 'profile' | 'admin'>('account')
const monthlyFee = ref(0)
const payments = ref<Payment[]>([])
const receipts = ref<Receipt[]>([])
const selectedReceipt = ref<Receipt>({ number: '', date: '', concept: '', amount: 0, method: '' })
const cardholder = ref('')
const cardNumber = ref('')
const cardExpiry = ref('')
const cardCvc = ref('')
const paymentMethod = ref<'card' | 'qr'>('card')
const checkoutMethod = ref<'credito' | 'debito' | 'qr'>('credito')
const checkoutNotice = ref('')
const checkoutPreparing = ref(false)
const preparedPaymentOrder = ref<PreparedPaymentOrder | null>(null)
const preparedPaymentMessage = ref('')
const cancellingPreparedPayment = ref(false)
const selectedPaymentIds = ref<string[]>([])
const paymentComplete = ref(false)
const confirmedAmount = ref(0)
const receiptActionMessage = ref('')

const debt = computed(() => payments.value.filter((payment) => payment.status !== 'Pagado').reduce((total, payment) => total + payment.amount, 0))
const pendingPayments = computed(() => payments.value.filter((payment) => payment.status !== 'Pagado').sort((a, b) => a.dueDate.localeCompare(b.dueDate)))
const selectedPayments = computed(() => pendingPayments.value.filter((payment) => selectedPaymentIds.value.includes(payment.id)))
const selectedPaymentAmount = computed(() => selectedPayments.value.reduce((total, payment) => total + payment.amount, 0))
const nextPendingPayment = (type: Payment['type']) =>
  computed(() => payments.value.filter((payment) => payment.type === type && payment.status !== 'Pagado').sort((a, b) => a.dueDate.localeCompare(b.dueDate))[0])
const nextPendingMonthlyPayment = nextPendingPayment('mensual')
const nextPendingExtraPayment = nextPendingPayment('extra')
const pesos = (amount: number) =>
  new Intl.NumberFormat('es-AR', {
    style: 'currency',
    currency: 'ARS',
    maximumFractionDigits: 0,
  }).format(amount)

async function apiFetch(input: RequestInfo | URL, init: RequestInit = {}) {
  const controller = new AbortController()
  const timeout = window.setTimeout(() => controller.abort(), apiTimeoutMs)
  try {
    return await fetch(input, { credentials: 'include', ...init, signal: controller.signal })
  } catch (error) {
    if (error instanceof DOMException && error.name === 'AbortError') throw new Error('La solicitud demoró demasiado. Intentá nuevamente.')
    throw new Error('No se pudo conectar con ADPUT. Verificá tu conexión e intentá nuevamente.')
  } finally {
    window.clearTimeout(timeout)
  }
}

async function readApiJson<T>(response: Response) {
  try {
    return (await response.json()) as T
  } catch {
    throw new Error(response.ok ? 'El servidor respondió de forma inválida.' : 'No se pudo completar la solicitud.')
  }
}

function errorMessage(error: unknown, fallback: string) {
  return error instanceof Error ? error.message : fallback
}

async function loadAccount() {
  const response = await apiFetch(`${apiUrl}/mi-cuenta`)
  const data = await readApiJson<AccountResponse | { error: string }>(response)
  if (!response.ok || !('cuotas' in data)) throw new Error('error' in data ? data.error : 'No se pudo cargar tu cuenta.')

  memberNumber.value = data.socio.numeroSocio
  payments.value = data.cuotas.map((cuota) => ({
    id: cuota.id,
    date: formatDate(cuota.periodo),
    concept: cuota.descripcion,
    amount: cuota.payableAmountCents / 100,
    baseAmount: cuota.baseAmountCents / 100,
    lateFee: cuota.lateFeeCents / 100,
    dueDate: cuota.fechaVencimiento,
    type: cuota.tipo,
    overdue: cuota.overdue,
    status: cuota.estado === 'pagado' ? 'Pagado' : cuota.overdue ? 'Vencida' : 'Pendiente',
  }))
  monthlyFee.value = payments.value.find((payment) => payment.type === 'mensual')?.baseAmount ?? 0
  await loadPreparedPaymentOrder()
}

async function loadPreparedPaymentOrder() {
  const response = await apiFetch(`${apiUrl}/pagos/ordenes/preparada`)
  const data = await readApiJson<{ orden?: PreparedPaymentOrder | null; error?: string }>(response)
  if (!response.ok) throw new Error(data.error ?? 'No se pudo consultar el pago preparado.')
  preparedPaymentOrder.value = data.orden ?? null
}

function formatDate(value: string) {
  const [year, month, day] = value.slice(0, 10).split('-').map(Number)
  return new Intl.DateTimeFormat('es-AR', { day: 'numeric', month: 'long', year: 'numeric' }).format(
    new Date(year!, month! - 1, day!),
  )
}

function formatDateTime(value: string) {
  return new Intl.DateTimeFormat('es-AR', { dateStyle: 'short', timeStyle: 'short' }).format(new Date(value))
}

function formatMonth(value: string) {
  const [year, month] = value.slice(0, 10).split('-').map(Number)
  return new Intl.DateTimeFormat('es-AR', { month: 'long', year: 'numeric' }).format(new Date(year!, month! - 1, 1))
}

function formatPeriod(value: string) {
  const [year, month] = value.slice(0, 10).split('-')
  return `${month}-${year?.slice(-2)}`
}

async function signIn() {
  loginError.value = ''
  if (!email.value || !password.value) {
    loginError.value = 'Ingresá tu correo y contraseña.'
    return
  }
  try {
    const response = await apiFetch(`${apiUrl}/auth/login`, {
      method: 'POST',
      credentials: 'include',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ email: email.value, password: password.value }),
    })
    const data = await readApiJson<{
      usuario?: { nombre: string; apellido: string; email: string; rol: 'socio' | 'administrador' }
      error?: string
    }>(response)
    if (!response.ok || !data.usuario) throw new Error(data.error ?? 'No se pudo iniciar sesión.')
    memberName.value = data.usuario.nombre
    profileLastName.value = data.usuario.apellido
    profileEmail.value = data.usuario.email
    userRole.value = data.usuario.rol
    if (data.usuario.rol === 'administrador') {
      currentView.value = 'admin'
      password.value = ''
      await loadExtract()
      await loadMonthlyFeeConfig()
      signedIn.value = true
      return
    }
    await loadAccount()
    currentView.value = 'account'
    signedIn.value = true
    password.value = ''
  } catch (error) {
    loginError.value = error instanceof Error ? error.message : 'No se pudo iniciar sesión.'
  }
}

async function createExtraCharge() {
  extraChargeMessage.value = ''
  try {
    const response = await apiFetch(`${apiUrl}/admin/cargos-extra`, {
    method: 'POST',
    credentials: 'include',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      descripcion: extraChargeDescription.value,
      montoPesos: extraChargeAmount.value,
      fechaVencimiento: extraChargeDueDate.value,
    }),
  })
  const data = await readApiJson<{ id?: string; error?: string }>(response)
  if (!response.ok || !data.id) {
    extraChargeMessage.value = data.error ?? 'No se pudo crear el cargo.'
    return
  }
  extraChargeMessage.value = 'Borrador creado. Revisalo y confirmá su aplicación antes de generar deuda.'
  extraChargeDescription.value = ''
  extraChargeAmount.value = null
  extraChargeDueDate.value = ''
  await loadExtract()
    closeExtraChargeDialog()
  } catch (error) {
    extraChargeMessage.value = errorMessage(error, 'No se pudo crear el cargo.')
  }
}

async function loadExtraCharges() {
  const response = await apiFetch(`${apiUrl}/admin/cargos-extra`)
  const data = await readApiJson<{ cargos?: ExtraCharge[]; error?: string }>(response)
  if (!response.ok || !data.cargos) throw new Error(data.error ?? 'No se pudieron cargar los cargos extras.')
  extraCharges.value = data.cargos
}

async function openExtraChargeManagementDialog() {
  extraChargeManagementMessage.value = ''
  editingExtraCharge.value = null
  applyingExtraCharge.value = null
  extraChargeManagementDialog.value?.showModal()
  try {
    await loadExtraCharges()
  } catch (error) {
    extraChargeManagementMessage.value = error instanceof Error ? error.message : 'No se pudieron cargar los cargos extras.'
  }
}

function closeExtraChargeManagementDialog() {
  extraChargeManagementDialog.value?.close()
}

function onExtraChargeManagementDialogClose() {
  extraChargeManagementTrigger.value?.focus()
}

function editExtraCharge(charge: ExtraCharge) {
  extraChargeManagementMessage.value = ''
  applyingExtraCharge.value = null
  editingExtraCharge.value = {
    id: charge.id,
    descripcion: charge.descripcion,
    montoPesos: charge.montoCentavos / 100,
    fechaVencimiento: charge.fechaVencimiento,
  }
}

function startApplyingExtraCharge(charge: ExtraCharge) {
  extraChargeManagementMessage.value = ''
  editingExtraCharge.value = null
  applyingExtraCharge.value = charge
  extraChargeApplicationConfirmed.value = false
}

async function applyExtraCharge() {
  const charge = applyingExtraCharge.value
  if (!charge || !extraChargeApplicationConfirmed.value) return
  extraChargeManagementMessage.value = ''
  try {
    const response = await apiFetch(`${apiUrl}/admin/cargos-extra/${charge.id}/aplicar`, {
    method: 'POST',
    credentials: 'include',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ confirmar: true }),
  })
  const data = await readApiJson<{ sociosAlcanzados?: number; error?: string }>(response)
  if (!response.ok || data.sociosAlcanzados === undefined) {
    extraChargeManagementMessage.value = data.error ?? 'No se pudo aplicar el cargo extra.'
    return
  }
  applyingExtraCharge.value = null
  extraChargeApplicationConfirmed.value = false
  extraChargeManagementMessage.value = `Cargo aplicado a ${data.sociosAlcanzados} socio(s).`
    await Promise.all([loadExtraCharges(), loadExtract()])
  } catch (error) {
    extraChargeManagementMessage.value = errorMessage(error, 'No se pudo aplicar el cargo extra.')
  }
}

async function saveExtraCharge() {
  const charge = editingExtraCharge.value
  if (!charge) return
  extraChargeManagementMessage.value = ''
  try {
    const response = await apiFetch(`${apiUrl}/admin/cargos-extra/${charge.id}`, {
    method: 'PATCH',
    credentials: 'include',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ descripcion: charge.descripcion, montoPesos: charge.montoPesos, fechaVencimiento: charge.fechaVencimiento }),
  })
  const data = await readApiJson<{ error?: string }>(response)
  if (!response.ok) {
    extraChargeManagementMessage.value = data.error ?? 'No se pudo editar el cargo extra.'
    return
  }
  editingExtraCharge.value = null
  extraChargeManagementMessage.value = 'Cargo extra actualizado.'
    await Promise.all([loadExtraCharges(), loadExtract()])
  } catch (error) {
    extraChargeManagementMessage.value = errorMessage(error, 'No se pudo editar el cargo extra.')
  }
}

async function deleteExtraCharge(charge: ExtraCharge) {
  if (!confirm(`¿Eliminar el cargo “${charge.descripcion}” para ${charge.sociosAlcanzados} socio(s)?`)) return
  extraChargeManagementMessage.value = ''
  try {
    const response = await apiFetch(`${apiUrl}/admin/cargos-extra/${charge.id}`, { method: 'DELETE' })
  if (!response.ok) {
    const data = await readApiJson<{ error?: string }>(response)
    extraChargeManagementMessage.value = data.error ?? 'No se pudo eliminar el cargo extra.'
    return
  }
  if (editingExtraCharge.value?.id === charge.id) editingExtraCharge.value = null
  extraChargeManagementMessage.value = 'Cargo extra eliminado.'
    await Promise.all([loadExtraCharges(), loadExtract()])
  } catch (error) {
    extraChargeManagementMessage.value = errorMessage(error, 'No se pudo eliminar el cargo extra.')
  }
}

function openExtraChargeDialog() {
  extraChargeMessage.value = ''
  extraChargeDialog.value?.showModal()
  void nextTick(() => extraChargeDescriptionInput.value?.focus())
}

function closeExtraChargeDialog() {
  extraChargeDialog.value?.close()
}

function onExtraChargeDialogClose() {
  extraChargeTrigger.value?.focus()
}

async function loadMonthlyFeeConfig() {
  const response = await apiFetch(`${apiUrl}/admin/cuota-mensual`)
  const data = await readApiJson<{ configuracion?: { montoCentavos: number; vigenteDesde: string } | null; error?: string }>(response)
  if (!response.ok) throw new Error(data.error ?? 'No se pudo cargar la configuración de la cuota.')
  if (!response.ok || !data.configuracion) return
  monthlyFeeAmount.value = data.configuracion.montoCentavos / 100
  const nextMonth = nextMonthStart()
  if (data.configuracion.vigenteDesde >= nextMonth) monthlyFeeEffectiveMonth.value = data.configuracion.vigenteDesde.slice(0, 7)
}

function openMonthlyFeeDialog() {
  monthlyFeeMessage.value = ''
  monthlyFeeDialog.value?.showModal()
  void nextTick(() => monthlyFeeAmountInput.value?.focus())
}

function closeMonthlyFeeDialog() {
  monthlyFeeDialog.value?.close()
}

function onMonthlyFeeDialogClose() {
  monthlyFeeTrigger.value?.focus()
}

async function saveMonthlyFee() {
  monthlyFeeMessage.value = ''
  try {
    const response = await apiFetch(`${apiUrl}/admin/cuota-mensual`, {
    method: 'POST',
    credentials: 'include',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ montoPesos: monthlyFeeAmount.value, vigenteDesde: `${monthlyFeeEffectiveMonth.value}-01` }),
  })
  const data = await readApiJson<{ montoCentavos?: number; vigenteDesde?: string; error?: string }>(response)
  if (!response.ok || data.montoCentavos === undefined || !data.vigenteDesde) {
    monthlyFeeMessage.value = data.error ?? 'No se pudo guardar la cuota mensual.'
    return
  }
  monthlyFeeMessage.value = `Cuota mensual configurada: ${pesos(data.montoCentavos / 100)} desde ${formatMonth(data.vigenteDesde)}.`
    closeMonthlyFeeDialog()
  } catch (error) {
    monthlyFeeMessage.value = errorMessage(error, 'No se pudo guardar la cuota mensual.')
  }
}

function extractQuery(exporting = false) {
  const params = new URLSearchParams({
    fechaCorte: extractDate.value,
    situacion: extractSituation.value,
    busqueda: extractSearch.value,
    pagina: String(exporting ? 1 : extractPage.value),
    tamanoPagina: String(exporting ? 10_000 : 50),
  })
  if (exporting) params.set('exportar', '1')
  return params
}

async function loadExtract() {
  const requestId = ++extractRequest
  extractError.value = ''
  extractLoading.value = true
  try {
    const response = await apiFetch(`${apiUrl}/admin/extracto-cobranzas?${extractQuery()}`)
    const data = await readApiJson<ExtractResponse | { error: string }>(response)
    if (!response.ok || !('socios' in data)) throw new Error('error' in data ? data.error : 'No se pudo cargar el extracto.')
    if (requestId === extractRequest) extract.value = data
  } catch (error) {
    if (requestId === extractRequest) extractError.value = error instanceof Error ? error.message : 'No se pudo cargar el extracto.'
  } finally {
    if (requestId === extractRequest) extractLoading.value = false
  }
}

async function openAdmin() {
  currentView.value = 'admin'
  try {
    await Promise.all([loadExtract(), loadMonthlyFeeConfig()])
  } catch (error) {
    monthlyFeeMessage.value = errorMessage(error, 'No se pudo cargar la configuración de la cuota.')
  }
}

async function applyExtract() {
  clearTimeout(extractSearchTimer)
  extractPage.value = 1
  await loadExtract()
}

async function changeExtractPage(delta: number) {
  const nextPage = extractPage.value + delta
  if (nextPage < 1 || (extract.value && (nextPage - 1) * extract.value.tamanoPagina >= extract.value.total)) return
  extractPage.value = nextPage
  await loadExtract()
}

async function signOut() {
  try {
    await apiFetch(`${apiUrl}/auth/logout`, { method: 'POST' })
  } catch {
    // Clearing the local session is still safer than keeping a stale interface.
  }
  password.value = ''
  payments.value = []
  preparedPaymentOrder.value = null
  preparedPaymentMessage.value = ''
  extractRequest += 1
  extract.value = null
  currentView.value = 'account'
  signedIn.value = false
}

async function updateProfile() {
  profileMessage.value = ''
  try {
    const response = await apiFetch(`${apiUrl}/mi-perfil`, {
    method: 'PATCH',
    credentials: 'include',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      nombre: memberName.value,
      apellido: profileLastName.value,
      email: profileEmail.value,
      contrasenaActual: profileCurrentPassword.value || undefined,
    }),
  })
  const data = await readApiJson<{ usuario?: { nombre: string; apellido: string; email: string }; error?: string }>(response)
  if (!response.ok || !data.usuario) {
    profileMessage.value = data.error ?? 'No se pudieron actualizar tus datos.'
    return
  }
  memberName.value = data.usuario.nombre
  profileLastName.value = data.usuario.apellido
  profileEmail.value = data.usuario.email
  profileCurrentPassword.value = ''
    profileMessage.value = 'Datos actualizados.'
  } catch (error) {
    profileMessage.value = errorMessage(error, 'No se pudieron actualizar tus datos.')
  }
}

async function changePassword() {
  passwordMessage.value = ''
  try {
    const response = await apiFetch(`${apiUrl}/auth/cambiar-contrasena`, {
    method: 'POST',
    credentials: 'include',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      contrasenaActual: profileCurrentPassword.value,
      nuevaContrasena: profileNewPassword.value,
    }),
  })
  if (!response.ok) {
    const data = await readApiJson<{ error?: string }>(response)
    passwordMessage.value = data.error ?? 'No se pudo cambiar la contraseña.'
    return
  }
  profileCurrentPassword.value = ''
  profileNewPassword.value = ''
    passwordMessage.value = 'Contraseña actualizada. Las otras sesiones fueron cerradas.'
  } catch (error) {
    passwordMessage.value = errorMessage(error, 'No se pudo cambiar la contraseña.')
  }
}

type NativeField = HTMLInputElement | HTMLSelectElement | HTMLTextAreaElement

function nativeField(event: Event): NativeField | null {
  const target = event.target
  return target instanceof HTMLInputElement || target instanceof HTMLSelectElement || target instanceof HTMLTextAreaElement ? target : null
}

function translateNativeValidation(event: Event) {
  const field = nativeField(event)
  if (!field) return
  field.setCustomValidity('')
  const { validity } = field
  let message = ''
  if (validity.valueMissing) message = field instanceof HTMLInputElement && field.type === 'checkbox' ? 'Confirmá esta opción para continuar.' : 'Completá este campo.'
  else if (validity.typeMismatch) message = field instanceof HTMLInputElement && field.type === 'email' ? 'Ingresá un correo electrónico válido.' : 'Ingresá un valor válido.'
  else if (validity.tooShort) message = field instanceof HTMLSelectElement ? 'Ingresá un valor válido.' : `Ingresá al menos ${field.minLength} caracteres.`
  else if (validity.tooLong) message = field instanceof HTMLSelectElement ? 'Ingresá un valor válido.' : `Ingresá como máximo ${field.maxLength} caracteres.`
  else if (validity.rangeUnderflow) message = `Ingresá un valor igual o mayor a ${field instanceof HTMLInputElement ? field.min : ''}.`
  else if (validity.rangeOverflow) message = `Ingresá un valor igual o menor a ${field instanceof HTMLInputElement ? field.max : ''}.`
  else if (validity.stepMismatch || validity.badInput || validity.patternMismatch) message = 'Ingresá un valor válido.'
  field.setCustomValidity(message)
}

function clearNativeValidation(event: Event) {
  nativeField(event)?.setCustomValidity('')
}

onMounted(async () => {
  document.addEventListener('invalid', translateNativeValidation, true)
  document.addEventListener('input', clearNativeValidation, true)
  try {
  const response = await apiFetch(`${apiUrl}/auth/me`)
  const data = await readApiJson<{
    usuario?: { nombre: string; apellido: string; email: string; rol: 'socio' | 'administrador' }
    socio?: { numeroSocio: string }
  }>(response)
  if (!response.ok || !data.usuario) return
  memberName.value = data.usuario.nombre
  profileLastName.value = data.usuario.apellido
  profileEmail.value = data.usuario.email
  userRole.value = data.usuario.rol
  if (data.usuario.rol === 'administrador') {
    signedIn.value = true
    currentView.value = 'admin'
    await loadExtract()
    await loadMonthlyFeeConfig()
    return
  }
  if (!data.socio) return
  memberNumber.value = data.socio.numeroSocio
  try {
    await loadAccount()
    signedIn.value = true
  } catch (error) {
    loginError.value = errorMessage(error, 'No se pudo cargar tu cuenta.')
  }
  } catch (error) {
    loginError.value = errorMessage(error, 'No se pudo comprobar tu sesión. Intentá nuevamente.')
  }
})

watch([extractDate, extractSituation], () => {
  if (!signedIn.value || currentView.value !== 'admin') return
  clearTimeout(extractSearchTimer)
  extractPage.value = 1
  void loadExtract()
})

watch(extractSearch, () => {
  if (!signedIn.value || currentView.value !== 'admin') return
  clearTimeout(extractSearchTimer)
  extractSearchTimer = setTimeout(() => {
    extractPage.value = 1
    void loadExtract()
  }, 400)
})

onBeforeUnmount(() => {
  clearTimeout(extractSearchTimer)
  document.removeEventListener('invalid', translateNativeValidation, true)
  document.removeEventListener('input', clearNativeValidation, true)
})

function openPayment() {
  checkoutNotice.value = ''
  const preparedIds = preparedPaymentOrder.value?.cuotaIds.filter((id) => pendingPayments.value.some((payment) => payment.id === id)) ?? []
  selectedPaymentIds.value = preparedIds.length > 0 ? preparedIds : pendingPayments.value.map((payment) => payment.id)
  currentView.value = 'payment'
}

function selectPayments(type: 'mensual' | 'extra' | 'todos') {
  selectedPaymentIds.value = pendingPayments.value.filter((payment) => type === 'todos' || payment.type === type).map((payment) => payment.id)
  checkoutNotice.value = ''
}

async function continueToPayment() {
  if (selectedPayments.value.length === 0) {
    checkoutNotice.value = 'Elegí al menos un concepto para continuar.'
    return
  }
  checkoutPreparing.value = true
  checkoutNotice.value = ''
  try {
    const response = await apiFetch(`${apiUrl}/pagos/ordenes`, {
      method: 'POST',
      credentials: 'include',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        cuotaIds: selectedPayments.value.map((payment) => payment.id),
        canal: checkoutMethod.value === 'qr' ? 'qr' : 'checkout',
      }),
    })
    const data = await readApiJson<{ orden?: PreparedPaymentOrder; reutilizada?: boolean; siroDisponible?: boolean; error?: string }>(response)
    if (!response.ok || !data.orden) throw new Error(data.error ?? 'No se pudo preparar la orden de pago.')
    preparedPaymentOrder.value = data.orden
    if (!data.siroDisponible) {
      checkoutNotice.value = data.reutilizada
        ? 'Ya tenés este pago preparado. Todavía no se realizó ningún cobro.'
        : 'El pago quedó preparado. Todavía no se realizó ningún cobro.'
      return
    }
    checkoutNotice.value = data.siroDisponible
      ? 'La operación fue creada. Te estamos redirigiendo a SIRO.'
      : `Orden ${data.orden.id.slice(0, 8)} preparada. SIRO todavía no está configurado; no se realizó ningún cobro.`
  } catch (error) {
    checkoutNotice.value = error instanceof Error ? error.message : 'No se pudo preparar la orden de pago.'
  } finally {
    checkoutPreparing.value = false
  }
}

async function cancelPreparedPaymentOrder() {
  const order = preparedPaymentOrder.value
  if (!order || !confirm('¿Cancelar este pago preparado? No se realizará ningún cobro.')) return
  cancellingPreparedPayment.value = true
  preparedPaymentMessage.value = ''
  try {
    const response = await apiFetch(`${apiUrl}/pagos/ordenes/${order.id}/cancelar`, {
      method: 'POST',
      credentials: 'include',
    })
    if (!response.ok) {
      const data = await readApiJson<{ error?: string }>(response)
      throw new Error(data.error ?? 'No se pudo cancelar el pago preparado.')
    }
    preparedPaymentOrder.value = null
    preparedPaymentMessage.value = 'El pago preparado fue cancelado. No se realizó ningún cobro.'
  } catch (error) {
    preparedPaymentMessage.value = error instanceof Error ? error.message : 'No se pudo cancelar el pago preparado.'
  } finally {
    cancellingPreparedPayment.value = false
  }
}

function simulatePayment() {
  const pendingPayment = payments.value.find((payment) => payment.status === 'Pendiente')
  if (!pendingPayment) return

  pendingPayment.status = 'Pagado'
  confirmedAmount.value = pendingPayment.amount
  const receipt: Receipt = {
    number: '0001-00004618',
    date: '6 de octubre de 2026',
    concept: pendingPayment.concept,
    amount: pendingPayment.amount,
    method: paymentMethod.value === 'card' ? 'Tarjeta de crédito o débito' : 'Pago con QR',
  }
  receipts.value.unshift(receipt)
  selectedReceipt.value = receipt
  cardholder.value = ''
  cardNumber.value = ''
  cardExpiry.value = ''
  cardCvc.value = ''
  paymentComplete.value = true
}

function receiptText(receipt: Receipt) {
  return [
    'ADPUT - Comprobante de pago',
    `Recibo: ${receipt.number}`,
    `Fecha: ${receipt.date}`,
    'Socio: Marcos #1842',
    `Concepto: ${receipt.concept}`,
    `Medio de pago: ${receipt.method}`,
    `Importe abonado: ${pesos(receipt.amount)}`,
  ].join('\n')
}

function downloadReceipt() {
  const receipt = selectedReceipt.value
  const file = new Blob([receiptText(receipt)], { type: 'text/plain;charset=utf-8' })
  const url = URL.createObjectURL(file)
  const link = document.createElement('a')
  link.href = url
  link.download = `comprobante-${receipt.number}.txt`
  link.click()
  URL.revokeObjectURL(url)
  receiptActionMessage.value = 'Comprobante descargado.'
}

async function shareReceipt() {
  const receipt = selectedReceipt.value
  const text = receiptText(receipt)
  if (navigator.share) {
    try {
      await navigator.share({ title: `Comprobante ${receipt.number}`, text })
      receiptActionMessage.value = ''
    } catch {
      // The user may dismiss the native share sheet; that needs no error message.
    }
    return
  }

  if (navigator.clipboard) {
    try {
      await navigator.clipboard.writeText(text)
      receiptActionMessage.value = 'Tu navegador no permite compartir: el comprobante fue copiado.'
    } catch {
      receiptActionMessage.value = 'No se pudo copiar el comprobante desde este navegador.'
    }
    return
  }

  receiptActionMessage.value = 'Tu navegador no permite compartir ni copiar este comprobante.'
}

function argentinaDateInput() {
  const parts = new Intl.DateTimeFormat('en-US', {
    timeZone: 'America/Argentina/Buenos_Aires',
    year: 'numeric',
    month: '2-digit',
    day: '2-digit',
  }).formatToParts(new Date())
  const value = (type: string) => parts.find((part) => part.type === type)?.value
  return `${value('year')}-${value('month')}-${value('day')}`
}

function nextMonthStart() {
  const [year, month] = argentinaDateInput().split('-').map(Number)
  return new Date(Date.UTC(year!, month!, 1)).toISOString().slice(0, 10)
}

function situationLabel(situation: ExtractRow['situacion']) {
  if (situation === 'al_dia') return 'Al día'
  if (situation === 'debe_1') return 'Debe 1 cuota'
  return 'Debe 2 o más'
}

function exportRows(data: ExtractResponse) {
  return data.socios.map((row) => [
    row.socio.numeroSocio,
    `${row.socio.nombre} ${row.socio.apellido}`,
    situationLabel(row.situacion),
    pesos(row.importeCuotasAdeudadasCentavos / 100),
    pesos(row.importeExtrasAdeudadosCentavos / 100),
    row.ultimaCuotaPagadaPeriodo ? formatPeriod(row.ultimaCuotaPagadaPeriodo) : 'Sin cuotas pagadas',
  ])
}

async function downloadExtract() {
  extractExporting.value = 'xlsx'
  try {
    const response = await apiFetch(`${apiUrl}/admin/extracto-cobranzas?${extractQuery(true)}`)
    const data = await readApiJson<ExtractResponse | { error: string }>(response)
    if (!response.ok || !('socios' in data)) throw new Error('error' in data ? data.error : 'No se pudo preparar el extracto.')
    const rows = exportRows(data)
    const headers = ['N.º socio', 'Socio', 'Situación', 'Deuda cuotas', 'Extras adeudados', 'Última cuota pagada']
    const filename = `adput-extracto-${data.fechaCorte}`
    const XLSX = await import('xlsx')
    const sheet = XLSX.utils.aoa_to_sheet([
      [`ADPUT - Situación al ${data.fechaCorte}`],
      [`Al día: ${data.resumen.alDia}`, `Debe 1: ${data.resumen.debeUna}`, `Debe 2 o más: ${data.resumen.debeDosOMas}`],
      [],
      headers,
      ...rows,
    ])
    sheet['!cols'] = [14, 28, 16, 18, 18, 20].map((width) => ({ wch: width }))
    const workbook = XLSX.utils.book_new()
    XLSX.utils.book_append_sheet(workbook, sheet, 'Extracto')
    XLSX.writeFile(workbook, `${filename}.xlsx`)
  } catch (error) {
    extractError.value = error instanceof Error ? error.message : 'No se pudo generar el extracto.'
  } finally {
    extractExporting.value = null
  }
}
</script>

<template>
  <main v-if="!signedIn" class="login-page">
    <section class="login-card" aria-label="Ingreso a ADPUT">
      <img
        class="logo logo-login"
        src="/adput-logo.png"
        alt="ADPUT, Asociación Deportiva de Profesionales Universitarios de Tucumán"
      />
      <p class="intro">Ingresá para consultar tus pagos y el estado de tu cuenta.</p>

      <form @submit.prevent="signIn">
        <label for="email">Correo electrónico</label>
        <input
          id="email"
          v-model.trim="email"
          type="email"
          autocomplete="username"
          inputmode="email"
          maxlength="254"
          required
        />

        <label for="password">Contraseña</label>
        <input
          id="password"
          v-model="password"
          type="password"
          autocomplete="current-password"
          maxlength="1024"
          required
        />

        <p v-if="loginError" class="error" role="alert">{{ loginError }}</p>
        <button type="submit">Ingresar</button>
      </form>

      <p class="help">¿Necesitás acceso? Comunicate con secretaría de ADPUT.</p>
      <p class="prototype">Demo visual: no ingreses una contraseña real.</p>
    </section>
  </main>

  <main v-else class="app-page">
    <header>
      <img
        class="logo logo-header"
        src="/adput-logo.png"
        alt="ADPUT, Asociación Deportiva de Profesionales Universitarios de Tucumán"
      />
      <div class="header-actions">
        <button v-if="userRole === 'administrador'" class="link-button" type="button" @click="openAdmin">Administración</button>
        <button class="link-button" type="button" @click="currentView = 'profile'">Mi perfil</button>
        <button class="sign-out" type="button" @click="signOut">Cerrar sesión</button>
      </div>
    </header>

    <template v-if="currentView === 'account'">
      <section class="welcome" aria-labelledby="welcome-title">
        <div>
          <p class="eyebrow">MI CUENTA</p>
          <h1 id="welcome-title">Hola, {{ memberName }}</h1>
          <p>Socio #{{ memberNumber }}</p>
        </div>
        <div class="due-dates">
          <div class="due-date" :class="{ overdue: nextPendingMonthlyPayment?.overdue }">
            <span>{{ nextPendingMonthlyPayment?.overdue ? 'Cuota mensual vencida' : 'Próximo vencimiento de cuota' }}</span>
            <strong>{{ nextPendingMonthlyPayment ? formatDate(nextPendingMonthlyPayment.dueDate) : 'No hay cuotas pendientes' }}</strong>
            <small v-if="nextPendingMonthlyPayment">
              {{
                nextPendingMonthlyPayment.overdue
                  ? `Incluye recargo del 10%: ${pesos(nextPendingMonthlyPayment.lateFee)}`
                  : `Hasta esa fecha abonás ${pesos(nextPendingMonthlyPayment.baseAmount)}.`
              }}
            </small>
          </div>
          <div v-if="nextPendingExtraPayment" class="due-date extra-due-date" :class="{ overdue: nextPendingExtraPayment.overdue }">
            <span>{{ nextPendingExtraPayment.overdue ? 'Cargo extra vencido' : 'Vencimiento de cargo extra' }}</span>
            <strong>{{ formatDate(nextPendingExtraPayment.dueDate) }}</strong>
            <small>{{ nextPendingExtraPayment.concept }} · {{ pesos(nextPendingExtraPayment.amount) }}</small>
          </div>
        </div>
      </section>

      <section class="summary" aria-label="Account summary">
        <article class="balance" :class="{ clear: debt === 0 }">
          <p>Saldo actual</p>
          <strong>{{ debt === 0 ? 'Al día' : pesos(debt) }}</strong>
          <span>{{
            debt === 0 ? 'No registrás cuotas pendientes.' : 'Tenés cuotas pendientes de pago.'
          }}</span>
          <button v-if="debt > 0" class="pay-button" type="button" @click="openPayment">
            Pagar cuota
          </button>
        </article>
        <article>
          <p>Cuota mensual</p>
          <strong>{{ pesos(monthlyFee) }}</strong>
          <span>Del 1 al 10 de cada mes. Desde el 11 se aplica un recargo del 10%.</span>
        </article>
        <article>
          <p>Estado de cuenta</p>
          <strong class="status">Activa</strong>
          <span>Tu membresía está vigente.</span>
        </article>
      </section>

      <section v-if="preparedPaymentOrder || preparedPaymentMessage" class="prepared-payment" aria-live="polite">
        <template v-if="preparedPaymentOrder">
          <div>
            <p class="eyebrow">PAGO PREPARADO</p>
            <strong>Tenés un pago listo para continuar</strong>
            <p>
              {{ preparedPaymentOrder.items.length }} concepto(s) por {{ pesos(preparedPaymentOrder.montoCentavos / 100) }}. Todavía no se realizó ningún cobro.
              {{ preparedPaymentOrder.venceEn ? `Podés continuarlo hasta el ${formatDateTime(preparedPaymentOrder.venceEn)}.` : '' }}
            </p>
          </div>
          <div class="prepared-payment-actions">
            <button class="link-button" type="button" @click="openPayment">Revisar selección</button>
            <button class="prepared-payment-cancel" type="button" :disabled="cancellingPreparedPayment" @click="cancelPreparedPaymentOrder">
              {{ cancellingPreparedPayment ? 'Cancelando…' : 'Cancelar' }}
            </button>
          </div>
        </template>
        <p v-if="preparedPaymentMessage" class="profile-message" role="status">{{ preparedPaymentMessage }}</p>
      </section>

      <section class="content-grid">
        <section class="panel" aria-labelledby="payments-title">
          <div class="section-heading">
            <div>
              <p class="eyebrow">HISTORIAL</p>
              <h2 id="payments-title">Mis cuotas</h2>
            </div>
            <button class="link-button" type="button" @click="currentView = 'receipts'">
              Ver comprobantes
            </button>
          </div>
          <div class="payments" role="table" aria-label="Historial de pagos">
            <div v-for="payment in payments" :key="payment.concept" class="payment" role="row">
              <div role="cell">
                <strong>{{ payment.concept }}</strong
                ><span>{{ payment.date }}</span>
              </div>
              <div class="payment-right" role="cell">
                <strong>{{ pesos(payment.amount) }}</strong
                ><span :class="payment.status.toLowerCase()">{{ payment.status }}</span>
                <small v-if="payment.lateFee">Incluye recargo del 10%: {{ pesos(payment.lateFee) }}</small>
              </div>
            </div>
            <p v-if="payments.length === 0" class="empty-state">No hay cuotas emitidas para tu cuenta.</p>
          </div>
        </section>

        <aside class="panel payment-info" aria-labelledby="payment-info-title">
          <p class="eyebrow">PAGOS</p>
          <h2 id="payment-info-title">Cuota social</h2>
          <p>
            Consultá los medios de pago en secretaría. Tu pago aparecerá aquí cuando sea acreditado.
          </p>
          <p class="contact">Consultá en secretaría de ADPUT.</p>
        </aside>
      </section>
    </template>

    <section
      v-else-if="currentView === 'receipts'"
      class="receipts-view"
      aria-labelledby="receipts-title"
    >
      <button class="back-button" type="button" @click="currentView = 'account'">
        ← Volver a mi cuenta
      </button>
      <div class="receipts-heading">
        <div>
          <p class="eyebrow">HISTORIAL</p>
          <h1 id="receipts-title">Mis comprobantes</h1>
          <p>Seleccioná un comprobante para ver el detalle.</p>
        </div>
        <span class="receipt-count">{{ receipts.length }} disponibles</span>
      </div>

      <div v-if="receipts.length" class="receipts-layout">
        <nav class="receipt-list" aria-label="Lista de comprobantes">
          <button
            v-for="receipt in receipts"
            :key="receipt.number"
            class="receipt-item"
            :class="{ selected: selectedReceipt.number === receipt.number }"
            type="button"
            @click="selectedReceipt = receipt"
          >
            <span>{{ receipt.concept }}</span>
            <small>{{ receipt.date }}</small>
            <strong>{{ pesos(receipt.amount) }}</strong>
          </button>
        </nav>

        <article class="receipt-preview" aria-label="Vista previa del comprobante seleccionado">
          <div class="receipt-topline"><span>ADPUT</span><strong>COMPROBANTE DE PAGO</strong></div>
          <div class="receipt-title">
            <div>
              <p>RECIBO N°</p>
              <h2>{{ selectedReceipt.number }}</h2>
            </div>
            <p>{{ selectedReceipt.date }}</p>
          </div>
          <div class="receipt-member">
            <span>RECIBIDO DE</span>
            <strong>Marcos · Socio #1842</strong>
          </div>
          <div class="receipt-line">
            <span>Concepto</span><strong>{{ selectedReceipt.concept }}</strong>
          </div>
          <div class="receipt-line">
            <span>Medio de pago</span><strong>{{ selectedReceipt.method }}</strong>
          </div>
          <div class="receipt-total">
            <span>IMPORTE ABONADO</span><strong>{{ pesos(selectedReceipt.amount) }}</strong>
          </div>
          <p class="receipt-note">
            Este comprobante es una simulación para la demostración del sitio.
          </p>
          <div class="receipt-actions">
            <button type="button" @click="downloadReceipt">Descargar</button>
            <button class="share-button" type="button" @click="shareReceipt">Compartir</button>
          </div>
          <p v-if="receiptActionMessage" class="receipt-action-message" role="status">
            {{ receiptActionMessage }}
          </p>
        </article>
      </div>
      <p v-else class="empty-state">No hay comprobantes acreditados todavía.</p>
    </section>

    <section v-else-if="currentView === 'admin'" class="receipts-view" aria-labelledby="admin-title">
      <div class="receipts-heading">
        <div>
          <p class="eyebrow">ADMINISTRACIÓN</p>
          <h1 id="admin-title">Cobranzas</h1>
          <p>Consultá cuotas, cargos extra y comprobantes de todos los socios.</p>
        </div>
        <div class="admin-actions">
          <button ref="monthlyFeeTrigger" class="monthly-fee-button" type="button" @click="openMonthlyFeeDialog">
            Modificar cuota mensual
          </button>
          <button ref="extraChargeManagementTrigger" class="monthly-fee-button" type="button" @click="openExtraChargeManagementDialog">
            Gestionar cargos extras
          </button>
          <button ref="extraChargeTrigger" class="new-extra-charge" type="button" @click="openExtraChargeDialog">
            Nuevo cargo extra
          </button>
        </div>
      </div>
      <p v-if="extraChargeMessage" class="profile-message" role="status">{{ extraChargeMessage }}</p>
      <p v-if="monthlyFeeMessage" class="profile-message" role="status">{{ monthlyFeeMessage }}</p>

      <dialog ref="extraChargeDialog" class="extra-charge-dialog" aria-labelledby="extra-charge-title" @cancel.prevent="closeExtraChargeDialog" @close="onExtraChargeDialogClose">
        <form class="profile-form dialog-form" @submit.prevent="createExtraCharge">
          <p class="eyebrow">NUEVO CARGO</p>
          <h2 id="extra-charge-title">Crear cargo extra</h2>
          <p class="profile-help">Se guardará como borrador. No genera deuda hasta que lo confirmes desde la gestión de cargos.</p>
          <label for="extra-description">Concepto</label>
          <input id="extra-description" ref="extraChargeDescriptionInput" v-model.trim="extraChargeDescription" placeholder="Seguro - primer semestre" maxlength="120" required />
          <label for="extra-amount">Importe en pesos</label>
          <input id="extra-amount" v-model.number="extraChargeAmount" type="number" min="1" max="10000000" step="1" required />
          <label for="extra-due-date">Fecha de vencimiento</label>
          <input id="extra-due-date" v-model="extraChargeDueDate" type="date" required />
          <p v-if="extraChargeMessage" class="error" role="alert">{{ extraChargeMessage }}</p>
          <div class="dialog-actions">
            <button class="dialog-cancel" type="button" @click="closeExtraChargeDialog">Cancelar</button>
            <button type="submit">Crear cargo</button>
          </div>
        </form>
      </dialog>

      <dialog ref="extraChargeManagementDialog" class="extra-charge-dialog" aria-labelledby="extra-charge-management-title" @cancel.prevent="closeExtraChargeManagementDialog" @close="onExtraChargeManagementDialogClose">
        <section class="dialog-form">
          <p class="eyebrow">CARGOS EXTRA</p>
          <h2 id="extra-charge-management-title">Gestionar cargos extras</h2>
          <p class="profile-help">Podés editar o eliminar cargos sin pagos ni intentos de pago.</p>
          <p v-if="extraChargeManagementMessage" class="error" role="alert">{{ extraChargeManagementMessage }}</p>
          <div class="extra-charge-list">
            <article v-for="charge in extraCharges" :key="charge.id" class="extra-charge-item">
              <strong>{{ charge.descripcion }}</strong>
              <span>{{ pesos(charge.montoCentavos / 100) }} · vence {{ formatDate(charge.fechaVencimiento) }}</span>
              <small>{{ charge.estado === 'borrador' ? 'Borrador: todavía no genera deuda' : `${charge.sociosAlcanzados} socio(s) · aplicado` }}{{ charge.bloqueado ? ' · no editable: hay pagos o intentos de pago' : '' }}</small>
              <div v-if="!charge.bloqueado" class="extra-charge-item-actions">
                <button v-if="charge.estado === 'borrador'" type="button" @click="startApplyingExtraCharge(charge)">Aplicar a todos</button>
                <button class="dialog-cancel" type="button" @click="editExtraCharge(charge)">Editar</button>
                <button class="delete-charge" type="button" @click="deleteExtraCharge(charge)">Eliminar</button>
              </div>
            </article>
            <p v-if="extraCharges.length === 0" class="empty-state">No hay cargos extra creados.</p>
          </div>
          <form v-if="applyingExtraCharge" class="profile-form extra-charge-edit" @submit.prevent="applyExtraCharge">
            <h3>Aplicar cargo a todos los socios</h3>
            <p class="profile-help"><strong>{{ applyingExtraCharge.descripcion }}</strong> generará una deuda de {{ pesos(applyingExtraCharge.montoCentavos / 100) }} para todos los socios activos.</p>
            <label class="confirmation-check"><input v-model="extraChargeApplicationConfirmed" type="checkbox" required /> Confirmo que revisé el concepto, importe y vencimiento.</label>
            <div class="dialog-actions">
              <button class="dialog-cancel" type="button" @click="applyingExtraCharge = null; extraChargeApplicationConfirmed = false">Cancelar</button>
              <button type="submit" :disabled="!extraChargeApplicationConfirmed">Aplicar cargo</button>
            </div>
          </form>
          <form v-if="editingExtraCharge" class="profile-form extra-charge-edit" @submit.prevent="saveExtraCharge">
            <h3>Editar cargo extra</h3>
            <label for="edit-extra-description">Concepto</label>
            <input id="edit-extra-description" v-model.trim="editingExtraCharge.descripcion" maxlength="120" required />
            <label for="edit-extra-amount">Importe en pesos</label>
            <input id="edit-extra-amount" v-model.number="editingExtraCharge.montoPesos" type="number" min="1" max="10000000" step="1" required />
            <label for="edit-extra-due-date">Fecha de vencimiento</label>
            <input id="edit-extra-due-date" v-model="editingExtraCharge.fechaVencimiento" type="date" required />
            <div class="dialog-actions">
              <button class="dialog-cancel" type="button" @click="editingExtraCharge = null">Cancelar</button>
              <button type="submit">Guardar cambios</button>
            </div>
          </form>
          <div class="dialog-actions">
            <button class="dialog-cancel" type="button" @click="closeExtraChargeManagementDialog">Cerrar</button>
          </div>
        </section>
      </dialog>

      <dialog ref="monthlyFeeDialog" class="extra-charge-dialog" aria-labelledby="monthly-fee-title" @cancel.prevent="closeMonthlyFeeDialog" @close="onMonthlyFeeDialogClose">
        <form class="profile-form dialog-form" @submit.prevent="saveMonthlyFee">
          <p class="eyebrow">CONFIGURACIÓN</p>
          <h2 id="monthly-fee-title">Modificar cuota mensual</h2>
          <p class="profile-help">El nuevo importe se usará desde el mes elegido. Las cuotas ya emitidas no cambian.</p>
          <label for="monthly-fee-amount">Importe mensual en pesos</label>
          <input id="monthly-fee-amount" ref="monthlyFeeAmountInput" v-model.number="monthlyFeeAmount" type="number" min="1" max="10000000" step="1" required />
          <label for="monthly-fee-effective">Vigente desde</label>
          <input id="monthly-fee-effective" v-model="monthlyFeeEffectiveMonth" type="month" :min="nextMonthStart().slice(0, 7)" required />
          <p v-if="monthlyFeeMessage" class="error" role="alert">{{ monthlyFeeMessage }}</p>
          <div class="dialog-actions">
            <button class="dialog-cancel" type="button" @click="closeMonthlyFeeDialog">Cancelar</button>
            <button type="submit">Guardar importe</button>
          </div>
        </form>
      </dialog>

      <section class="extract-section" aria-labelledby="extract-title">
        <div class="section-heading">
          <div>
            <p class="eyebrow">EXTRACTO</p>
            <h2 id="extract-title">Situación de socios</h2>
          </div>
        </div>
        <form class="extract-filters" @submit.prevent="applyExtract">
          <label>Fecha de corte<input v-model="extractDate" type="date" required /></label>
          <label>Situación
            <select v-model="extractSituation">
              <option value="todos">Todos los socios</option>
              <option value="al_dia">Al día</option>
              <option value="debe_1">Debe 1 cuota</option>
              <option value="debe_2_mas">Debe 2 o más</option>
            </select>
          </label>
          <label class="extract-search">Buscar socio o número<input v-model.trim="extractSearch" type="search" maxlength="100" placeholder="Nombre o N.º de socio" /></label>
          <button type="submit" :disabled="extractLoading">Actualizar</button>
        </form>

        <div v-if="extract" class="extract-summary" aria-label="Resumen del extracto">
          <article><span>Al día</span><strong>{{ extract.resumen.alDia }}</strong></article>
          <article><span>Debe 1 cuota</span><strong>{{ extract.resumen.debeUna }}</strong></article>
          <article><span>Debe 2 o más</span><strong>{{ extract.resumen.debeDosOMas }}</strong></article>
          <article><span>Deuda cuotas</span><strong>{{ pesos(extract.resumen.deudaCuotasCentavos / 100) }}</strong></article>
        </div>

        <div class="extract-actions">
          <span v-if="extract">{{ extract.total }} socio(s) según la fecha de corte.</span>
          <div>
            <button type="button" :disabled="extractExporting !== null" @click="downloadExtract">{{ extractExporting === 'xlsx' ? 'Generando…' : 'Descargar XLSX' }}</button>
          </div>
        </div>
        <p v-if="extractError" class="error" role="alert">{{ extractError }}</p>
        <p v-else-if="extractLoading" class="empty-state">Calculando extracto…</p>
        <div v-else-if="extract?.socios.length" class="extract-table-wrap">
          <table class="extract-table">
            <thead><tr><th>N.º socio</th><th>Socio</th><th>Situación</th><th>Deuda cuotas</th><th>Extras adeudados</th><th>Última cuota pagada</th></tr></thead>
            <tbody>
              <tr v-for="row in extract.socios" :key="row.socio.numeroSocio">
                <td>{{ row.socio.numeroSocio }}</td>
                <td><strong>{{ row.socio.nombre }} {{ row.socio.apellido }}</strong></td>
                <td><span class="extract-status" :class="row.situacion">{{ situationLabel(row.situacion) }}</span></td>
                <td>{{ pesos(row.importeCuotasAdeudadasCentavos / 100) }}</td>
                <td>{{ pesos(row.importeExtrasAdeudadosCentavos / 100) }}</td>
                <td>{{ row.ultimaCuotaPagadaPeriodo ? formatPeriod(row.ultimaCuotaPagadaPeriodo) : 'Sin cuotas pagadas' }}</td>
              </tr>
            </tbody>
          </table>
        </div>
        <p v-else-if="extract" class="empty-state">No hay socios que coincidan con estos filtros.</p>
        <div v-if="extract && extract.total > extract.tamanoPagina" class="extract-pagination">
          <button type="button" :disabled="extract.pagina === 1 || extractLoading" @click="changeExtractPage(-1)">Anterior</button>
          <span>Página {{ extract.pagina }} de {{ Math.ceil(extract.total / extract.tamanoPagina) }}</span>
          <button type="button" :disabled="extract.pagina * extract.tamanoPagina >= extract.total || extractLoading" @click="changeExtractPage(1)">Siguiente</button>
        </div>
      </section>
    </section>

    <section v-else-if="currentView === 'profile'" class="receipts-view" aria-labelledby="profile-title">
      <button class="back-button" type="button" @click="currentView = 'account'">← Volver a mi cuenta</button>
      <div class="receipts-heading">
        <div>
          <p class="eyebrow">MI PERFIL</p>
          <h1 id="profile-title">Datos personales</h1>
          <p>Actualizá tus datos o cambiá tu contraseña.</p>
        </div>
      </div>
      <div class="profile-grid">
        <form class="panel profile-form" @submit.prevent="updateProfile">
          <h2>Datos de contacto</h2>
          <label for="profile-name">Nombre</label>
          <input id="profile-name" v-model.trim="memberName" autocomplete="given-name" maxlength="100" required />
          <label for="profile-last-name">Apellido</label>
          <input id="profile-last-name" v-model.trim="profileLastName" autocomplete="family-name" maxlength="100" required />
          <label for="profile-email">Correo electrónico</label>
          <input id="profile-email" v-model.trim="profileEmail" type="email" autocomplete="email" maxlength="254" required />
          <label for="profile-password">Contraseña actual <small>(solo si cambiás el correo)</small></label>
          <input id="profile-password" v-model="profileCurrentPassword" type="password" autocomplete="current-password" maxlength="1024" />
          <button type="submit">Guardar datos</button>
        </form>
        <form class="panel profile-form" @submit.prevent="changePassword">
          <h2>Cambiar contraseña</h2>
          <label for="current-password">Contraseña actual</label>
          <input id="current-password" v-model="profileCurrentPassword" type="password" autocomplete="current-password" maxlength="1024" required />
          <label for="new-password">Nueva contraseña</label>
          <input id="new-password" v-model="profileNewPassword" type="password" autocomplete="new-password" minlength="8" maxlength="1024" required />
          <p class="profile-help">Usá al menos 8 caracteres. Se cerrarán las demás sesiones activas.</p>
          <button type="submit">Cambiar contraseña</button>
          <p v-if="passwordMessage" class="profile-message" role="status" aria-live="polite">{{ passwordMessage }}</p>
        </form>
      </div>
      <p v-if="profileMessage" class="profile-message" role="status">{{ profileMessage }}</p>
    </section>

    <section v-else-if="false" class="payment-view" aria-labelledby="payment-title">
      <button class="back-button" type="button" @click="currentView = 'account'">
        ← Volver a mi cuenta
      </button>
      <div class="payment-layout">
        <section>
          <p class="eyebrow">PAGO DE CUOTA</p>
          <h1 id="payment-title">Abonar cuota de octubre</h1>
          <p class="payment-intro">Elegí cómo querés abonar tu cuota.</p>

          <template v-if="!paymentComplete">
            <div class="payment-methods" role="radiogroup" aria-label="Medio de pago">
              <button
                class="payment-method"
                :class="{ selected: paymentMethod === 'card' }"
                type="button"
                role="radio"
                :aria-checked="paymentMethod === 'card'"
                @click="paymentMethod = 'card'"
              >
                <strong>Tarjeta</strong><span>Crédito o débito</span>
              </button>
              <button
                class="payment-method"
                :class="{ selected: paymentMethod === 'qr' }"
                type="button"
                role="radio"
                :aria-checked="paymentMethod === 'qr'"
                @click="paymentMethod = 'qr'"
              >
                <strong>QR</strong><span>Escaneá desde tu billetera</span>
              </button>
            </div>

            <form
              v-if="paymentMethod === 'card'"
              class="card-form"
              @submit.prevent="simulatePayment"
            >
              <label for="cardholder">Titular de la tarjeta</label>
              <input id="cardholder" v-model.trim="cardholder" autocomplete="cc-name" required />

              <label for="card-number">Número de tarjeta</label>
              <input
                id="card-number"
                v-model="cardNumber"
                inputmode="numeric"
                autocomplete="cc-number"
                placeholder="0000 0000 0000 0000"
                required
              />

              <div class="card-details">
                <div>
                  <label for="card-expiry">Vencimiento</label>
                  <input
                    id="card-expiry"
                    v-model="cardExpiry"
                    inputmode="numeric"
                    autocomplete="cc-exp"
                    placeholder="MM/AA"
                    required
                  />
                </div>
                <div>
                  <label for="card-cvc">Código de seguridad</label>
                  <input
                    id="card-cvc"
                    v-model="cardCvc"
                    inputmode="numeric"
                    autocomplete="cc-csc"
                    placeholder="123"
                    required
                  />
                </div>
              </div>

              <button type="submit">Confirmar pago de {{ pesos(debt) }}</button>
              <p class="card-disclaimer">Demo visual: no ingreses datos reales de tarjeta.</p>
            </form>

            <section v-else class="qr-payment" aria-label="Pago con código QR">
              <div class="qr-demo" aria-hidden="true"><span>ADPUT</span><strong>QR</strong></div>
              <div>
                <h2>Escaneá el QR</h2>
                <p>
                  Abrí la billetera virtual que uses y escaneá este código para pagar
                  {{ pesos(debt) }}.
                </p>
                <button type="button" @click="simulatePayment">Ya realicé el pago</button>
                <small>Demo visual: el QR no procesa pagos reales.</small>
              </div>
            </section>
          </template>

          <div v-else class="payment-success" role="status">
            <p class="eyebrow">PAGO CONFIRMADO</p>
            <h2>¡Listo, Marcos!</h2>
            <p>Tu cuota de octubre fue abonada. Ya podés ver el comprobante.</p>
            <div>
              <button type="button" @click="currentView = 'receipts'">Ver comprobante</button>
              <button class="secondary-button" type="button" @click="currentView = 'account'">
                Ir a mi cuenta
              </button>
            </div>
          </div>
        </section>

        <aside class="payment-summary" aria-label="Resumen del pago">
          <p class="eyebrow">RESUMEN</p>
          <h2>Cuota social</h2>
          <p>Octubre 2026</p>
          <div>
            <span>Total a pagar</span
            ><strong>{{ pesos(paymentComplete ? confirmedAmount : debt) }}</strong>
          </div>
          <small
            >El pago real se integrará con un procesador de pagos seguro durante la siguente
            etapa.</small
          >
        </aside>
      </div>
    </section>
    <section v-else class="payment-view" aria-labelledby="payment-title">
      <button class="back-button" type="button" @click="currentView = 'account'">← Volver a mi cuenta</button>
      <div class="payment-layout">
        <section>
          <p class="eyebrow">PAGO DE CUOTA</p>
          <h1 id="payment-title">Prepará tu pago online</h1>
          <p class="payment-intro">
            Elegí los conceptos a abonar. Podés pagar uno, varios meses, un extra o cualquier combinación pendiente.
          </p>
          <section class="payment-selection" aria-labelledby="payment-concepts-title">
            <div class="section-heading">
              <div>
                <p class="eyebrow">CONCEPTOS</p>
                <h2 id="payment-concepts-title">¿Qué querés abonar?</h2>
              </div>
            </div>
            <div class="payment-selection-actions" aria-label="Selección rápida">
              <button type="button" @click="selectPayments('mensual')">Solo cuotas</button>
              <button type="button" @click="selectPayments('extra')">Solo extras</button>
              <button type="button" @click="selectPayments('todos')">Todo pendiente</button>
            </div>
            <p class="profile-help">Podés ajustar la selección antes de preparar el pago.</p>
            <label v-for="payment in pendingPayments" :key="payment.id" class="payment-concept" :class="{ selected: selectedPaymentIds.includes(payment.id) }">
              <input v-model="selectedPaymentIds" type="checkbox" :value="payment.id" />
              <span>
                <strong>{{ payment.type === 'mensual' ? 'Cuota mensual' : 'Cargo extra' }}</strong>
                <small>{{ payment.concept }} · vence {{ formatDate(payment.dueDate) }}{{ payment.overdue ? ' · vencido' : '' }}</small>
              </span>
              <strong>{{ pesos(payment.amount) }}</strong>
            </label>
            <p v-if="pendingPayments.length === 0" class="empty-state">No tenés conceptos pendientes.</p>
          </section>
          <div class="payment-methods" role="radiogroup" aria-label="Medio de pago">
            <button
              class="payment-method"
              :class="{ selected: checkoutMethod === 'credito' }"
              type="button"
              role="radio"
              :aria-checked="checkoutMethod === 'credito'"
              @click="checkoutMethod = 'credito'"
            >
              <strong>Tarjeta de crédito</strong><span>Pago seguro con SIRO</span>
            </button>
            <button
              class="payment-method"
              :class="{ selected: checkoutMethod === 'debito' }"
              type="button"
              role="radio"
              :aria-checked="checkoutMethod === 'debito'"
              @click="checkoutMethod = 'debito'"
            >
              <strong>Tarjeta de débito</strong><span>Pago seguro con SIRO</span>
            </button>
            <button
              class="payment-method"
              :class="{ selected: checkoutMethod === 'qr' }"
              type="button"
              role="radio"
              :aria-checked="checkoutMethod === 'qr'"
              @click="checkoutMethod = 'qr'"
            >
              <strong>Pago con QR</strong><span>Escaneá desde tu billetera</span>
            </button>
          </div>
          <button type="button" :disabled="selectedPayments.length === 0 || checkoutPreparing" @click="continueToPayment">{{ checkoutPreparing ? 'Preparando operación…' : 'Preparar pago con SIRO' }}</button>
          <p v-if="checkoutNotice" class="profile-message" role="status">{{ checkoutNotice }}</p>
        </section>
        <aside class="payment-summary" aria-label="Resumen del pago">
          <p class="eyebrow">RESUMEN</p>
          <h2>{{ selectedPayments.length === 1 ? selectedPayments[0]?.concept : `${selectedPayments.length} conceptos seleccionados` }}</h2>
          <div><span>Total a pagar</span><strong>{{ pesos(selectedPaymentAmount) }}</strong></div>
        </aside>
      </div>
    </section>
  </main>
</template>

<style>
:root {
  color: #102f56;
  background: #f5f2e8;
  font-family: 'Segoe UI', ui-sans-serif, system-ui, sans-serif;
}

* {
  box-sizing: border-box;
}

body {
  margin: 0;
  min-width: 320px;
}

button,
input {
  font: inherit;
}

button {
  cursor: pointer;
}

.login-page {
  height: 100vh;
  height: 100dvh;
  display: grid;
  place-items: center;
  overflow-y: auto;
  padding: 24px;
  background:
    radial-gradient(circle at 80% 12%, #c9dfef 0, transparent 28rem),
    linear-gradient(135deg, #082b53, #124d80);
}

.login-card {
  width: min(100%, 430px);
  padding: 44px;
  background: #fffdf6;
  border: 6px solid #df362c;
  box-shadow: 0 20px 60px #05192c66;
}

.logo {
  display: block;
  mix-blend-mode: multiply;
  object-fit: cover;
  object-position: center;
}

.logo-login {
  width: 300px;
  max-width: 100%;
  height: 165px;
  margin: -6px auto 16px;
}

.logo-header {
  width: 175px;
  height: 100px;
}

h1,
h2 {
  margin: 0;
  font-family: Bahnschrift, 'Arial Narrow', 'Segoe UI', sans-serif;
  font-weight: 700;
  letter-spacing: -0.02em;
}

h1 {
  font-size: clamp(2.15rem, 7vw, 3.25rem);
  line-height: 1.02;
  margin-top: 28px;
}

h2 {
  font-size: 1.55rem;
}

.intro {
  line-height: 1.55;
  color: #536775;
  margin: 18px 0 30px;
}

form {
  display: grid;
  gap: 9px;
}

label {
  font-size: 0.88rem;
  font-weight: 600;
  margin-top: 8px;
}

input {
  width: 100%;
  min-height: 46px;
  padding: 10px 12px;
  border: 1px solid #aab9c6;
  border-radius: 4px;
  background: #fff;
  color: #102f56;
}

input:focus {
  outline: 3px solid #b9d6ec;
  border-color: #0b3e6d;
}

form button {
  min-height: 48px;
  margin-top: 14px;
  border: 0;
  border-radius: 4px;
  background: #df362c;
  color: white;
  font-weight: 700;
}

form button:hover {
  background: #b72026;
}

.error {
  margin: 8px 0 0;
  color: #b72026;
  font-size: 0.9rem;
}

.help,
.prototype {
  font-size: 0.84rem;
  line-height: 1.45;
  color: #607285;
}

.help {
  margin: 28px 0 6px;
}

.prototype {
  margin: 0;
}

.app-page {
  max-width: 1120px;
  margin: auto;
  padding: 24px 30px 60px;
}

header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  min-height: 104px;
}

.header-actions {
  display: flex;
  align-items: center;
  gap: 18px;
}

.sign-out,
.link-button {
  border: 0;
  background: transparent;
  color: #0b3e6d;
  font-weight: 700;
}

.link-button:disabled {
  color: #8a99a5;
  cursor: not-allowed;
}

.welcome {
  display: flex;
  justify-content: space-between;
  align-items: end;
  gap: 20px;
  margin: 70px 0 32px;
}

.welcome h1 {
  margin: 8px 0;
}

.welcome p:not(.eyebrow) {
  margin: 0;
  color: #5b6f82;
}

.eyebrow {
  margin: 0;
  color: #58718b;
  font-size: 0.72rem;
  font-weight: 700;
  letter-spacing: 0.13em;
}

.due-date {
  min-width: 170px;
  padding: 15px 20px;
  border-left: 4px solid #df362c;
  background: #e7f0f8;
}

.due-dates {
  display: flex;
  flex-wrap: wrap;
  justify-content: end;
  gap: 10px;
}

.extra-due-date {
  border-left-color: #58718b;
  background: #f1f5f8;
}

.due-date.overdue {
  border-left-color: #9a3c35;
  background: #fae9e7;
}

.due-date small {
  display: block;
  margin-top: 6px;
  color: #536775;
  line-height: 1.35;
}

.due-date span,
.due-date strong {
  display: block;
}

.due-date span {
  color: #33516e;
  font-size: 0.78rem;
}

.due-date strong {
  margin-top: 3px;
  font-size: 1.1rem;
}

.summary {
  display: grid;
  grid-template-columns: 1.25fr 1fr 1fr;
  gap: 16px;
}

.summary article,
.panel {
  border: 1px solid #d8e0e6;
  background: #fffdf6;
}

.summary article {
  min-height: 164px;
  padding: 22px;
}

.summary p {
  margin: 0;
  color: #58718b;
  font-size: 0.87rem;
}

.summary strong {
  display: block;
  margin: 14px 0 5px;
  font-size: 1.55rem;
}

.summary span {
  color: #5b6f82;
  font-size: 0.84rem;
}

.balance {
  background: #0b3e6d !important;
  border: none !important;
  color: #fff;
}

.balance p,
.balance span {
  color: #e9f2fa;
}

.balance.clear {
  background: #0d5b78 !important;
}

.pay-button {
  width: 100%;
  min-height: 42px;
  margin-top: 18px;
  border: 0;
  border-radius: 4px;
  background: #df362c;
  color: #fff;
  font-weight: 700;
}

.pay-button:hover {
  background: #b72026;
}

.status {
  color: #16704c;
}

.content-grid {
  display: grid;
  grid-template-columns: minmax(0, 1.7fr) minmax(260px, 0.8fr);
  gap: 16px;
  margin-top: 16px;
}

.prepared-payment {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 18px;
  margin-top: 16px;
  padding: 18px 22px;
  border-left: 4px solid #0d5b78;
  background: #e7f0f8;
}

.prepared-payment p {
  margin: 4px 0 0;
  color: #36516c;
}

.prepared-payment-actions {
  display: flex;
  flex: none;
  gap: 12px;
  align-items: center;
}

.prepared-payment-cancel {
  min-height: 40px;
  border: 1px solid #9a3c35;
  border-radius: 4px;
  background: transparent;
  color: #9a3c35;
  font-weight: 700;
  padding: 0 12px;
}

.panel {
  padding: 26px;
}

.section-heading {
  display: flex;
  justify-content: space-between;
  gap: 18px;
  align-items: start;
}

.section-heading h2 {
  margin-top: 6px;
}

.payments {
  margin-top: 18px;
}

.payment {
  display: flex;
  justify-content: space-between;
  gap: 16px;
  padding: 17px 0;
  border-top: 1px solid #dde4e9;
}

.payment strong,
.payment span {
  display: block;
}

.payment span {
  margin-top: 4px;
  color: #607285;
  font-size: 0.84rem;
}

.payment-right {
  text-align: right;
}

.payment-right small {
  display: block;
  margin-top: 5px;
  color: #9a3c35;
}

.payment-right .pagado {
  color: #16704c;
}

.payment-right .pendiente {
  color: #c42a26;
}

.payment-right .vencida {
  color: #9a3c35;
}

.payment-info {
  align-self: start;
  background: #e7f0f8;
  border-color: #c4dae9;
}

.payment-info h2 {
  margin: 8px 0 15px;
}

.payment-info > p:not(.eyebrow) {
  margin: 0 0 22px;
  color: #36516c;
  line-height: 1.55;
}

.contact {
  color: #0b3e6d;
  font-weight: 700;
}

.receipts-view {
  margin-top: 38px;
}

.profile-grid {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 20px;
}

.profile-form h2 {
  margin-bottom: 12px;
}

.profile-form small,
.profile-help,
.profile-message {
  color: #536775;
}

.profile-help,
.profile-message {
  line-height: 1.5;
}

.new-extra-charge {
  min-height: 44px;
  border: 0;
  border-radius: 4px;
  background: #df362c;
  color: #fff;
  font-weight: 700;
  padding: 0 16px;
}

.admin-actions {
  display: flex;
  flex-wrap: wrap;
  justify-content: end;
  gap: 10px;
}

.monthly-fee-button {
  min-height: 44px;
  border: 1px solid #0b3e6d;
  border-radius: 4px;
  background: transparent;
  color: #0b3e6d;
  font-weight: 700;
  padding: 0 16px;
}

.extra-charge-dialog {
  width: min(100% - 32px, 520px);
  padding: 0;
  border: 0;
  border-radius: 6px;
  background: #fffdf6;
  box-shadow: 0 24px 64px rgb(16 47 86 / 0.28);
}

.extra-charge-dialog::backdrop {
  background: rgb(11 62 109 / 0.55);
}

.dialog-form {
  padding: 28px;
}

.dialog-form .profile-help {
  margin: 0 0 20px;
}

.extra-charge-list {
  display: grid;
  gap: 10px;
  max-height: 42dvh;
  overflow: auto;
}

.extra-charge-item {
  display: grid;
  gap: 4px;
  padding: 14px;
  border: 1px solid #d8e0e6;
}

.extra-charge-item span,
.extra-charge-item small {
  color: #536775;
  font-size: 0.84rem;
}

.extra-charge-item-actions {
  display: flex;
  gap: 10px;
  margin-top: 8px;
}

.extra-charge-item-actions button {
  min-height: 38px;
  border: 0;
  border-radius: 4px;
  background: #0b3e6d;
  color: #fff;
  font-weight: 700;
  padding: 0 12px;
}

.extra-charge-item-actions .dialog-cancel {
  border: 1px solid #0b3e6d;
  background: transparent;
  color: #0b3e6d;
}

.extra-charge-item-actions .delete-charge {
  border: 0;
  background: #b72026;
  color: #fff;
}

.extra-charge-edit {
  margin-top: 20px;
  padding-top: 20px;
  border-top: 1px solid #d8e0e6;
}

.confirmation-check {
  display: flex;
  align-items: start;
  gap: 10px;
  line-height: 1.4;
}

.confirmation-check input {
  appearance: auto;
  width: 20px;
  min-width: 20px;
  min-height: 20px;
  margin: 0;
  padding: 0;
  accent-color: #0b3e6d;
}

.dialog-actions {
  display: flex;
  justify-content: end;
  gap: 10px;
  margin-top: 24px;
}

.dialog-actions button {
  min-height: 44px;
  margin-top: 0;
  padding: 0 16px;
}

.dialog-actions .dialog-cancel {
  border: 1px solid #0b3e6d;
  background: transparent;
  color: #0b3e6d;
}

.back-button {
  border: 0;
  background: transparent;
  color: #0b3e6d;
  font-weight: 700;
  padding: 0;
}

.receipts-heading {
  display: flex;
  justify-content: space-between;
  align-items: end;
  gap: 20px;
  margin: 28px 0;
}

.receipts-heading h1 {
  margin: 6px 0 8px;
}

.receipts-heading p:not(.eyebrow) {
  margin: 0;
  color: #5b6f82;
}

.receipt-count {
  padding: 8px 12px;
  border-radius: 999px;
  background: #e7f0f8;
  color: #33516e;
  font-size: 0.82rem;
  font-weight: 700;
  white-space: nowrap;
}

.receipts-layout {
  display: grid;
  grid-template-columns: minmax(250px, 0.8fr) minmax(0, 1.45fr);
  gap: 16px;
  align-items: start;
}

.receipt-list,
.receipt-preview {
  border: 1px solid #d8e0e6;
  background: #fffdf6;
}

.receipt-list {
  padding: 8px;
}

.receipt-item {
  display: grid;
  width: 100%;
  gap: 4px;
  padding: 16px;
  border: 0;
  border-left: 4px solid transparent;
  background: transparent;
  color: #102f56;
  text-align: left;
}

.receipt-item + .receipt-item {
  border-top: 1px solid #dde4e9;
}

.receipt-item:hover,
.receipt-item.selected {
  border-left-color: #df362c;
  background: #e7f0f8;
}

.receipt-item small {
  color: #607285;
}

.receipt-item strong {
  margin-top: 4px;
}

.receipt-preview {
  min-height: 475px;
  padding: 34px;
  box-shadow: 12px 12px 0 #e7f0f8;
}

.receipt-topline {
  display: flex;
  justify-content: space-between;
  padding-bottom: 18px;
  border-bottom: 3px solid #0b3e6d;
  color: #0b3e6d;
  font-size: 0.78rem;
  letter-spacing: 0.08em;
}

.receipt-topline span {
  color: #df362c;
  font-family: Bahnschrift, 'Arial Narrow', sans-serif;
  font-size: 1.25rem;
  font-weight: 800;
}

.receipt-title {
  display: flex;
  justify-content: space-between;
  gap: 16px;
  align-items: start;
  padding: 26px 0;
}

.receipt-title p,
.receipt-member span,
.receipt-total span {
  margin: 0;
  color: #58718b;
  font-size: 0.72rem;
  font-weight: 700;
  letter-spacing: 0.1em;
}

.receipt-title h2 {
  margin-top: 5px;
}

.receipt-member {
  display: grid;
  gap: 5px;
  padding: 18px 0;
  border-top: 1px solid #dde4e9;
  border-bottom: 1px solid #dde4e9;
}

.receipt-line,
.receipt-total {
  display: flex;
  justify-content: space-between;
  gap: 20px;
  padding: 18px 0;
}

.receipt-line {
  border-bottom: 1px solid #dde4e9;
}

.receipt-line span {
  color: #5b6f82;
}

.receipt-line strong {
  text-align: right;
}

.receipt-total {
  align-items: end;
  margin-top: 8px;
}

.receipt-total strong {
  color: #0b3e6d;
  font-size: 1.8rem;
}

.receipt-note {
  margin: 12px 0 0;
  color: #607285;
  font-size: 0.78rem;
}

.receipt-actions {
  display: flex;
  gap: 10px;
  margin-top: 22px;
}

.receipt-actions button {
  min-height: 40px;
  border: 0;
  border-radius: 4px;
  background: #0b3e6d;
  color: #fff;
  font-weight: 700;
  padding: 0 14px;
}

.receipt-actions .share-button {
  border: 1px solid #0b3e6d;
  background: transparent;
  color: #0b3e6d;
}

.receipt-action-message {
  margin: 12px 0 0;
  color: #16704c;
  font-size: 0.78rem;
}

.payment-view {
  margin-top: 38px;
}

.payment-layout {
  display: grid;
  grid-template-columns: minmax(0, 1.35fr) minmax(260px, 0.65fr);
  gap: 48px;
  margin-top: 28px;
}

.payment-layout h1 {
  margin: 6px 0 10px;
}

.payment-intro {
  margin: 0;
  color: #5b6f82;
}

.payment-selection {
  max-width: 560px;
  margin-top: 26px;
}

.payment-selection h2 {
  margin-top: 6px;
}

.payment-selection-actions {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
  margin: 14px 0 10px;
}

.payment-selection-actions button {
  min-height: 38px;
  border: 1px solid #0b3e6d;
  border-radius: 4px;
  background: transparent;
  color: #0b3e6d;
  font-weight: 700;
  padding: 0 12px;
}

.payment-concept {
  display: grid;
  grid-template-columns: auto minmax(0, 1fr) auto;
  align-items: center;
  gap: 12px;
  margin: 0;
  padding: 13px;
  border: 1px solid #d8e0e6;
  background: #fffdf6;
  color: #102f56;
}

.payment-concept + .payment-concept {
  border-top: 0;
}

.payment-concept.selected {
  border-color: #0b3e6d;
  background: #e7f0f8;
}

.payment-concept input {
  appearance: auto;
  width: 20px;
  min-width: 20px;
  min-height: 20px;
  margin: 0;
  padding: 0;
  accent-color: #0b3e6d;
}

.payment-concept small {
  display: block;
  margin-top: 3px;
  color: #5b6f82;
  line-height: 1.35;
}

.payment-concept > strong {
  text-align: right;
}

.payment-methods {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 12px;
  max-width: 560px;
  margin-top: 26px;
}

.payment-method {
  display: grid;
  gap: 4px;
  min-height: 82px;
  padding: 14px;
  border: 1px solid #c4dae9;
  border-radius: 4px;
  background: #fffdf6;
  color: #0b3e6d;
  text-align: left;
}

.payment-method span {
  color: #607285;
  font-size: 0.8rem;
}

.payment-method.selected {
  border-color: #0b3e6d;
  background: #0b3e6d;
  color: #fff;
}

.payment-method.selected span {
  color: #e9f2fa;
}

.card-form {
  max-width: 560px;
  margin-top: 30px;
}

.card-details {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 16px;
}

.card-details div {
  display: grid;
  gap: 9px;
}

.card-disclaimer {
  margin: 0;
  color: #607285;
  font-size: 0.78rem;
}

.qr-payment {
  display: flex;
  align-items: center;
  gap: 24px;
  max-width: 560px;
  margin-top: 30px;
  padding: 24px;
  border: 1px solid #d8e0e6;
  background: #fffdf6;
}

.qr-demo {
  display: grid;
  width: 160px;
  min-width: 160px;
  height: 160px;
  place-content: center;
  border: 8px solid #fff;
  outline: 1px solid #0b3e6d;
  background:
    repeating-linear-gradient(90deg, #0b3e6d 0 8px, transparent 8px 16px),
    repeating-linear-gradient(#0b3e6d 0 8px, transparent 8px 16px), #e7f0f8;
  color: #fff;
  text-align: center;
}

.qr-demo span,
.qr-demo strong {
  display: block;
  padding: 2px 6px;
  background: #0b3e6d;
  font-size: 0.68rem;
  letter-spacing: 0.08em;
}

.qr-demo strong {
  color: #df362c;
  font-size: 1rem;
}

.qr-payment h2 {
  margin-bottom: 8px;
}

.qr-payment p {
  margin: 0 0 18px;
  color: #5b6f82;
  line-height: 1.5;
}

.qr-payment button {
  min-height: 44px;
  border: 0;
  border-radius: 4px;
  background: #df362c;
  color: #fff;
  font-weight: 700;
  padding: 0 14px;
}

.qr-payment small {
  display: block;
  margin-top: 12px;
  color: #607285;
  font-size: 0.75rem;
}

.payment-summary {
  align-self: start;
  padding: 26px;
  border: 1px solid #c4dae9;
  background: #e7f0f8;
}

.payment-summary h2 {
  margin: 8px 0 4px;
}

.payment-summary > p:not(.eyebrow) {
  margin: 0;
  color: #5b6f82;
}

.payment-summary > div {
  display: flex;
  justify-content: space-between;
  align-items: end;
  margin: 28px 0 20px;
  padding: 18px 0;
  border-top: 1px solid #b9d3e5;
  border-bottom: 1px solid #b9d3e5;
}

.payment-summary strong {
  color: #0b3e6d;
  font-size: 1.5rem;
}

.payment-summary small {
  display: block;
  color: #5b6f82;
  line-height: 1.45;
}

.payment-success {
  max-width: 560px;
  margin-top: 30px;
  padding: 30px;
  border-left: 5px solid #16704c;
  background: #e9f4ec;
}

.payment-success h2 {
  margin: 8px 0;
}

.payment-success > p:not(.eyebrow) {
  margin: 0 0 22px;
  color: #355d47;
}

.payment-success button {
  min-height: 44px;
  border: 0;
  border-radius: 4px;
  background: #0b3e6d;
  color: #fff;
  font-weight: 700;
  padding: 0 16px;
}

.payment-success .secondary-button {
  margin-left: 8px;
  border: 1px solid #0b3e6d;
  background: transparent;
  color: #0b3e6d;
}

.extract-section {
  margin-top: 34px;
}

.extract-filters {
  display: grid;
  grid-template-columns: 180px 180px minmax(200px, 1fr) auto;
  gap: 12px;
  margin-top: 18px;
}

.extract-filters label {
  display: grid;
  gap: 6px;
  color: #33516e;
  font-size: 0.78rem;
  font-weight: 700;
}

.extract-filters input,
.extract-filters select {
  width: 100%;
  min-height: 42px;
  padding: 0 10px;
  border: 1px solid #c4dae9;
  border-radius: 4px;
  background: #fffdf6;
  color: #102f56;
}

.extract-filters button,
.extract-actions button,
.extract-pagination button {
  min-height: 42px;
  border: 1px solid #0b3e6d;
  border-radius: 4px;
  background: #0b3e6d;
  color: #fff;
  font-weight: 700;
  padding: 0 14px;
}

.extract-filters button {
  align-self: end;
}

.extract-summary {
  display: grid;
  grid-template-columns: repeat(4, minmax(0, 1fr));
  gap: 10px;
  margin-top: 18px;
}

.extract-summary article {
  padding: 14px;
  border: 1px solid #d8e0e6;
  background: #fffdf6;
}

.extract-summary span,
.extract-summary strong {
  display: block;
}

.extract-summary span {
  color: #607285;
  font-size: 0.76rem;
}

.extract-summary strong {
  margin-top: 6px;
  color: #102f56;
  font-size: 1.2rem;
}

.extract-actions,
.extract-actions > div,
.extract-pagination {
  display: flex;
  align-items: center;
  gap: 16px;
}

.extract-actions {
  justify-content: space-between;
  margin: 18px 0;
}

.extract-actions > span,
.extract-pagination span {
  color: #607285;
  font-size: 0.84rem;
}

.extract-actions .share-button,
.extract-pagination button {
  background: transparent;
  color: #0b3e6d;
}

.extract-table-wrap {
  overflow-x: auto;
  border: 1px solid #d8e0e6;
  background: #fffdf6;
}

.extract-table {
  width: 100%;
  min-width: 760px;
  border-collapse: collapse;
  text-align: left;
}

.extract-table th,
.extract-table td {
  padding: 13px 14px;
  border-bottom: 1px solid #dde4e9;
}

.extract-table th {
  color: #58718b;
  font-size: 0.72rem;
  font-weight: 700;
  letter-spacing: 0.06em;
  text-transform: capitalize;
}

.extract-table td {
  color: #102f56;
  font-size: 0.86rem;
}

.extract-table td strong,
.extract-table td small {
  display: block;
}

.extract-table td small {
  margin-top: 3px;
  color: #607285;
}

.extract-status {
  display: inline-block;
  padding: 6px 10px;
  border-radius: 999px;
  font-size: 0.75rem;
  font-weight: 700;
  white-space: nowrap;
}

.extract-status.al_dia {
  background: #e9f4ec;
  color: #16704c;
}

.extract-status.debe_1 {
  background: #e7f0f8;
  color: #33516e;
}

.extract-status.debe_2_mas {
  background: #fae9e7;
  color: #9a3c35;
}

.extract-pagination {
  justify-content: center;
  margin-top: 16px;
}

@media (max-width: 700px) {
  input {
    font-size: 16px;
  }

  .login-page {
    padding: 16px;
  }

  .app-page {
    padding: 12px 16px 36px;
  }

  header {
    min-height: 82px;
  }

  .logo-header {
    width: 130px;
    height: 74px;
  }

  .sign-out,
  .link-button,
  .back-button {
    min-height: 44px;
    padding: 0 4px;
  }

  .welcome {
    margin: 32px 0 24px;
    align-items: start;
    flex-direction: column;
  }

  .summary,
  .content-grid,
  .receipts-layout,
  .profile-grid,
  .payment-layout {
    grid-template-columns: 1fr;
  }

  .prepared-payment,
  .prepared-payment-actions {
    align-items: stretch;
    flex-direction: column;
  }

  .prepared-payment-actions > button {
    width: 100%;
  }

  .receipts-heading {
    align-items: start;
    flex-direction: column;
  }

  .new-extra-charge {
    width: 100%;
  }

  .admin-actions {
    display: grid;
    width: 100%;
    grid-template-columns: 1fr;
  }

  .monthly-fee-button {
    width: 100%;
  }

  .extra-charge-dialog {
    width: 100vw;
    max-width: none;
    min-height: 100dvh;
    margin: 0;
    border-radius: 0;
  }

  .dialog-form {
    display: flex;
    min-height: 100dvh;
    flex-direction: column;
    padding: 28px 20px;
  }

  .dialog-actions {
    display: grid;
    grid-template-columns: 1fr 1fr;
    margin-top: auto;
    padding-top: 24px;
  }

  .summary {
    gap: 12px;
  }

  .summary article,
  .panel {
    min-height: 0;
    padding: 20px;
  }

  .section-heading {
    align-items: center;
  }

  .payment {
    padding: 14px 0;
  }

  .receipt-preview {
    min-height: 0;
    padding: 24px;
    box-shadow: none;
  }

  .receipt-topline,
  .receipt-title {
    align-items: start;
    flex-direction: column;
  }

  .receipt-actions {
    display: grid;
    grid-template-columns: 1fr 1fr;
  }

  .receipt-actions button {
    min-height: 44px;
  }

  .payment-layout {
    gap: 24px;
  }

  .due-date {
    width: 100%;
  }

  .due-dates {
    width: 100%;
  }

  .login-card {
    padding: 28px 20px;
    border-width: 4px;
  }

  .payment-success > div {
    display: grid;
    gap: 8px;
  }

  .payment-success .secondary-button {
    margin-left: 0;
  }

  .extract-filters,
  .extract-summary {
    grid-template-columns: 1fr 1fr;
  }

  .extract-filters button {
    min-height: 44px;
  }
}

@media (max-width: 380px) {
  .card-details {
    grid-template-columns: 1fr;
  }

  .qr-payment {
    align-items: start;
    flex-direction: column;
  }

  .receipt-actions {
    grid-template-columns: 1fr;
  }

  .extract-filters,
  .extract-summary {
    grid-template-columns: 1fr;
  }

  .extract-actions {
    align-items: start;
    flex-direction: column;
  }
}
</style>
