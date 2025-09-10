<template>
  <div class="events-container min-h-screen mx-auto w-full max-w-6xl p-5 text-n-slate-12">
    <!-- Cabeçalho -->
    <div class="events-header flex items-center justify-between gap-4 flex-wrap mb-6">
      <h2 class="events-title text-xl font-medium text-n-accent" style="padding-left: 75px;">Agendamentos</h2>
      <div class="events-actions flex items-center gap-2">
        <Button faded slate @click="loadSchedules" :disabled="loading">
          <template v-if="loading">Carregando...</template>
          <template v-else>Recarregar</template>
        </Button>
        <Button @click="openCreateForm">Novo Agendamento</Button>
      </div>
    </div>

    <!-- Estado vazio -->
    <div
      v-if="!loading && !readyList.length"
      class="empty-state text-center rounded-xl outline outline-1 outline-n-container bg-n-solid-1 p-8"
    >
      <div class="empty-state__icon text-4xl mb-2">⏱️</div>
      <h2 class="empty-state__title text-xl font-semibold mb-1" style="padding-left: 30px;">Sem agendamentos</h2>
      <p class="empty-state__description text-n-slate-10 mb-6">
        Crie seu primeiro agendamento.
      </p>
      <Button lg @click="openCreateForm">Criar</Button>
    </div>

    <!-- Lista -->
    <div v-else-if="!loading" class="events-list w-full" style="padding: 0 75px;">
      <!-- Filtros / Busca -->
      <div class="events-filters flex items-center justify-between gap-4 flex-wrap mb-5">
        <div class="search-box flex-1 max-w-[420px]">
          <input
            v-model="searchQuery"
            type="text"
            placeholder="Buscar por nome, canal ou expressão..."
            class="search-input w-full px-4 py-3 rounded-lg bg-n-background dark:bg-n-solid-1 text-n-slate-12 placeholder:text-n-slate-10 outline outline-1 outline-n-weak focus:outline-n-brand focus:ring-1 focus:ring-n-brand"
          />
        </div>

        <div class="flex items-center gap-3 flex-wrap">
          <!-- filtro canal -->
          <div class="segmented inline-flex items-center gap-1 rounded-full bg-n-solid-1 outline outline-1 outline-n-container p-1">
            <button
              v-for="opt in channelOptions"
              :key="opt.value"
              class="segmented__btn px-3 py-1.5 text-xs rounded-full"
              :class="channelFilter === opt.value ? 'bg-n-solid-2 text-n-slate-12' : 'text-n-slate-11 hover:bg-n-solid-2'"
              @click="setChannel(opt.value)"
            >{{ opt.label }}</button>
          </div>

          <!-- filtro status -->
          <div class="segmented inline-flex items-center gap-1 rounded-full bg-n-solid-1 outline outline-1 outline-n-container p-1">
            <button
              v-for="opt in statusOptions"
              :key="opt.value"
              class="segmented__btn px-3 py-1.5 text-xs rounded-full"
              :class="statusFilter === opt.value ? 'bg-n-solid-2 text-n-slate-12' : 'text-n-slate-11 hover:bg-n-solid-2'"
              @click="setStatus(opt.value)"
            >{{ opt.label }}</button>
          </div>

          <!-- legenda -->
          <div class="legend flex items-center gap-4 text-xs text-n-slate-10">
            <span class="legend-item inline-flex items-center gap-2">
              <span class="status-dot w-2 h-2 rounded-full bg-n-teal-9"></span>
              Ativo
            </span>
            <span class="legend-item inline-flex items-center gap-2">
              <span class="status-dot w-2 h-2 rounded-full bg-n-slate-9"></span>
              Inativo
            </span>
          </div>
        </div>
      </div>

      <!-- Grid -->
      <div class="events-grid grid grid-cols-1 md:grid-cols-2 xl:grid-cols-3 gap-5 mb-8">
        <div
          v-for="it in pagedList"
          :key="it.eventId"
          class="event-card rounded-xl outline outline-1 outline-n-container bg-n-solid-1 shadow transition hover:shadow-lg"
          :class="{ 'opacity-75': !it.enabled }"
        >
          <!-- Header do card -->
          <div class="event-card__header flex justify-between items-start p-5 border-b border-n-weak bg-n-solid-2">
            <div class="event-card__title-section min-w-0">
              <h3 class="event-card__title text-base font-semibold mb-2 truncate" :title="it.eventId">{{ it.eventId }}</h3>
              <div class="flex gap-2 items-center">
                <span
                  class="event-card__status text-[11px] font-semibold px-2 py-0.5 rounded-full"
                  :class="it.enabled ? 'bg-n-teal-1 text-n-teal-11' : 'bg-n-slate-3 text-n-slate-10'"
                >
                  {{ it.enabled ? 'Habilitado' : 'Desabilitado' }}
                </span>
                <span class="pill text-[11px] px-2 py-0.5 rounded-full outline outline-1 outline-n-weak" :title="it.channel">
                  {{ it.channel === 'email' ? 'Email' : 'WhatsApp' }}
                </span>
              </div>
            </div>

            <div class="event-card__actions flex items-center gap-2">
              <Button
                sm ghost slate
                @click="toggleEnabled(it)"
                :disabled="togglingId === it.eventId"
                :title="it.enabled ? 'Desabilitar' : 'Habilitar'"
              >
                <template v-if="it.enabled">
                  <svg width="20" height="20" viewBox="0 0 24 24" fill="none">
                    <path d="M8 5V19M16 5V19" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/>
                  </svg>
                </template>
                <template v-else>
                  <svg width="20" height="20" viewBox="0 0 24 24" fill="none">
                    <path d="M16.6582 9.28638C18.098 10.1862 18.8178 10.6361 19.0647 11.2122C19.2803 11.7152 19.2803 12.2847 19.0647 12.7878C18.8178 13.3638 18.098 13.8137 16.6582 14.7136L9.896 18.94C8.29805 19.9387 7.49907 20.4381 6.83973 20.385C6.26501 20.3388 5.73818 20.0469 5.3944 19.584C5 19.053 5 18.1108 5 16.2264V7.77357C5 5.88919 5 4.94701 5.3944 4.41598C5.73818 3.9531 6.26501 3.66111 6.83973 3.6149C7.49907 3.5619 8.29805 4.06126 9.896 5.05998L16.6582 9.28638Z" stroke="currentColor" stroke-width="2" stroke-linejoin="round"/>
                  </svg>
                </template>
              </Button>

              <Button sm ghost slate title="Editar" @click="editSchedule(it)">
                <svg width="20" height="20" viewBox="0 0 24 24" fill="none">
                  <path d="M21.2799 6.40005L11.7399 15.94C10.7899 16.89 7.96987 17.33 7.33987 16.7C6.70987 16.07 7.13987 13.25 8.08987 12.3L17.6399 2.75002C17.8754 2.49308 18.1605 2.28654 18.4781 2.14284C18.7956 1.99914 19.139 1.92124 19.4875 1.9139C19.8359 1.90657 20.1823 1.96991 20.5056 2.10012C20.8289 2.23033 21.1225 2.42473 21.3686 2.67153C21.6147 2.91833 21.8083 3.21243 21.9376 3.53609C22.0669 3.85976 22.1294 4.20626 22.1211 4.55471C22.1128 4.90316 22.0339 5.24635 21.8894 5.5635C21.7448 5.88065 21.5375 6.16524 21.2799 6.40005V6.40005Z" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/>
                  <path d="M11 4H6C4.93913 4 3.92178 4.42142 3.17163 5.17157C2.42149 5.92172 2 6.93913 2 8V18C2 19.0609 2.42149 20.0783 3.17163 20.8284C3.92178 21.5786 4.93913 22 6 22H17C19.21 22 20 20.2 20 18V13" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/>
                </svg>
              </Button>

              <Button sm ghost ruby title="Excluir" @click="confirmDelete(it)" :disabled="deletingId === it.eventId">
                <svg width="20" height="20" viewBox="0 0 24 24" fill="none">
                  <path d="M3 3L21 21M18 6L17.6 12M17.2498 17.2527L17.1991 18.0129C17.129 19.065 17.0939 19.5911 16.8667 19.99C16.6666 20.3412 16.3648 20.6235 16.0011 20.7998C15.588 21 15.0607 21 14.0062 21H9.99377C8.93927 21 8.41202 21 7.99889 20.7998C7.63517 20.6235 7.33339 20.3412 7.13332 19.99C6.90607 19.5911 6.871 19.065 6.80086 18.0129L6 6H4M16 6L15.4559 4.36754C15.1837 3.55086 14.4194 3 13.5585 3H10.4416C9.94243 3 9.47576 3.18519 9.11865 3.5M11.6133 6H20M14 14V17M10 10V17" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/>
                </svg>
              </Button>
            </div>
          </div>

          <!-- Conteúdo -->
          <div class="event-card__content p-5">
            <div class="event-info mb-4">
              <div class="event-info__item flex items-baseline mb-2">
                <label class="event-info__label min-w-[120px] text-sm font-medium text-n-slate-10">Modelo:</label>
                <span class="event-info__value text-sm">{{ it.runAt ? 'Pontual (at)' : 'Recorrente (cron)' }}</span>
              </div>

              <div class="event-info__item flex items-baseline mb-2" v-if="!it.runAt">
                <label class="event-info__label min-w-[120px] text-sm font-medium text-n-slate-10">Quando:</label>
                <span class="event-info__value text-sm">{{ humanRecurring(it) }}</span>
              </div>
              <div class="event-info__item flex items-baseline mb-2" v-else>
                <label class="event-info__label min-w-[120px] text-sm font-medium text-n-slate-10">Executar em:</label>
                <span class="event-info__value text-sm">{{ formatDateTime(it.runAt) }}</span>
              </div>

              <div class="event-info__item flex items-baseline mb-2">
                <label class="event-info__label min-w-[120px] text-sm font-medium text-n-slate-10">Expressão:</label>
                <span class="event-info__value text-sm font-mono">{{ it.scheduleExpression || '—' }}</span>
              </div>

              <div class="event-info__item flex items-baseline mb-2">
                <label class="event-info__label min-w-[120px] text-sm font-medium text-n-slate-10">Vigência:</label>
                <span class="event-info__value text-sm">
                  <template v-if="it.startAt || it.endAt">
                    {{ it.startAt ? formatDateTime(it.startAt) : '—' }} — {{ it.endAt ? formatDateTime(it.endAt) : '—' }}
                  </template>
                  <template v-else>—</template>
                </span>
              </div>

              <div class="event-info__item flex items-baseline">
                <label class="event-info__label min-w-[120px] text-sm font-medium text-n-slate-10">Destinatários:</label>
                <span class="event-info__value text-sm">{{ it.recipients.length }}</span>
              </div>
            </div>

            <Button
              v-if="it.recipients.length"
              ghost
              slate
              class="w-full justify-start"
              @click="toggleRecipients(it.eventId)"
            >
              <span class="mr-2">{{ expanded.includes(it.eventId) ? '▼' : '▶' }}</span>
              {{ expanded.includes(it.eventId) ? 'Ocultar' : 'Ver' }} destinatários
            </Button>

            <div v-if="expanded.includes(it.eventId)" class="clients-list mt-4 border-t border-n-weak pt-4">
              <div class="clients-grid flex flex-col gap-2">
                <div
                  v-for="(r, idx) in it.recipients"
                  :key="idx"
                  class="client-card flex items-center justify-between p-3 rounded-lg outline outline-1 outline-n-container bg-n-solid-1"
                >
                  <div class="client-info">
                    <div class="client-name text-sm font-medium">{{ r.name || '(sem nome)' }}</div>
                    <div class="client-phone text-xs text-n-slate-10">
                      <template v-if="it.channel === 'email'">{{ r.email }}</template>
                      <template v-else>{{ r.phone }}</template>
                    </div>
                  </div>
                  <div class="client-actions">
                    <Button xs ghost slate @click="previewMessage(it, r)">Prévia</Button>
                  </div>
                </div>
              </div>
            </div>
          </div> <!-- /content -->
        </div>
      </div>

      <!-- Paginação -->
      <div v-if="totalPages > 1" class="pagination flex justify-center items-center gap-4 mt-8">
        <Button sm ghost slate @click="prevPage" :disabled="currentPage <= 1">← Anterior</Button>
        <span class="pagination-info text-sm text-n-slate-10">Página {{ currentPage }} de {{ totalPages }}</span>
        <Button sm ghost slate @click="nextPage" :disabled="currentPage >= totalPages">Próxima →</Button>
      </div>
    </div>

    <!-- Loading -->
    <div v-if="loading" class="loading-state text-center py-16 text-n-slate-10">
      <div class="w-8 h-8 border-2 border-n-weak border-t-n-brand rounded-full animate-spin mx-auto mb-4"></div>
      <p>Carregando agendamentos...</p>
    </div>

    <!-- Modal principal (criar/editar) -->
    <div v-if="showForm" class="modal-overlay fixed inset-0 bg-black/70 grid place-items-center z-[1000] p-5" @click.self="closeForm">
      <div class="modal-content bg-n-solid-1 rounded-xl outline outline-1 outline-n-container shadow-2xl w-full max-w-4xl max-h-[90vh] overflow-hidden">
        <div class="modal-header flex justify-between items-center p-5 border-b border-n-weak bg-n-solid-2">
          <h2 class="modal-title text-lg font-semibold">{{ selected ? 'Editar Agendamento' : 'Novo Agendamento' }}</h2>
          <Button sm ghost slate class="modal-close" @click="closeForm">✕</Button>
        </div>
        <div class="modal-body max-h-[calc(90vh-80px)] overflow-y-auto p-4">
          <EventForm :value="selected" @saved="handleSaved" />
        </div>
      </div>
    </div>

    <!-- Modal de prévia -->
    <div v-if="preview.open" class="modal-overlay fixed inset-0 bg-black/70 grid place-items-center z-[1100] p-5" @click.self="closePreview">
      <div class="modal-content bg-n-solid-1 rounded-xl outline outline-1 outline-n-container shadow-2xl w-full max-w-2xl max-h-[80vh] overflow-hidden">
        <div class="modal-header flex justify-between items-center p-4 border-b border-n-weak bg-n-solid-2">
          <h2 class="modal-title text-base font-semibold">Prévia</h2>
          <Button sm ghost slate class="modal-close" @click="closePreview">✕</Button>
        </div>
        <div class="modal-body p-4">
          <pre class="preview-box whitespace-pre-wrap text-sm text-n-slate-12 bg-n-solid-1 outline outline-1 outline-n-container rounded-lg p-3">{{ preview.text }}</pre>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import axios from 'axios'
import EventForm from './EventForm.vue'
import Button from 'dashboard/components-next/button/Button.vue'

const API_BASE = 'https://f4wzfjousg.execute-api.us-east-1.amazonaws.com/schedules'
const tz = 'America/Sao_Paulo'
const DOW_LABEL = { SUN: 'Dom', MON: 'Seg', TUE: 'Ter', WED: 'Qua', THU: 'Qui', FRI: 'Sex', SAT: 'Sáb' }

function fmtDateTime(d) {
  if (!d) return ''
  try {
    const asISO = typeof d === 'string' && d.includes(' ') ? d.replace(' ', 'T') : d
    const date = new Date(asISO)
    if (isNaN(date.getTime())) return d
    return date.toLocaleString('pt-BR', {
      timeZone: tz,
      year: 'numeric', month: '2-digit', day: '2-digit',
      hour: '2-digit', minute: '2-digit'
    })
  } catch { return d }
}

export default {
  name: 'SchedulesIndex',
  components: { EventForm, Button },
  data() {
    return {
      api: axios.create({ baseURL: API_BASE }),
      loading: false,
      showForm: false,
      schedules: [],
      selected: null,
      togglingId: null,
      deletingId: null,

      // filtros / busca / paginação
      searchQuery: '',
      channelFilter: 'all', // all | whatsapp | email
      statusFilter: 'all',  // all | enabled | disabled
      channelOptions: [
        { value: 'all', label: 'Todos' },
        { value: 'whatsapp', label: 'WhatsApp' },
        { value: 'email', label: 'Email' }
      ],
      statusOptions: [
        { value: 'all', label: 'Todos' },
        { value: 'enabled', label: 'Ativos' },
        { value: 'disabled', label: 'Inativos' }
      ],
      expanded: [],
      currentPage: 1,
      pageSize: 6,

      // prévia
      preview: { open: false, text: '' },
    }
  },
  computed: {
    readyList() {
      let list = this.schedules.slice()

      // filtro por canal
      if (this.channelFilter !== 'all') {
        list = list.filter(it => (it.channel || '').toLowerCase() === this.channelFilter)
      }
      // filtro por status
      if (this.statusFilter !== 'all') {
        const want = this.statusFilter === 'enabled'
        list = list.filter(it => !!it.enabled === want)
      }
      // busca textual
      const q = this.searchQuery.trim().toLowerCase()
      if (q) {
        list = list.filter(it =>
          (it.eventId || '').toLowerCase().includes(q) ||
          (it.channel || '').toLowerCase().includes(q) ||
          (it.scheduleExpression || '').toLowerCase().includes(q)
        )
      }
      return list
    },
    totalPages() {
      return Math.max(1, Math.ceil(this.readyList.length / this.pageSize))
    },
    pagedList() {
      const start = (this.currentPage - 1) * this.pageSize
      return this.readyList.slice(start, start + this.pageSize)
    }
  },
  watch: {
    searchQuery() { this.currentPage = 1 },
    channelFilter() { this.currentPage = 1 },
    statusFilter() { this.currentPage = 1 },
    readyList(newList) {
      const max = Math.max(1, Math.ceil(newList.length / this.pageSize))
      if (this.currentPage > max) this.currentPage = max
    }
  },
  mounted() {
    this.loadSchedules()
    window.addEventListener('keydown', this.onKeydown)
  },
  beforeUnmount() {
    window.removeEventListener('keydown', this.onKeydown)
  },
  methods: {
    onKeydown(e) {
      if (e.key === 'Escape') {
        if (this.showForm) this.closeForm()
        if (this.preview.open) this.closePreview()
      }
    },
    setChannel(v) { this.channelFilter = v },
    setStatus(v) { this.statusFilter = v },

    openCreateForm() { this.selected = null; this.showForm = true },
    closeForm() { this.showForm = false; this.selected = null },

    prevPage() { if (this.currentPage > 1) this.currentPage-- },
    nextPage() { if (this.currentPage < this.totalPages) this.currentPage++ },

    formatDateTime: fmtDateTime,
    humanRecurring(it) {
      const days = (it.daysOfWeek || []).map(d => DOW_LABEL[d] || d).join(', ')
      const time = it.time || '—'
      const tzs = it.timezone || 'America/Sao_Paulo'
      return days ? `${days} às ${time} (${tzs})` : '—'
    },

    toggleRecipients(id) {
      const i = this.expanded.indexOf(id)
      if (i > -1) this.expanded.splice(i, 1)
      else this.expanded.push(id)
    },

    dayKeyFor(timezone) {
      try {
        const now = new Date()
        const fmt = new Intl.DateTimeFormat('en-US', { timeZone: timezone || 'UTC', weekday: 'short' })
        return fmt.format(now).toUpperCase().slice(0,3) // MON/TUE/WED...
      } catch { return 'MON' }
    },

    renderTemplate(tpl, vars) {
      if (!tpl) return ''
      const v = { ...(vars || {}) }
      if (v.name && !v.nome) v.nome = v.name
      if (v.nome && !v.name) v.name = v.nome
      return tpl.replace(/\{\{\s*([a-zA-Z0-9_\-\.]+)\s*\}\}/g, (_, k) => (v[k] ?? ''))
    },

    previewMessage(it, r) {
      const chan = it.channel
      let preview = ''
      const varsMap = { name: r.name || '', email: r.email || '', phone: r.phone || '', ...(r.vars || {}) }
      if (chan === 'email') {
        const subj = this.renderTemplate(it.payload?.subject || 'Mensagem', varsMap)
        const text = this.renderTemplate(it.payload?.text || it.payload?.html || '', varsMap)
        preview = `Assunto: ${subj}\n\n${text}`
      } else {
        const k = this.dayKeyFor(it.timezone)
        const byDay = it.payload?.messagesByDay || {}
        const base = it.payload?.message || ''
        const chosen = byDay[k] || base
        preview = this.renderTemplate(chosen || '', varsMap)
      }
      this.preview = { open: true, text: preview || '(sem conteúdo)' }
    },

    closePreview() { this.preview = { open: false, text: '' } },

    async loadSchedules() {
      this.loading = true
      try {
        const { data } = await this.api.get('') // GET /schedules
        const items = Array.isArray(data?.items)
          ? data.items
          : Array.isArray(data) ? data
          : []
        this.schedules = items.map(row => ({
          eventId: row.Name,
          channel: row.Channel,
          recipients: row.Recipients || [],
          payload: row.Payload || {},
          enabled: !!row.Enabled,
          scheduleExpression: row.ScheduleExpression || '',
          timezone: row.Timezone || null,
          daysOfWeek: row.DaysOfWeek || [],
          time: row.Time || null,
          runAt: row.RunAt || null,
          startAt: row.StartAt || null,
          endAt: row.EndAt || null,
          agent: row.Agent || null,
        }))
      } catch (e) {
        console.error('Falha ao carregar:', e)
        this.schedules = []
      } finally {
        this.loading = false
      }
    },

    editSchedule(it) {
      this.selected = {
        Name: it.eventId,
        Channel: it.channel,
        Recipients: JSON.parse(JSON.stringify(it.recipients || [])),
        Payload: JSON.parse(JSON.stringify(it.payload || {})),
        Enabled: it.enabled,
        ScheduleExpression: it.scheduleExpression,
        Timezone: it.timezone,
        DaysOfWeek: it.daysOfWeek,
        Time: it.time,
        RunAt: it.runAt,
        StartAt: it.startAt,
        EndAt: it.endAt,
        Agent: it.agent || null,
      }
      this.showForm = true
    },

    async toggleEnabled(it) {
      if (!it?.eventId) return
      this.togglingId = it.eventId
      try {
        await this.api.patch(`/${encodeURIComponent(it.eventId)}`, { enabled: !it.enabled })
        it.enabled = !it.enabled
      } catch (e) {
        console.error('Erro ao alternar enabled:', e)
        alert('Não foi possível atualizar o status.')
      } finally {
        this.togglingId = null
      }
    },

    async confirmDelete(it) {
      if (!it?.eventId || this.deletingId) return
      const ok = confirm(`Excluir o agendamento "${it.eventId}"?`)
      if (!ok) return
      this.deletingId = it.eventId
      try {
        await this.api.delete(`/${encodeURIComponent(it.eventId)}`)
        this.schedules = this.schedules.filter(x => x.eventId !== it.eventId)
      } catch (e) {
        console.error('Erro ao excluir:', e)
        alert('Falha ao excluir.')
      } finally {
        this.deletingId = null
      }
    },

    handleSaved() {
      this.showForm = false
      this.selected = null
      this.loadSchedules()
    }
  }
}
</script>

<style scoped>
</style>
