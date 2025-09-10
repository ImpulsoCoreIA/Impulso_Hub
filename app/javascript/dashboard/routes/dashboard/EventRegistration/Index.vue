<template>
  <div class="events-container">
    <!-- Cabeçalho -->
    <div class="events-header">
      <div class="events-title-wrap">
        <h1 class="events-title">Agendamentos</h1>
        <div class="stats">
          <span class="stat-pill">Total: {{ schedules.length }}</span>
          <span class="stat-pill stat-pill--ok">Ativos: {{ activeCount }}</span>
          <span class="stat-pill stat-pill--muted">Inativos: {{ inactiveCount }}</span>
        </div>
      </div>
      <div class="events-actions">
        <button class="btn btn--secondary" @click="loadSchedules" :disabled="loading">
          <span v-if="loading">Carregando...</span>
          <span v-else>Recarregar</span>
        </button>
        <button class="btn btn--primary" @click="openCreateForm">Novo Agendamento</button>
      </div>
    </div>

    <!-- Toolbar fixa: busca + filtros + ordenação -->
    <div class="toolbar">
      <div class="search-box">
        <svg class="search-icon" viewBox="0 0 24 24" aria-hidden="true">
          <path d="M21 21l-4.3-4.3M10.5 18a7.5 7.5 0 1 1 0-15 7.5 7.5 0 0 1 0 15Z" stroke="currentColor" stroke-width="2" fill="none" stroke-linecap="round"/>
        </svg>
        <input
          v-model="searchQuery"
          type="text"
          placeholder="Buscar por nome, canal ou expressão..."
          class="search-input"
        />
      </div>

      <div class="segmented">
        <div class="segmented__group" role="tablist" aria-label="Filtro de canal">
          <button
            v-for="opt in channelOptions"
            :key="opt.value"
            :class="['segmented__btn', { 'is-active': channelFilter === opt.value }]"
            role="tab"
            @click="channelFilter = opt.value"
          >{{ opt.label }}</button>
        </div>

        <div class="segmented__group" role="tablist" aria-label="Filtro de status">
          <button
            v-for="opt in statusOptions"
            :key="opt.value"
            :class="['segmented__btn', { 'is-active': statusFilter === opt.value }]"
            role="tab"
            @click="statusFilter = opt.value"
          >{{ opt.label }}</button>
        </div>

        <div class="sorter">
          <label class="sorter__label">Ordenar</label>
          <select v-model="sortKey" class="sorter__select">
            <option value="name">Nome</option>
            <option value="channel">Canal</option>
            <option value="when">Quando</option>
          </select>
          <button class="btn btn--ghost btn--small" @click="toggleSortDir" :title="sortDir === 'asc' ? 'Ascendente' : 'Descendente'">
            <span v-if="sortDir === 'asc'">▲</span>
            <span v-else>▼</span>
          </button>
        </div>
      </div>
    </div>

    <!-- Estado vazio -->
    <div v-if="!loading && !readyList.length" class="empty-state">
      <div class="empty-state__icon">⏱️</div>
      <h2 class="empty-state__title">Nada por aqui…</h2>
      <p class="empty-state__description">
        Crie seu primeiro agendamento ou ajuste os filtros/busca.
      </p>
      <div class="empty-actions">
        <button class="btn btn--primary btn--large" @click="openCreateForm">Criar agendamento</button>
        <button class="btn btn--secondary btn--large" @click="clearFilters">Limpar filtros</button>
      </div>
    </div>

    <!-- Lista -->
    <div v-else-if="!loading" class="events-list">
      <div class="events-grid">
        <div
          v-for="it in pagedList"
          :key="it.eventId"
          class="event-card"
          :class="{ 'event-card--inactive': !it.enabled }"
        >
          <!-- Header do card -->
          <div class="event-card__header">
            <div class="event-card__title-section">
              <h3 class="event-card__title" :title="it.eventId">{{ it.eventId }}</h3>
              <div class="event-card__meta">
                <span
                  class="event-card__status"
                  :class="it.enabled ? 'event-card__status--active' : 'event-card__status--inactive'"
                >
                  {{ it.enabled ? 'Habilitado' : 'Desabilitado' }}
                </span>
                <span class="pill" :title="it.channel">
                  {{ it.channel === 'email' ? 'Email' : 'WhatsApp' }}
                </span>
              </div>
            </div>

            <div class="event-card__actions">
              <button
                class="btn btn--small btn--ghost"
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
              </button>

              <button class="btn btn--small btn--ghost" @click="editSchedule(it)" title="Editar">
                <svg width="20" height="20" viewBox="0 0 24 24" fill="none">
                  <path d="M21.28 6.4L11.74 15.94c-.95.95-3.77 1.39-4.4.76-.63-.63-.2-3.45.75-4.39l9.55-9.55a3.2 3.2 0 0 1 4.53 4.53Z" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/>
                  <path d="M11 4H6a4 4 0 0 0-4 4v10a4 4 0 0 0 4 4h11c2.21 0 3-1.8 3-4v-5" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/>
                </svg>
              </button>

              <button class="btn btn--small btn--ghost btn--danger" @click="confirmDelete(it)" title="Excluir">
                <svg width="20" height="20" viewBox="0 0 24 24" fill="none">
                  <path d="M3 3L21 21M18 6L17.6 12M17.25 17.25l-.05.76c-.07 1.05-.1 1.58-.33 1.98-.2.35-.5.63-.86.81-.42.2-.95.2-2 .2H10c-1.06 0-1.59 0-2-.2a1.98 1.98 0 0 1-.86-.81c-.23-.4-.26-.93-.32-1.98L6 6H4M16 6l-.54-1.63A2 2 0 0 0 13.56 3h-3.12M11.61 6H20M14 14v3M10 10v7" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/>
                </svg>
              </button>
            </div>
          </div>

          <!-- Conteúdo -->
          <div class="event-card__content">
            <div class="event-info">
              <div class="event-info__item">
                <label class="event-info__label">Modelo:</label>
                <span class="event-info__value">{{ it.runAt ? 'Pontual (at)' : 'Recorrente (cron)' }}</span>
              </div>

              <div class="event-info__item" v-if="!it.runAt">
                <label class="event-info__label">Quando:</label>
                <span class="event-info__value">{{ humanRecurring(it) }}</span>
              </div>
              <div class="event-info__item" v-else>
                <label class="event-info__label">Executar em:</label>
                <span class="event-info__value">{{ formatDateTime(it.runAt) }}</span>
              </div>

              <div class="event-info__item">
                <label class="event-info__label">Expressão:</label>
                <span class="event-info__value font-mono cut">{{ it.scheduleExpression || '—' }}</span>
              </div>

              <div class="event-info__item">
                <label class="event-info__label">Vigência:</label>
                <span class="event-info__value">
                  <template v-if="it.startAt || it.endAt">
                    {{ it.startAt ? formatDateTime(it.startAt) : '—' }} — {{ it.endAt ? formatDateTime(it.endAt) : '—' }}
                  </template>
                  <template v-else>—</template>
                </span>
              </div>

              <div class="event-info__item" v-if="it.channel==='whatsapp'">
                <label class="event-info__label">Agente:</label>
                <span class="event-info__value cut">{{ it.agent || '—' }}</span>
              </div>

              <div class="event-info__item">
                <label class="event-info__label">Destinatários:</label>
                <span class="event-info__value">{{ it.recipients.length }}</span>
              </div>
            </div>

            <button
              class="clients-toggle"
              @click="toggleRecipients(it.eventId)"
              v-if="it.recipients.length"
            >
              <span class="chev" :class="{ 'chev--open': expanded.includes(it.eventId) }">▸</span>
              {{ expanded.includes(it.eventId) ? 'Ocultar' : 'Ver' }} destinatários
            </button>

            <div v-if="expanded.includes(it.eventId)" class="clients-list">
              <div class="clients-grid">
                <div v-for="(r, idx) in it.recipients" :key="idx" class="client-card">
                  <div class="client-info">
                    <div class="client-name">{{ r.name || '(sem nome)' }}</div>
                    <div class="client-phone">
                      <template v-if="it.channel === 'email'">{{ r.email }}</template>
                      <template v-else>{{ r.phone }}</template>
                    </div>
                  </div>
                  <div class="client-actions">
                    <button class="btn btn--tiny btn--ghost" @click="previewMessage(it, r)">Prévia</button>
                  </div>
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>

      <!-- Paginação -->
      <div class="pagination" v-if="totalPages > 1 || pageSizeOptions.length">
        <div class="pagination__left">
          <label class="pagination__label">Itens por página</label>
          <select v-model.number="pageSize" class="pagination__select">
            <option v-for="n in pageSizeOptions" :key="n" :value="n">{{ n }}</option>
          </select>
        </div>

        <div class="pagination__center">
          <button class="btn btn--small btn--ghost" @click="goFirst" :disabled="currentPage <= 1">«</button>
          <button class="btn btn--small btn--ghost" @click="currentPage--" :disabled="currentPage <= 1">←</button>
          <span class="pagination-info">Página {{ currentPage }} de {{ totalPages }}</span>
          <button class="btn btn--small btn--ghost" @click="currentPage++" :disabled="currentPage >= totalPages">→</button>
          <button class="btn btn--small btn--ghost" @click="goLast" :disabled="currentPage >= totalPages">»</button>
        </div>

        <div class="pagination__right">
          <span class="pagination-info">{{ readyList.length }} resultado(s)</span>
        </div>
      </div>
    </div>

    <!-- Loading -->
    <div v-if="loading" class="loading-state">
      <div class="loading-spinner"></div>
      <p>Carregando agendamentos...</p>
    </div>

    <!-- Modal principal (criar/editar) -->
    <div v-if="showForm" class="modal-overlay" @click.self="closeForm">
      <div class="modal-content">
        <div class="modal-header">
          <h2 class="modal-title">{{ selected ? 'Editar Agendamento' : 'Novo Agendamento' }}</h2>
          <button class="btn btn--ghost btn--small modal-close" @click="closeForm">✕</button>
        </div>
        <div class="modal-body">
          <EventForm :value="selected" @saved="handleSaved" />
        </div>
      </div>
    </div>

    <!-- Modal de prévia -->
    <div v-if="preview.open" class="modal-overlay" @click.self="closePreview">
      <div class="modal-content modal-content--sm">
        <div class="modal-header">
          <h2 class="modal-title">Prévia</h2>
          <button class="btn btn--ghost btn--small modal-close" @click="closePreview">✕</button>
        </div>
        <div class="modal-body">
          <pre class="preview-box">{{ preview.text }}</pre>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import axios from 'axios'
import EventForm from './EventForm.vue'

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
  components: { EventForm },
  data() {
    return {
      api: axios.create({ baseURL: API_BASE }),
      loading: false,
      showForm: false,
      schedules: [],
      selected: null,
      togglingId: null,
      deletingId: null,
      searchQuery: '',
      channelFilter: 'all',
      statusFilter: 'all',
      sortKey: 'name',
      sortDir: 'asc',
      expanded: [],
      currentPage: 1,
      pageSize: 6,
      pageSizeOptions: [6, 12, 24, 48],
      preview: { open: false, text: '' },
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
    }
  },
  computed: {
    activeCount() { return this.schedules.filter(s => s.enabled).length },
    inactiveCount() { return this.schedules.filter(s => !s.enabled).length },

    readyList() {
      // base
      let list = this.schedules.slice()

      // filtros
      if (this.channelFilter !== 'all') {
        list = list.filter(it => (it.channel || '').toLowerCase() === this.channelFilter)
      }
      if (this.statusFilter !== 'all') {
        const want = this.statusFilter === 'enabled'
        list = list.filter(it => !!it.enabled === want)
      }

      // busca
      const q = this.searchQuery.trim().toLowerCase()
      if (q) {
        list = list.filter(it =>
          (it.eventId || '').toLowerCase().includes(q) ||
          (it.channel || '').toLowerCase().includes(q) ||
          (it.scheduleExpression || '').toLowerCase().includes(q)
        )
      }

      // ordenação
      const dir = this.sortDir === 'asc' ? 1 : -1
      list.sort((a, b) => {
        if (this.sortKey === 'channel') {
          return (a.channel || '').localeCompare(b.channel || '') * dir
        }
        if (this.sortKey === 'when') {
          // tenta ordenar por runAt; se recorrente, usa time (HH:mm)
          const av = a.runAt ? a.runAt : (a.time || '')
          const bv = b.runAt ? b.runAt : (b.time || '')
          return String(av).localeCompare(String(bv)) * dir
        }
        // default: nome
        return (a.eventId || '').localeCompare(b.eventId || '') * dir
      })

      return list
    },

    totalPages() { return Math.max(1, Math.ceil(this.readyList.length / this.pageSize)) },

    pagedList() {
      const start = (this.currentPage - 1) * this.pageSize
      return this.readyList.slice(start, start + this.pageSize)
    }
  },
  mounted() {
    this.loadSchedules()
    window.addEventListener('keydown', this.onKeydown)
  },
  beforeUnmount() { window.removeEventListener('keydown', this.onKeydown) },
  watch: {
    searchQuery() { this.currentPage = 1 },
    channelFilter() { this.currentPage = 1 },
    statusFilter() { this.currentPage = 1 },
    pageSize() { this.currentPage = 1 },
    readyList(newList) {
      // se a página atual ficou além do fim, volta
      const maxPage = Math.max(1, Math.ceil(newList.length / this.pageSize))
      if (this.currentPage > maxPage) this.currentPage = maxPage
    }
  },
  methods: {
    onKeydown(e) { if (e.key === 'Escape') { if (this.showForm) this.closeForm(); if (this.preview.open) this.closePreview() } },
    openCreateForm() { this.selected = null; this.showForm = true },
    closeForm() { this.showForm = false; this.selected = null },
    clearFilters() { this.searchQuery = ''; this.channelFilter = 'all'; this.statusFilter = 'all' },
    toggleSortDir() { this.sortDir = this.sortDir === 'asc' ? 'desc' : 'asc' },
    goFirst() { this.currentPage = 1 },
    goLast() { this.currentPage = this.totalPages },
    formatDateTime: fmtDateTime,
    humanRecurring(it) {
      const DOW_LABEL = { SUN: 'Dom', MON: 'Seg', TUE: 'Ter', WED: 'Qua', THU: 'Qui', FRI: 'Sex', SAT: 'Sáb' }
      const days = (it.daysOfWeek || []).map(d => DOW_LABEL[d] || d).join(', ')
      const time = it.time || '—'
      const tzs = it.timezone || 'America/Sao_Paulo'
      return days ? `${days} às ${time} (${tzs})` : '—'
    },
    toggleRecipients(id) {
      const i = this.expanded.indexOf(id)
      if (i > -1) this.expanded.splice(i, 1); else this.expanded.push(id)
    },
    dayKeyFor(timezone) {
      try {
        const now = new Date()
        const fmt = new Intl.DateTimeFormat('en-US', { timeZone: timezone || 'UTC', weekday: 'short' })
        const k = fmt.format(now).toUpperCase().slice(0,3)
        return k
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
        const msg = this.renderTemplate(chosen || '', varsMap)
        preview = msg
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
      } finally { this.loading = false }
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
      } finally { this.togglingId = null }
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
      } finally { this.deletingId = null }
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
/* ----- Base ----- */
.events-container{
  /* offset do header fixo do seu app (ajuste se precisar: 56/64/72px) */
  --app-header-offset: 64px;

  min-height: 100vh;
  width: 100%;
  max-width: none;        /* evita ser comprimido pelo pai flex */
  margin: 0;              /* neutraliza o "empurrão" do auto dentro de flex */
  padding: 20px clamp(12px, 2vw, 24px);

  background: #1a1a1a;
  color: #e5e5e5;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
}
.font-mono{ font-family: ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, "Liberation Mono", "Courier New", monospace; }

/* ----- Header ----- */
.events-header{
  display:flex; justify-content:space-between; align-items:center;
  margin-bottom:10px; flex-wrap:wrap; gap:16px;
}
.events-title-wrap{ display:flex; align-items:baseline; gap:12px; flex-wrap:wrap; }
.events-title{ font-size:26px; font-weight:800; margin:0; color:#fff; }
.stats{ display:flex; gap:8px; flex-wrap:wrap; }
.stat-pill{ padding:4px 10px; border-radius:999px; border:1px solid #333; background:#242424; font-size:12px; color:#d6d6d6; }
.stat-pill--ok{ border-color:#0f5132; background:#0b2e22; color:#10b981; }
.stat-pill--muted{ color:#9ca3af; }

/* ----- Actions ----- */
.events-actions{ display:flex; gap:12px; }

/* ----- Toolbar fixa ----- */
.toolbar{
  position: sticky;
  top: var(--app-header-offset);
  z-index: 10;

  display:grid; grid-template-columns: 1.2fr auto auto; gap:12px;
  align-items:center; padding:12px 0 14px; margin-bottom:12px;
  background: linear-gradient(180deg, rgba(26,26,26,.95), rgba(26,26,26,.85) 60%, rgba(26,26,26,0) 100%);
  backdrop-filter: blur(4px);
  border-bottom:1px solid #2a2a2a;
}
/* se a viewport for baixa, desgruda a toolbar pra não esmagar conteúdo */
@media (max-height: 720px){
  .toolbar{ position: static; }
}

.search-box{ position:relative; }
.search-icon{
  position:absolute; left:12px; top:50%; transform:translateY(-50%);
  width:18px; height:18px; color:#8a8a8a; pointer-events:none;
}
.search-input{
  width:100%; padding:12px 14px 12px 38px; border:1px solid #333; border-radius:10px;
  font-size:14px; background:#232323; color:#e5e5e5;
}
.search-input:focus{ outline:none; border-color:#4f9cf9; box-shadow:0 0 0 1px #4f9cf9; }
.search-input::placeholder{ color:#808080; }

.segmented{ display:flex; gap:12px; align-items:center; justify-content:flex-end; flex-wrap:wrap; }
.segmented__group{ display:inline-flex; background:#222; border:1px solid #333; border-radius:999px; padding:3px; }
.segmented__btn{
  padding:6px 10px; font-size:12px; color:#bdbdbd; background:transparent; border:0;
  border-radius:999px; cursor:pointer; transition:.15s;
}
.segmented__btn:hover{ color:#fff; background:#2b2b2b; }
.segmented__btn.is-active{ color:#fff; background:#3a3a3a; }

.sorter{ display:inline-flex; gap:8px; align-items:center; flex-wrap:nowrap; }
.sorter__label{ font-size:12px; color:#a0a0a0; }
.sorter__select{
  -webkit-appearance:none; -moz-appearance:none; appearance:none;
  padding:8px 28px 8px 10px; border:1px solid #333; border-radius:8px;
  background:#232323; color:#e5e5e5; font-size:12px;
  background-image:
    linear-gradient(45deg, transparent 50%, #9aa0a6 50%),
    linear-gradient(135deg, #9aa0a6 50%, transparent 50%);
  background-repeat:no-repeat; background-size:6px 6px, 6px 6px;
  background-position:right 10px center, right 6px center;
}

/* ----- Lista / Grid ----- */
/* centraliza somente o conteúdo rolável, não o container inteiro */
.events-list{ width:100%; max-width:1200px; margin-inline:auto; }

.events-grid{
  display:grid;
  grid-template-columns: repeat(auto-fill, minmax(360px, 1fr)); /* auto-fill evita "buraco" à esquerda */
  gap:16px; margin-bottom:24px;
  align-items:stretch; justify-items:stretch;
}
.event-card{
  width:100%;
  background:#232323; border:1px solid #323232; border-radius:14px; overflow:hidden;
  transition:border-color .2s, transform .2s, box-shadow .2s;
}
.event-card:hover{ border-color:#4a4a4a; transform:translateY(-1px); box-shadow:0 8px 18px rgba(0,0,0,.25); }
.event-card--inactive{ opacity:.85; }
.event-card__header{
  display:flex; justify-content:space-between; align-items:flex-start; gap:12px;
  padding:16px; border-bottom:1px solid #2f2f2f; background:#2b2b2b;
}
.event-card__title-section{ display:grid; gap:6px; min-width:0; }
.event-card__title{ font-size:18px; font-weight:700; margin:0; color:#fff; overflow:hidden; text-overflow:ellipsis; white-space:nowrap; }
.event-card__meta{ display:inline-flex; gap:8px; align-items:center; flex-wrap:wrap; }
.pill{ padding:2px 8px; border-radius:999px; border:1px solid #3a3a3a; font-size:12px; color:#d6d6d6; background:#262626; }
.event-card__status{ font-size:12px; font-weight:600; padding:4px 8px; border-radius:999px; }
.event-card__status--active{ background:#0b2e22; color:#10b981; border:1px solid #0f5132; }
.event-card__status--inactive{ background:#2f3136; color:#9ca3af; border:1px solid #3a3a3a; }
.event-card__actions{ display:inline-flex; gap:8px; }

.event-card__content{ padding:16px; }
.event-info{ display:grid; gap:8px; margin-bottom:12px; }
.event-info__item{ display:grid; grid-template-columns: 140px 1fr; gap:10px; align-items:baseline; }
.event-info__label{ font-weight:500; color:#a0a0a0; font-size:13px; }
.event-info__value{ color:#e5e5e5; font-size:14px; }
.cut{ overflow:hidden; text-overflow:ellipsis; white-space:nowrap; }

/* Destinatários */
.clients-toggle{
  display:inline-flex; align-items:center; gap:8px;
  background:#2b2b2b; border:1px solid #3a3a3a; border-radius:8px;
  padding:8px 12px; font-size:12px; color:#c9c9c9; cursor:pointer;
}
.clients-toggle:hover{ background:#333; color:#fff; }
.chev{ display:inline-block; transition:transform .2s; }
.chev--open{ transform:rotate(90deg); }

.clients-list{ margin-top:14px; border-top:1px solid #2f2f2f; padding-top:12px; }
.clients-grid{ display:flex; flex-direction:column; gap:8px; }
.client-card{
  display:flex; justify-content:space-between; align-items:center;
  padding:10px 12px; background:#1f1f1f; border:1px solid #2d2d2d; border-radius:10px;
}
.client-name{ font-weight:600; color:#fff; font-size:14px; }
.client-phone{ color:#a0a0a0; font-size:12px; }

/* Paginação */
.pagination{
  display:grid; grid-template-columns:1fr auto 1fr; gap:12px; align-items:center;
  margin-top:8px; padding:10px 0; border-top:1px solid #2a2a2a;
}
.pagination__left, .pagination__center, .pagination__right{ display:flex; align-items:center; gap:8px; }
.pagination__left{ justify-content:flex-start; }
.pagination__center{ justify-content:center; }
.pagination__right{ justify-content:flex-end; }
.pagination__label{ font-size:12px; color:#a0a0a0; }
.pagination__select{
  -webkit-appearance:none; -moz-appearance:none; appearance:none;
  padding:8px 28px 8px 10px; border:1px solid #333; border-radius:8px;
  background:#232323; color:#e5e5e5; font-size:12px;
  background-image:
    linear-gradient(45deg, transparent 50%, #9aa0a6 50%),
    linear-gradient(135deg, #9aa0a6 50%, transparent 50%);
  background-repeat:no-repeat; background-size:6px 6px, 6px 6px;
  background-position:right 10px center, right 6px center;
}
.pagination-info{ font-size:13px; color:#a0a0a0; }

/* Empty / Loading */
.empty-state{
  text-align:center; padding:64px 24px; background:#232323; border-radius:14px; border:1px solid #323232;
}
.empty-state__icon{ font-size:64px; margin-bottom:16px; }
.empty-state__title{ font-size:22px; font-weight:700; margin:0 0 8px 0; color:#fff; }
.empty-state__description{ font-size:15px; color:#a0a0a0; margin:0 0 24px 0; }
.empty-actions{ display:flex; gap:10px; justify-content:center; flex-wrap:wrap; }

.loading-state{ text-align:center; padding:64px; color:#a0a0a0; }
.loading-spinner{ width:32px; height:32px; border:3px solid #3a3a3a; border-top-color:#4f9cf9; border-radius:50%; animation: spin 1s linear infinite; margin:0 auto 16px; }
@keyframes spin{ to{ transform: rotate(360deg); } }

/* Modal(s) */
.modal-overlay{ position:fixed; inset:0; background:rgba(0,0,0,.7); display:flex; align-items:center; justify-content:center; z-index:1000; padding:20px; }
.modal-content{ background:#2a2a2a; border-radius:12px; border:1px solid #3a3a3a; box-shadow:0 20px 25px -5px rgba(0,0,0,.3); width:100%; max-width:980px; max-height:90vh; overflow:hidden; }
.modal-content--sm{ max-width:640px; }
.modal-header{ display:flex; justify-content:space-between; align-items:center; padding:16px 20px; border-bottom:1px solid #3a3a3a; background:#333; }
.modal-title{ font-size:18px; font-weight:700; margin:0; color:#fff; }
.modal-body{ max-height: calc(90vh - 80px); overflow-y:auto; padding:16px; }
.preview-box{
  white-space:pre-wrap; background:#1f1f1f; color:#e5e5e5; border:1px solid #343434;
  border-radius:10px; padding:12px; font-size:14px; line-height:1.45;
}

/* Buttons */
.btn{ display:inline-flex; align-items:center; justify-content:center; padding:8px 16px; font-size:14px; font-weight:600; border-radius:10px; border:none; cursor:pointer; transition:all .15s; text-decoration:none; white-space:nowrap; }
.btn:disabled{ opacity:.6; cursor:not-allowed; }
.btn--primary{ background:#4f9cf9; color:#fff; box-shadow:0 6px 14px rgba(79,156,249,.22); }
.btn--primary:hover:not(:disabled){ background:#3b82f6; transform: translateY(-1px); }
.btn--secondary{ background:#2a2a2a; color:#e5e5e5; border:1px solid #3a3a3a; }
.btn--secondary:hover:not(:disabled){ background:#333; }
.btn--ghost{ background:transparent; color:#bdbdbd; border:1px solid transparent; }
.btn--ghost:hover:not(:disabled){ background:#333; color:#fff; }
.btn--danger:hover:not(:disabled){ color:#ef4444; }
.btn--small{ padding:6px 12px; font-size:12px; }
.btn--tiny{ padding:4px 8px; font-size:11px; }
.btn--large{ padding:12px 24px; font-size:16px; }

/* Responsivo */
@media (max-width: 900px){
  .toolbar{ grid-template-columns:1fr; row-gap:10px; }
  .segmented{ justify-content:space-between; }
}
@media (max-width: 768px){
  .events-container{ padding:16px; }
  .events-header{ flex-direction:column; align-items:stretch; }
  .events-actions{ width:100%; }
  .events-actions .btn{ flex:1; }
  .events-grid{ grid-template-columns:1fr; }
  .event-card__header{ flex-direction:column; gap:10px; align-items:stretch; }
  .event-card__actions{ justify-content:stretch; }
  .event-card__actions .btn{ flex:1; }
  .event-info__item{ grid-template-columns:120px 1fr; }
}
</style>