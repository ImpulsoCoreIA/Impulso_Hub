<template>
  <div class="event-registration text-n-slate-12">
    <div class="event-registration__header">
      <h1 class="event-registration__title text-n-slate-12">
        {{ isEdit ? 'Editar Agendamento' : 'Novo Agendamento' }}
      </h1>
      <p class="event-registration__description text-n-slate-10">
        Configure envio automático por WhatsApp ou Email (SES)
      </p>
    </div>

    <div class="event-registration__content">
      <form @submit.prevent="submit" class="event-form bg-n-solid-1 border-n-container">
        <!-- bloco 1: nome/canal -->
        <div class="grid two-cols">
          <div class="event-form__group">
            <label class="event-form__label text-n-slate-11">Nome (único)</label>
            <input
              class="event-form__input text-n-slate-12"
              v-model.trim="form.name"
              :disabled="isEdit"
              placeholder="boas-vindas"
              required
            />
          </div>

          <div class="event-form__group">
            <label class="event-form__label text-n-slate-11">Canal</label>
            <select class="event-form__input text-n-slate-12" v-model="form.channel">
              <option value="whatsapp">WhatsApp</option>
              <option value="email">Email (SES)</option>
            </select>
          </div>
        </div>

        <!-- Agente/Remetente (WhatsApp) -->
        <div class="event-form__group" v-if="form.channel==='whatsapp'">
          <label class="event-form__label text-n-slate-11">Agente/Remetente (WhatsApp)</label>
          <select class="event-form__input text-n-slate-12" v-model="form.agent">
            <option disabled value="">Selecionar...</option>
            <option v-for="a in agents" :key="a.id" :value="a.number">
              {{ a.label }}
            </option>
          </select>
          <p class="hint text-n-slate-10">Mensagens serão enviadas a partir deste número.</p>
        </div>

        <!-- Variáveis personalizadas -->
        <div class="event-form__group">
          <label class="event-form__label text-n-slate-11">Variáveis personalizadas (placeholders)</label>
          <div class="dest-actions dest-actions--wrap">
            <input
              class="event-form__input dest-actions__grow text-n-slate-12"
              v-model="_newVar"
              placeholder="Ex.: var1, orderId"
              @keyup.enter.prevent="addVar"
            />
            <button type="button" class="btn btn--secondary" @click="addVar">Adicionar variável</button>
            <span class="hint text-n-slate-10">
              Use-as nas mensagens como <code v-pre>{{var1}}</code>, <code v-pre>{{orderId}}</code>.
            </span>
          </div>
          <div v-if="form.customFields?.length" class="hint hint--pills text-n-slate-10">
            <span>Atuais:</span>
            <code v-for="v in form.customFields" :key="v" class="pill">{{ v }}</code>
          </div>
        </div>

        <!-- Destinatários -->
        <div class="event-form__group">
          <div class="dest-toolbar">
            <label class="event-form__label text-n-slate-11">Destinatários</label>
            <div class="dest-actions">
              <button type="button" class="btn btn--secondary" @click="addRecipient">Adicionar</button>

              <!-- Importar CSV -->
              <label class="btn btn--secondary" title="Importar CSV">
                Importar CSV
                <input
                  type="file"
                  accept=".csv,text/csv"
                  class="visually-hidden"
                  @change="handleCsvUpload"
                />
              </label>

              <button type="button" class="btn btn--ghost" @click="downloadCsvTemplate">
                Baixar modelo CSV
              </button>
            </div>
          </div>

          <p class="hint text-n-slate-10">
            CSV com cabeçalho:
            <template v-if="form.channel==='email'">
              <code>name</code>/<code>nome</code>, <code>email</code>/<code>e-mail</code>
            </template>
            <template v-else>
              <code>name</code>/<code>nome</code>, <code>phone</code>/<code>telefone</code>/<code>celular</code>/<code>whatsapp</code>
            </template>
            (delimitador vírgula <code>,</code> ou ponto e vírgula <code>;</code>). As colunas extras viram variáveis.
          </p>

          <div class="recipients-grid">
            <div v-for="(r, idx) in form.recipients" :key="idx" class="recipient-row card bg-n-background border-n-container">
              <div class="recipient-col">
                <label class="event-form__label text-n-slate-11">Nome</label>
                <input class="event-form__input text-n-slate-12" v-model="r.name" placeholder="Ana" />
              </div>
              <div class="recipient-col" v-if="form.channel==='email'">
                <label class="event-form__label text-n-slate-11">Email</label>
                <input class="event-form__input text-n-slate-12" v-model="r.email" placeholder="ana@exemplo.com" />
              </div>
              <div class="recipient-col" v-else>
                <label class="event-form__label text-n-slate-11">Telefone (WhatsApp)</label>
                <input class="event-form__input text-n-slate-12" v-model="r.phone" placeholder="+5532999999999" />
              </div>

              <!-- Variáveis dinâmicas -->
              <div class="recipient-col" v-for="key in form.customFields" :key="key + '-' + idx">
                <label class="event-form__label text-n-slate-11">{{ key }}</label>
                <input class="event-form__input text-n-slate-12" v-model="r.vars[key]" :placeholder="key" />
              </div>

              <div class="recipient-actions">
                <button type="button" class="btn btn--ghost" @click="removeRecipient(idx)">Remover</button>
              </div>
            </div>

            <p v-if="!form.recipients.length" class="event-form__error">
              Adicione manualmente ou importe via CSV.
            </p>

            <p v-if="csvReport" class="csv-report" :class="csvReport.ok ? 'ok' : 'err'">
              {{ csvReport.msg }}
            </p>
          </div>
        </div>

        <!-- Payload (Email ou WhatsApp) -->
        <div class="grid two-cols" v-if="form.channel==='email'">
          <div class="event-form__group">
            <label class="event-form__label text-n-slate-11">Assunto</label>
            <input class="event-form__input text-n-slate-12" v-model="form.payload.subject" placeholder="Mensagem" />
          </div>
          <div></div>

          <div class="event-form__group col-span-2">
            <label class="event-form__label text-n-slate-11">Texto/Corpo</label>
            <textarea
              class="event-form__textarea text-n-slate-12"
              rows="5"
              v-model="form.payload.text"
              placeholder="Olá {{name}}!"
            ></textarea>
          </div>

          <div class="event-form__group col-span-2">
            <label class="event-form__label text-n-slate-11">HTML (opcional)</label>
            <textarea
              class="event-form__textarea text-n-slate-12"
              rows="5"
              v-model="form.payload.html"
              placeholder="<p>Olá {{name}}!</p>"
            ></textarea>
          </div>
        </div>

        <div v-else class="event-form__group">
          <label class="event-form__label text-n-slate-11">Mensagem padrão (fallback)</label>
          <textarea
            class="event-form__textarea text-n-slate-12"
            rows="4"
            v-model="form.payload.message"
            placeholder="Olá {{name}}!"
          ></textarea>
          <p class="hint text-n-slate-10">Para recorrentes, você pode definir uma <b>mensagem por dia</b> abaixo.</p>
        </div>

        <!-- Tipo de agendamento -->
        <div class="event-form__group box bg-n-background border-n-weak">
          <div class="row">
            <input type="checkbox" v-model="oneShot" id="oneshot" />
            <label for="oneshot" class="text-n-slate-12">Agendar uma única vez (runAt, UTC)</label>
          </div>

          <div v-if="oneShot" class="grid oneshot-grid">
            <div>
              <label class="event-form__label text-n-slate-11">Executar em (ISO UTC, sufixo Z)</label>
              <input class="event-form__input text-n-slate-12" v-model="form.runAt" placeholder="2025-09-05T15:00:00Z" />
            </div>
            <div>
              <label class="event-form__label text-n-slate-11">Timezone</label>
              <input class="event-form__input" value="— não aplicável —" disabled />
            </div>
          </div>

          <div v-else class="grid recurrent-grid">
            <div>
              <label class="event-form__label text-n-slate-11">Dias da semana</label>
              <div class="dow">
                <label v-for="d in DOW" :key="d.value" class="dow-item text-n-slate-12">
                  <input type="checkbox" :value="d.value" v-model="form.daysOfWeek" />
                  <span>{{ d.label }}</span>
                </label>
              </div>
            </div>
            <div>
              <label class="event-form__label text-n-slate-11">Horário (HH:mm)</label>
              <input class="event-form__input text-n-slate-12" v-model="form.time" placeholder="08:00" />
            </div>
            <div>
              <label class="event-form__label text-n-slate-11">Timezone</label>
              <input class="event-form__input text-n-slate-12" v-model="form.timezone" placeholder="America/Sao_Paulo" />
            </div>
          </div>

          <!-- Mensagens por dia (WhatsApp + recorrente) -->
          <div v-if="!oneShot && form.channel==='whatsapp'" class="event-form__group">
            <label class="event-form__label text-n-slate-11">Mensagens por dia selecionado</label>
            <div class="grid two-cols">
              <div v-for="d in form.daysOfWeek" :key="d" class="event-form__group">
                <label class="event-form__label text-n-slate-11">{{ labelFor(d) }}</label>
                <textarea
                  class="event-form__textarea text-n-slate-12"
                  rows="4"
                  v-model="form.payload.messagesByDay[d]"
                  :placeholder="`Mensagem para ${labelFor(d)} (suporta {{name}} e variáveis personalizadas)`"
                ></textarea>
                <p v-if="!dayHasMessage(d)" class="event-form__error">Obrigatório para {{ labelFor(d) }}</p>
              </div>
            </div>
            <p class="hint text-n-slate-10">
              Se algum dia não tiver mensagem, o formulário não permitirá salvar.
              O campo "Mensagem padrão" acima será usado como <i>fallback</i> quando não houver mensagem por dia.
            </p>
          </div>

          <!-- Vigência -->
          <div class="event-form__group">
            <label class="event-form__label text-n-slate-11">Vigência (UTC)</label>
            <div class="grid two-cols">
              <div>
                <label class="event-form__label text-n-slate-11">Início (startAt, ISO)</label>
                <input class="event-form__input text-n-slate-12" v-model="form.startAt" placeholder="2025-09-01T00:00:00Z" />
              </div>
              <div>
                <label class="event-form__label text-n-slate-11">Fim (endAt, ISO)</label>
                <input class="event-form__input text-n-slate-12" v-model="form.endAt" placeholder="2025-12-31T23:59:59Z" />
              </div>
            </div>
            <p v-if="!validDateRange" class="event-form__error">A data de fim deve ser posterior à data de início.</p>
            <p class="hint text-n-slate-10">O envio só ocorrerá se o horário atual estiver dentro dessa janela.</p>
          </div>

          <div class="event-form__group" style="margin-top:12px">
            <label class="event-form__checkbox-container">
              <input type="checkbox" v-model="form.enabled" class="event-form__checkbox" />
              <span class="event-form__checkbox-label text-n-slate-12">Agendamento habilitado</span>
            </label>
          </div>
        </div>

        <!-- Ações fixas -->
        <div class="event-form__actions actions--sticky">
          <button type="submit" class="btn btn--primary" :disabled="submitting || !isValid">
            {{ submitting ? 'Salvando...' : (isEdit ? 'Salvar alterações' : 'Criar agendamento') }}
          </button>
        </div>
      </form>

      <div v-if="status.show" class="alert" :class="status.ok ? 'alert--success' : 'alert--error'">
        <h3 class="alert__title">{{ status.ok ? '✓ Sucesso' : '✗ Erro' }}</h3>
        <p class="alert__message">{{ status.msg }}</p>
      </div>
    </div>
  </div>
</template>

<script>
import axios from 'axios'

const API_BASE = 'https://f4wzfjousg.execute-api.us-east-1.amazonaws.com/schedules'
const AGENTS_API = 'https://f4wzfjousg.execute-api.us-east-1.amazonaws.com/get-agents-list'
const DOW_LABEL = { SUN: 'Dom', MON: 'Seg', TUE: 'Ter', WED: 'Qua', THU: 'Qui', FRI: 'Sex', SAT: 'Sáb' }

export default {
  name: 'EventForm',
  props: { value: { type: Object, default: null } },
  data() {
    return {
      api: axios.create({ baseURL: API_BASE, headers: { 'Content-Type': 'application/json' } }),
      agents: [],
      agentsLoading: false,
      DOW: [
        { value: 'MON', label: 'Seg' }, { value: 'TUE', label: 'Ter' }, { value: 'WED', label: 'Qua' },
        { value: 'THU', label: 'Qui' }, { value: 'FRI', label: 'Sex' }, { value: 'SAT', label: 'Sáb' }, { value: 'SUN', label: 'Dom' }
      ],
      form: {
        name: '',
        channel: 'whatsapp',
        agent: '',
        recipients: [],
        payload: { message: 'Olá {{name}}!', messagesByDay: {} },
        customFields: [],
        enabled: true,
        daysOfWeek: ['MON'],
        time: '08:00',
        timezone: 'America/Sao_Paulo',
        runAt: '',
        startAt: '',
        endAt: ''
      },
      oneShot: false,
      submitting: false,
      status: { show: false, ok: true, msg: '' },
      csvReport: null,
      _newVar: ''
    }
  },
  computed: {
    isEdit() { return !!this.value?.Name },
    validDateRange() {
      const s = this.form.startAt?.trim(), e = this.form.endAt?.trim()
      if (!s || !e) return true
      const sd = new Date(s), ed = new Date(e)
      if (isNaN(sd.getTime()) || isNaN(ed.getTime())) return true
      return ed > sd
    },
    isValid() {
      if (!this.form.name) return false
      if (!this.form.recipients.length) return false

      if (this.form.channel === 'email') {
        if (!this.form.recipients.every(r => !!r.email)) return false
      } else {
        if (!this.form.recipients.every(r => !!r.phone)) return false
        if (!this.form.agent) return false
      }

      if (!this.validDateRange) return false

      if (this.oneShot) {
        return !!this.form.runAt
      } else {
        const baseOk = this.form.daysOfWeek.length > 0 && !!this.form.time && !!this.form.timezone
        if (!baseOk) return false
        if (this.form.channel === 'whatsapp') {
          return this.form.daysOfWeek.every(d => !!(this.form.payload?.messagesByDay?.[d] || '').trim())
        }
        return true
      }
    }
  },
  watch: {
    value: {
      immediate: true,
      handler(v) {
        if (!v) return
        this.form.name = v.Name || ''
        this.form.channel = v.Channel || 'whatsapp'
        this.form.agent = v.Agent || ''
        this.form.recipients = Array.isArray(v.Recipients) ? JSON.parse(JSON.stringify(v.Recipients)) : []

        const varSet = new Set()
        for (const r of this.form.recipients) {
          if (r && typeof r.vars === 'object') {
            Object.keys(r.vars).forEach(k => varSet.add(this.normalizeVarKey(k)))
          }
        }
        this.form.customFields = Array.from(varSet)

        const defaultPayload = this.form.channel === 'email'
          ? { subject: '', text: '', html: '' }
          : { message: 'Olá {{name}}!', messagesByDay: {} }
        const fromPayload = JSON.parse(JSON.stringify(v.Payload || defaultPayload))
        if (this.form.channel === 'whatsapp' && !fromPayload.messagesByDay) fromPayload.messagesByDay = {}
        this.form.payload = fromPayload
        this.form.enabled = !!v.Enabled

        this.form.startAt = v.StartAt || ''
        this.form.endAt = v.EndAt || ''

        if (v.RunAt) {
          this.oneShot = true
          this.form.runAt = v.RunAt
          this.form.daysOfWeek = []
          this.form.time = ''
          this.form.timezone = ''
        } else {
          this.oneShot = false
          this.form.daysOfWeek = v.DaysOfWeek || ['MON']
          this.form.time = v.Time || '08:00'
          this.form.timezone = v.Timezone || 'America/Sao_Paulo'
          this.form.runAt = ''
        }
      }
    },
    'form.channel'(ch) {
      this.form.recipients = (this.form.recipients || []).map(r => ({
        name: r.name || '',
        email: ch === 'email' ? (r.email || '') : undefined,
        phone: ch === 'whatsapp' ? (r.phone || '') : undefined,
        vars: r.vars || {}
      }))
      if (ch === 'email') {
        this.form.payload = {
          subject: this.form.payload.subject || '',
          text: this.form.payload.text || '',
          html: this.form.payload.html || ''
        }
      } else {
        this.form.payload = {
          message: this.form.payload.message || 'Olá {{name}}!',
          messagesByDay: this.form.payload.messagesByDay || {}
        }
      }
    },
    'form.daysOfWeek': {
      deep: true,
      handler(days) {
        if (this.form.channel !== 'whatsapp') return
        this.form.payload.messagesByDay = this.form.payload.messagesByDay || {}
        days.forEach(d => {
          if (!this.form.payload.messagesByDay[d]) this.$set(this.form.payload.messagesByDay, d, '')
        })
        Object.keys(this.form.payload.messagesByDay).forEach(k => {
          if (!days.includes(k)) this.$delete(this.form.payload.messagesByDay, k)
        })
      }
    },
    'form.customFields': {
      deep: true,
      handler(list) {
        const keys = (list || []).map(this.normalizeVarKey).filter(Boolean)
        for (const r of this.form.recipients) {
          r.vars = r.vars || {}
          Object.keys(r.vars).forEach(k => { if (!keys.includes(this.normalizeVarKey(k))) this.$delete(r.vars, k) })
          keys.forEach(k => { if (!(k in r.vars)) this.$set(r.vars, k, '') })
        }
      }
    }
  },
  methods: {
    labelFor(d) { return DOW_LABEL[d] || d },
    dayHasMessage(d) { return !!(this.form.payload?.messagesByDay?.[d] || '').trim() },
    normalizeHeader(s) {
      return (s || '')
        .toString()
        .trim()
        .toLowerCase()
        .normalize('NFD')
        .replace(/[\u0300-\u036f]/g, '')
        .replace(/[^a-z0-9_ -]/g, '')
    },
    normalizeVarKey(s) {
      return (s || '').toString().trim().toLowerCase().replace(/[^a-z0-9_\.\-]/g, '_')
    },
    async loadAgents() {
      this.agentsLoading = true
      try {
        const { data } = await axios.get(AGENTS_API)
        this.agents = (data?.items || []).map(x => ({ id: x.id, label: x.label, number: x.number }))
        if (!this.form.agent && this.agents.length) this.form.agent = this.agents[0].number
      } catch (e) {
        console.error('Falha ao carregar agentes', e)
      } finally { this.agentsLoading = false }
    },
    addVar() {
      const k = this.normalizeVarKey(this._newVar)
      if (!k) return
      if (!this.form.customFields.includes(k)) {
        this.form.customFields = [...this.form.customFields, k]
      }
      for (const r of this.form.recipients) {
        r.vars = r.vars || {}
        if (!(k in r.vars)) this.$set(r.vars, k, '')
      }
      this._newVar = ''
    },
    addRecipient() {
      this.form.recipients.push({
        name: '',
        phone: this.form.channel === 'whatsapp' ? '' : undefined,
        email: this.form.channel === 'email' ? '' : undefined,
        vars: Object.fromEntries((this.form.customFields || []).map(k => [this.normalizeVarKey(k), '']))
      })
    },
    removeRecipient(i) { this.form.recipients.splice(i, 1) },
    detectDelimiter(firstLine) {
      const commas = (firstLine.match(/,/g) || []).length
      const semis = (firstLine.match(/;/g) || []).length
      return semis > commas ? ';' : ','
    },
    parseCsvText(text) {
      const lines = text.split(/\r?\n/).filter(l => l.trim() !== '')
      if (!lines.length) return { headers: [], rows: [], originalHeaders: [] }
      const delimiter = this.detectDelimiter(lines[0])
      const split = (line) => {
        const out = []
        let cur = '', inQuotes = false
        for (let i = 0; i < line.length; i++) {
          const ch = line[i]
          if (ch === '"') {
            if (line[i + 1] === '"') { cur += '"'; i++; continue }
            inQuotes = !inQuotes
          } else if (ch === delimiter && !inQuotes) {
            out.push(cur); cur = ''
          } else {
            cur += ch
          }
        }
        out.push(cur)
        return out.map(s => s.trim())
      }
      const originalHeaders = split(lines[0])
      const headers = originalHeaders.map(this.normalizeHeader)
      const rows = lines.slice(1).map(split)
      return { headers, rows, originalHeaders }
    },
    handleCsvUpload(e) {
      const file = e.target.files && e.target.files[0]
      if (!file) return
      const reader = new FileReader()
      reader.onload = () => {
        try {
          const text = reader.result?.toString() || ''
          const { headers, rows, originalHeaders } = this.parseCsvText(text)
          if (!headers.length || !rows.length) {
            this.csvReport = { ok: false, msg: 'CSV vazio ou sem cabeçalho.' }
            return
          }
          const hset = new Map(headers.map((h, i) => [h, i]))
          const nameKey = ['name', 'nome'].find(k => hset.has(k))
          const emailKey = ['email', 'e-mail', 'e_mail'].find(k => hset.has(k))
          const phoneKey = ['phone', 'telefone', 'celular', 'whatsapp'].find(k => hset.has(k))
          const needsEmail = this.form.channel === 'email'
          const okHeaders = needsEmail ? (nameKey && emailKey) : (nameKey && phoneKey)
          if (!okHeaders) {
            this.csvReport = {
              ok: false,
              msg: needsEmail
                ? 'Cabeçalho inválido. Esperado: name/nome, email/e-mail.'
                : 'Cabeçalho inválido. Esperado: name/nome, phone/telefone/celular/whatsapp.'
            }
            return
          }

          const known = new Set([nameKey, emailKey, phoneKey].filter(Boolean))
          const varCols = headers
            .map((h, i) => ({ h, i }))
            .filter(({ h }) => !!h && !known.has(h))

          const newVarNames = varCols.map(({ i, h }) => this.normalizeVarKey(originalHeaders[i] || h))
          const merged = Array.from(new Set([...(this.form.customFields || []).map(this.normalizeVarKey), ...newVarNames]))
          this.form.customFields = merged

          const existingKeySet = new Set(
            (this.form.recipients || [])
              .map(r => (needsEmail ? (r.email || '').toLowerCase() : (r.phone || '')))
              .filter(Boolean)
          )

          let imported = 0, skipped = 0
          const newRecipients = []
          for (const row of rows) {
            const name = row[hset.get(nameKey)]?.trim()
            const email = emailKey ? (row[hset.get(emailKey)] || '').trim() : ''
            const phone = phoneKey ? (row[hset.get(phoneKey)] || '').trim() : ''
            if (!name && !email && !phone) { skipped++; continue }

            const vars = Object.fromEntries(
              varCols.map(({ i, h }) => [this.normalizeVarKey(originalHeaders[i] || h), (row[i] || '').trim()])
            )

            if (needsEmail) {
              if (!email || !this.isValidEmail(email)) { skipped++; continue }
              const k = email.toLowerCase()
              if (existingKeySet.has(k)) { skipped++; continue }
              existingKeySet.add(k)
              newRecipients.push({ name, email, vars }); imported++
            } else {
              if (!phone || !this.isLikelyPhone(phone)) { skipped++; continue }
              const normalizedPhone = phone.replace(/\s+/g, '')
              if (existingKeySet.has(normalizedPhone)) { skipped++; continue }
              existingKeySet.add(normalizedPhone)
              newRecipients.push({ name, phone: normalizedPhone, vars }); imported++
            }
          }

          this.form.recipients = [...this.form.recipients, ...newRecipients]
          this.csvReport = { ok: true, msg: `Importados ${imported}. Ignorados ${skipped}. Total agora: ${this.form.recipients.length}.` }
          e.target.value = ''
        } catch (err) {
          console.error(err); this.csvReport = { ok: false, msg: 'Erro ao processar o CSV.' }
        }
      }
      reader.readAsText(file, 'utf-8')
    },
    downloadCsvTemplate() {
      const needsEmail = this.form.channel === 'email'
      const headers = needsEmail ? ['name', 'email'] : ['name', 'phone']
      const sample = needsEmail
        ? [['Ana', 'ana@exemplo.com'], ['Bruno', 'bruno@exemplo.com']]
        : [['Ana', '+5532987654321'], ['Bruno', '+5531999999999']]
      const csvLines = [ headers.join(','), ...sample.map(row => row.map(v => `"${String(v).replace(/"/g, '""')}"`).join(',')) ]
      const csv = csvLines.join('\n')
      const blob = new Blob([csv], { type: 'text/csv;charset=utf-8;' })
      const url = URL.createObjectURL(blob)
      const a = document.createElement('a'); a.href = url
      a.download = needsEmail ? 'recipients_email_template.csv' : 'recipients_whatsapp_template.csv'
      document.body.appendChild(a); a.click(); document.body.removeChild(a); URL.revokeObjectURL(url)
    },
    isValidEmail(email) { return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email) },
    isLikelyPhone(phone) { const digits = phone.replace(/\D/g, ''); return digits.length >= 10 && digits.length <= 15 },
    buildBackendBody() {
      const base = {
        name: this.form.name,
        channel: this.form.channel,
        agent: this.form.channel === 'whatsapp' ? (this.form.agent || undefined) : undefined,
        recipients: this.form.recipients,
        payload: this.form.channel === 'email'
          ? {
              subject: this.form.payload?.subject || 'Mensagem',
              text: this.form.payload?.text || '',
              html: this.form.payload?.html || ''
            }
          : {
              message: this.form.payload?.message || 'Olá {{name}}!',
              messagesByDay: (!this.oneShot && this.form.channel === 'whatsapp')
                ? this.form.daysOfWeek.reduce((acc, d) => {
                    const v = (this.form.payload?.messagesByDay?.[d] || '').trim()
                    if (v) acc[d] = v
                    return acc
                  }, {})
                : undefined
            },
        enabled: !!this.form.enabled,
        startAt: (this.form.startAt || '').trim() || undefined,
        endAt: (this.form.endAt || '').trim() || undefined
      }

      if (this.oneShot) {
        return { ...base, runAt: this.form.runAt }
      }
      return {
        ...base,
        daysOfWeek: this.form.daysOfWeek,
        time: this.form.time,
        timezone: this.form.timezone || 'America/Sao_Paulo'
      }
    },
    async submit() {
      if (!this.isValid) return
      this.submitting = true
      this.status.show = false
      try {
        const body = this.buildBackendBody()
        if (this.isEdit) {
          await this.api.put(`/${encodeURIComponent(this.form.name)}`, body)
          this.ok('Agendamento atualizado com sucesso.')
        } else {
          const res = await this.api.post('', body)
          if (res.status === 200) this.ok('Agendamento criado com sucesso.')
        }
        this.$emit('saved')
      } catch (e) {
        console.error('Falha ao salvar:', e.response?.data || e)
        const msg = e?.response?.data?.error || 'Falha ao salvar. Verifique CORS do POST e os campos obrigatórios.'
        this.err(msg)
      } finally { this.submitting = false }
    },
    ok(msg) { this.status = { show: true, ok: true, msg } },
    err(msg) { this.status = { show: true, ok: false, msg } }
  },
  mounted() {
    this.loadAgents()
  }
}
</script>

<style scoped>
/* ===== Layout base ===== */
.event-registration {
  min-height: 100%;
  color: var(--n-slate-12);
  padding: 16px;
}
.event-registration__content {
  max-width: 1200px;
  margin: 0 auto;
}
.event-registration__header { margin: 0 auto 24px; text-align: center; max-width: 960px; }
.event-registration__title { font-size: 28px; font-weight: 800; margin-bottom: 8px; letter-spacing: .2px; }
.event-registration__description { font-size: 16px; }

/* ----- Form wrapper ----- */
.event-form {
  padding: 28px;
  border-radius: 16px;
  border: 1px solid var(--n-container);
  margin-bottom: 24px;
  box-shadow: 0 6px 18px rgba(0,0,0,.25);
}
.event-form__group { margin-bottom: 18px; }
.event-form__label { display: block; margin-bottom: 8px; font-weight: 600; font-size: 14px; }

/* ----- Inputs ----- */
.event-form__input,
.event-form__textarea {
  background-color: var(--n-background);
  color: var(--n-slate-12);
  width: 100%;
  padding: 12px 14px;
  border: 1px solid var(--n-container);
  border-radius: 12px;
  font-size: 16px;
  line-height: 1.4;
  transition: border-color .15s, box-shadow .15s, background-color .15s;
  min-height: 44px;
  box-sizing: border-box;
}
.event-form__textarea {
  min-height: 120px;
  resize: vertical;
}
.event-form__input:focus,
.event-form__textarea:focus {
  outline: none;
  border-color: var(--n-brand);
  box-shadow: 0 0 0 2px color-mix(in oklab, var(--n-brand) 25%, transparent);
  background-color: #111317;
}
.event-form__input::placeholder,
.event-form__textarea::placeholder { color: var(--n-slate-10); }
.event-form__error { color: var(--n-ruby-9); font-size: 14px; margin-top: 8px; }

/* select: evitar “corte” e dar mais área clicável */
select.event-form__input {
  appearance: none;
  background-image:
    linear-gradient(45deg, transparent 50%, var(--n-slate-10) 50%),
    linear-gradient(135deg, var(--n-slate-10) 50%, transparent 50%);
  background-position:
    calc(100% - 18px) calc(1em + 2px),
    calc(100% - 14px) calc(1em + 2px);
  background-size: 6px 6px, 6px 6px;
  background-repeat: no-repeat;
  padding-right: 40px;
  white-space: nowrap;
}

/* ----- Grids fluidas ----- */
.grid.two-cols {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(320px, 1fr));
  gap: 20px;
}
.col-span-2 { grid-column: 1 / -1; }

/* toolbar & actions */
.dest-toolbar {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 10px;
  gap: 12px;
  flex-wrap: wrap;
}
.dest-actions {
  display: flex;
  gap: 10px;
  align-items: center;
  flex-wrap: nowrap;
}
.dest-actions--wrap { flex-wrap: wrap; gap: 10px 12px; }
.dest-actions__grow { flex: 1 1 320px; min-width: 260px; }

.visually-hidden {
  position: absolute; width: 1px; height: 1px; padding: 0; margin: -1px;
  overflow: hidden; clip: rect(0,0,0,0); white-space: nowrap; border: 0;
}

.hint { font-size: 13px; margin-top: 6px; }
.hint--pills { display: flex; flex-wrap: wrap; gap: 6px 8px; align-items: center; }
.pill {
  background: var(--n-solid-1);
  border: 1px solid var(--n-container);
  border-radius: 999px;
  padding: 4px 10px;
  font-size: 12px;
}

/* ---- Cards (linhas de destinatários) ---- */
.recipients-grid { display: grid; gap: 12px; }
.card {
  border: 1px solid var(--n-container);
  border-radius: 14px;
  padding: 14px;
}
.recipient-row {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(260px, 1fr));
  gap: 12px;
  align-items: start;
}
.recipient-col { display: flex; flex-direction: column; gap: 6px; }
.recipient-actions { display: flex; align-items: center; justify-content: flex-end; }

/* CSV report */
.csv-report { font-size: 13px; margin-top: 6px; }
.csv-report.ok { color: var(--n-teal-11); }
.csv-report.err { color: var(--n-ruby-11); }

/* ----- Box de agendamento ----- */
.box {
  border: 1px solid var(--n-weak);
  border-radius: 14px;
  padding: 18px;
  background: var(--n-background);
}
.row { display: flex; gap: 12px; align-items: center; margin-bottom: 12px; }

.oneshot-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(300px, 1fr));
  gap: 16px;
}
.recurrent-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(260px, 1fr));
  gap: 16px;
}
.dow { display: flex; gap: 8px; flex-wrap: wrap; }
.dow-item { display: flex; align-items: center; gap: 6px; }
.event-form__checkbox { width: 18px; height: 18px; accent-color: var(--n-brand); }
.event-form__checkbox-container { display: flex; align-items: center; gap: 8px; cursor: pointer; }
.event-form__checkbox-label { }

/* ----- Ações ----- */
.event-form__actions {
  display: flex;
  justify-content: flex-end;
  gap: 10px;
  margin-top: 24px;
}
.actions--sticky {
  position: sticky;
  bottom: 12px;
  background: linear-gradient(180deg, transparent 0%, rgba(0,0,0,.35) 40%, rgba(0,0,0,.55) 100%);
  padding-top: 10px;
  z-index: 3;
}

/* ----- Buttons ----- */
.btn {
  padding: 12px 16px;
  border: none;
  border-radius: 12px;
  font-weight: 700;
  cursor: pointer;
  transition: all .2s;
  font-size: 14px;
}
.btn--primary { background: var(--n-brand); color: #fff; box-shadow: 0 6px 14px rgba(79,156,249,.22); }
.btn--primary:hover:not(:disabled) { background: var(--n-brand-strong); transform: translateY(-1px); }
.btn--secondary { background: var(--n-solid-1); border: 1px solid var(--n-container); color: var(--n-slate-12); }
.btn--secondary:hover:not(:disabled) { background: #222831; }
.btn--ghost { background: transparent; border: 1px solid var(--n-container); color: var(--n-slate-12); }
.btn:disabled { opacity: .6; cursor: not-allowed; }

/* ----- Alerts ----- */
.alert { padding: 16px; border-radius: 12px; margin-bottom: 24px; border: 1px solid var(--n-container); }
.alert--success { background: var(--n-teal-1); color: var(--n-teal-11); }
.alert--error { background: var(--n-ruby-1); color: var(--n-ruby-11); }
.alert__title { font-weight: 800; margin: 0 0 8px 0; }
.alert__message { margin: 0; }

/* ----- Responsivo extra ----- */
@media (max-width: 480px) {
  .event-form { padding: 20px; }
  .event-registration { padding: 10px; }
}
</style>
