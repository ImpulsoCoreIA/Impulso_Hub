<script setup>
import { computed, nextTick, onMounted, reactive, ref, watch } from 'vue';
import axios from 'axios';

import { useAlert } from 'dashboard/composables';
import Button from 'dashboard/components-next/button/Button.vue';

const props = defineProps({
  value: { type: Object, default: null },
});

const emit = defineEmits(['saved', 'close']);

const API_BASE =
  'https://f4wzfjousg.execute-api.us-east-1.amazonaws.com/schedules';
const AGENTS_API =
  'https://f4wzfjousg.execute-api.us-east-1.amazonaws.com/get-agents-list';
const TEMPLATES_API =
  'https://f4wzfjousg.execute-api.us-east-1.amazonaws.com/get-whatsapp-templates-list';

const DAYS_OF_WEEK = [
  { value: 'MON', label: 'Segunda' },
  { value: 'TUE', label: 'Terça' },
  { value: 'WED', label: 'Quarta' },
  { value: 'THU', label: 'Quinta' },
  { value: 'FRI', label: 'Sexta' },
  { value: 'SAT', label: 'Sábado' },
  { value: 'SUN', label: 'Domingo' },
];

const text = Object.freeze({
  name: {
    label: 'Nome do agendamento',
    placeholder: 'boas-vindas',
  },
  channel: {
    label: 'Canal',
    placeholder: 'Selecionar canal',
    options: {
      whatsapp: 'WhatsApp',
      email: 'Email (SES)',
    },
  },
  tooltips: {
    channel: 'Informe um nome para o agendamento antes de escolher o canal.',
    agent: 'Selecione o canal para habilitar os agentes disponíveis.',
  },
  agent: {
    label: 'Agente / remetente',
    hint: 'Mensagens serão enviadas a partir do número selecionado.',
    placeholder: 'Selecionar...',
  },
  customField: {
    placeholder: 'nova_variavel',
    add: 'Adicionar variável',
  },
  recipients: {
    title: 'Destinatários',
    subtitle:
      'Adicione manualmente, importe um CSV ou defina variáveis personalizadas.',
    import: 'Importar CSV',
    download: 'Modelo CSV',
    add: 'Adicionar',
    empty: 'Adicione manualmente ou importe via CSV para iniciar o envio.',
    remove: 'Remover',
    name: 'Nome',
    email: 'Email',
    phone: 'Telefone (WhatsApp)',
    noName: '(sem nome)',
    namePlaceholder: 'Ana',
    emailPlaceholder: 'ana@exemplo.com',
    phonePlaceholder: '+5532999999999',
    csv: {
      empty: 'CSV vazio ou sem cabeçalho.',
      invalidEmail: 'Cabeçalho inválido. Esperado: name/nome, email/e-mail.',
      invalidWhatsapp:
        'Cabeçalho inválido. Esperado: name/nome, phone/telefone/celular/whatsapp.',
      error: 'Erro ao processar o CSV.',
      summary: (imported, skipped, total) =>
        `Importados ${imported}. Ignorados ${skipped}. Total agora: ${total}.`,
    },
  },
  content: {
    title: 'Conteúdo da mensagem',
    description:
      'Personalize o conteúdo com variáveis utilizando o formato {{name}}.',
    subject: 'Assunto',
    subjectPlaceholder: 'Mensagem',
    text: 'Texto (plain)',
    textPlaceholder: 'Olá {{name}}!',
    html: 'HTML (opcional)',
    htmlPlaceholder: '<p>Olá {{name}}!</p>',
    message: 'Mensagem padrão',
    templateLabel: 'Templates',
    templateSelect: {
      loading: 'Carregando templates...',
      placeholder: 'Selecionar template...',
      empty: 'Nenhum template encontrado',
    },
    templateActions: {
      reload: 'Recarregar',
      sync: 'Sincronizar',
      apply: 'Aplicar',
    },
    templatePlaceholders: 'Placeholders do template:',
  },
  placeholders: {
    global: message => message,
    day: message => message,
  },
  schedule: {
    title: 'Agendamento',
    description:
      'Configure uma execução única ou defina os dias/horários recorrentes.',
    runOnce: 'Agendar apenas uma vez (runAt)',
    runAt: 'Executar em (ISO UTC)',
    runAtPlaceholder: '2025-09-05T15:00:00Z',
    timezone: 'Timezone',
    timezoneStatic: '— não aplicável —',
    days: 'Dias da semana',
    time: 'Horário (HH:mm)',
    timePlaceholder: '08:00',
    timezonePlaceholder: 'America/Sao_Paulo',
    dayMessagesTitle: 'Mensagens por dia selecionado',
    messageLabel: day => `Mensagem para ${day}`,
    missingMessage: day => `Obrigatório para ${day}.`,
  },
  timeframe: {
    start: 'Início (startAt, UTC)',
    startPlaceholder: '2025-09-01T00:00:00Z',
    end: 'Fim (endAt, UTC)',
    endPlaceholder: '2025-12-31T23:59:59Z',
    error: 'A data de fim deve ser posterior à data de início.',
  },
  toggle: 'Agendamento habilitado',
  actions: {
    save: 'Salvar agendamento',
    cancel: 'Cancelar',
  },
  status: {
    created: 'Agendamento criado com sucesso.',
    updated: 'Agendamento atualizado com sucesso.',
    error:
      'Falha ao salvar. Verifique os campos obrigatórios e tente novamente.',
  },
  alerts: {
    agents: 'Falha ao carregar agentes do WhatsApp.',
    templates: 'Falha ao carregar templates do WhatsApp.',
    syncSuccess: 'Templates sincronizados com sucesso.',
    syncError: 'Falha ao sincronizar templates.',
  },
});

const api = axios.create({
  baseURL: API_BASE,
  headers: { 'Content-Type': 'application/json' },
});

const agents = ref([]);
const agentsLoading = ref(false);
const templates = ref([]);
const templatesLoading = ref(false);
const selectedTemplateKey = ref('');

const form = reactive({
  name: '',
  channel: '',
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
  endAt: '',
});

const oneShot = ref(false);
const submitting = ref(false);
const status = reactive({ show: false, ok: true, msg: '' });
const csvReport = ref(null);
const newVariable = ref('');
const requiredPlaceholders = ref(new Set());
const placeholderGlobalError = ref('');
const placeholderDaysError = ref('');

const isEdit = computed(() => Boolean(props.value?.Name));
const isNameFilled = computed(() => !!form.name.trim());
const canSelectChannel = computed(() => isNameFilled.value);
const canSelectAgent = computed(() => canSelectChannel.value && !!form.channel);

const validDateRange = computed(() => {
  const start = form.startAt?.trim();
  const end = form.endAt?.trim();
  if (!start || !end) return true;
  const startDate = new Date(start);
  const endDate = new Date(end);
  if (Number.isNaN(startDate.getTime()) || Number.isNaN(endDate.getTime()))
    return true;
  return endDate > startDate;
});

const selectedTemplate = computed(
  () =>
    templates.value.find(
      template => template.key === selectedTemplateKey.value
    ) || null
);

function normalizeVarKey(value) {
  return (value || '')
    .toString()
    .trim()
    .toLowerCase()
    .replace(/[^a-z0-9_.-]/g, '_');
}

function normalizeHeader(value) {
  return (value || '')
    .toString()
    .trim()
    .toLowerCase()
    .normalize('NFD')
    .replace(/[\u0300-\u036f]/g, '')
    .replace(/[^a-z0-9_ -]/g, '');
}

function isValidEmail(email) {
  return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email || '');
}

function isLikelyPhone(phone) {
  const digits = (phone || '').replace(/\D/g, '');
  return digits.length >= 10 && digits.length <= 15;
}

function recipientHasValue(recipient, key) {
  const normalized = normalizeVarKey(key);
  if (normalized === 'name' || normalized === 'nome') {
    return Boolean(recipient.name && recipient.name.trim());
  }
  if (normalized === 'email') {
    return Boolean(recipient.email && recipient.email.trim());
  }
  if (['phone', 'telefone', 'celular', 'whatsapp'].includes(normalized)) {
    return Boolean(recipient.phone && recipient.phone.trim());
  }
  const value = recipient.vars?.[normalized];
  return value !== undefined && String(value).trim() !== '';
}

function extractPlaceholders(textValue) {
  const placeholders = new Set();
  if (!textValue) return placeholders;
  const regex = /\{\{\s*([a-zA-Z0-9_.-]+)\s*\}\}/g;
  let match = regex.exec(textValue);
  while (match) {
    placeholders.add(normalizeVarKey(match[1]));
    match = regex.exec(textValue);
  }
  return placeholders;
}

function placeholdersSatisfied() {
  const requiredKeys = Array.from(requiredPlaceholders.value || []);
  if (!requiredKeys.length) {
    placeholderGlobalError.value = '';
    placeholderDaysError.value = '';
    return true;
  }

  const missingCounter = new Map(requiredKeys.map(key => [key, 0]));
  form.recipients.forEach(recipient => {
    requiredKeys.forEach(key => {
      if (!recipientHasValue(recipient, key)) {
        missingCounter.set(key, missingCounter.get(key) + 1);
      }
    });
  });

  const missingList = Array.from(missingCounter.entries()).filter(
    ([, count]) => count > 0
  );
  if (!missingList.length) {
    placeholderGlobalError.value = '';
    placeholderDaysError.value = '';
    return true;
  }

  const message = `Preencha as variáveis requeridas nos destinatários: ${missingList
    .map(([key, count]) => `${key} (${count})`)
    .join(', ')}.`;

  placeholderGlobalError.value = text.placeholders.global(message);
  placeholderDaysError.value =
    !oneShot.value && form.channel === 'whatsapp'
      ? text.placeholders.day(message)
      : '';

  return false;
}

const isValid = computed(() => {
  if (!form.name || !form.name.trim()) return false;
  if (!form.channel) return false;
  if (!form.recipients.length) return false;

  if (form.channel === 'email') {
    const everyHasEmail = form.recipients.every(recipient =>
      Boolean(recipient.email)
    );
    if (!everyHasEmail) return false;
  } else {
    const everyHasPhone = form.recipients.every(recipient =>
      Boolean(recipient.phone)
    );
    if (!everyHasPhone) return false;
    if (!form.agent) return false;
  }

  if (!validDateRange.value) return false;
  if (!placeholdersSatisfied()) return false;

  if (oneShot.value) {
    return Boolean(form.runAt);
  }

  const dayConfigurationValid =
    form.daysOfWeek.length > 0 && Boolean(form.time) && Boolean(form.timezone);
  if (!dayConfigurationValid) return false;

  if (form.channel === 'whatsapp') {
    const allDaysHaveMessage = form.daysOfWeek.every(day => {
      const content = form.payload?.messagesByDay?.[day] || '';
      return Boolean(content.trim());
    });
    if (!allDaysHaveMessage) return false;
  }

  return true;
});

function recomputeRequiredPlaceholders() {
  const bucket = new Set();
  if (form.channel === 'email') {
    ['subject', 'text', 'html'].forEach(key => {
      extractPlaceholders(form.payload?.[key]).forEach(value =>
        bucket.add(value)
      );
    });
  } else {
    extractPlaceholders(form.payload?.message).forEach(value =>
      bucket.add(value)
    );
    if (!oneShot.value) {
      const byDay = form.payload?.messagesByDay || {};
      form.daysOfWeek.forEach(day => {
        extractPlaceholders(byDay[day]).forEach(value => bucket.add(value));
      });
    }
  }
  requiredPlaceholders.value = bucket;
  placeholderGlobalError.value = '';
  placeholderDaysError.value = '';
  placeholdersSatisfied();
}

function resetForm() {
  form.name = '';
  form.channel = '';
  form.agent = '';
  form.recipients = [];
  form.payload = { message: 'Olá {{name}}!', messagesByDay: {} };
  form.customFields = [];
  form.enabled = true;
  form.daysOfWeek = ['MON'];
  form.time = '08:00';
  form.timezone = 'America/Sao_Paulo';
  form.runAt = '';
  form.startAt = '';
  form.endAt = '';
  oneShot.value = false;
  selectedTemplateKey.value = '';
  csvReport.value = null;
  status.show = false;
  status.msg = '';
  status.ok = true;
  newVariable.value = '';
  requiredPlaceholders.value = new Set();
  placeholderGlobalError.value = '';
  placeholderDaysError.value = '';
}

function hydrateFromValue(value) {
  if (!value) {
    resetForm();
    recomputeRequiredPlaceholders();
    if (form.channel === 'whatsapp' && form.agent) {
      loadTemplates();
    } else {
      templates.value = [];
      selectedTemplateKey.value = '';
    }
    return;
  }

  form.name = value.Name || '';
  form.channel = value.Channel || '';
  form.agent = value.Agent || '';
  form.recipients = Array.isArray(value.Recipients)
    ? JSON.parse(JSON.stringify(value.Recipients))
    : [];

  const collected = new Set();
  form.recipients.forEach(recipient => {
    if (recipient && typeof recipient.vars === 'object') {
      Object.keys(recipient.vars).forEach(key =>
        collected.add(normalizeVarKey(key))
      );
    }
  });
  form.customFields = Array.from(collected);

  const defaultPayload =
    form.channel === 'email'
      ? { subject: '', text: '', html: '' }
      : { message: 'Olá {{name}}!', messagesByDay: {} };
  const payload = JSON.parse(JSON.stringify(value.Payload || defaultPayload));
  if (form.channel === 'whatsapp' && !payload.messagesByDay) {
    payload.messagesByDay = {};
  }
  form.payload = payload;

  form.enabled = Boolean(value.Enabled);
  form.startAt = value.StartAt || '';
  form.endAt = value.EndAt || '';

  if (value.RunAt) {
    oneShot.value = true;
    form.runAt = value.RunAt;
    form.daysOfWeek = [];
    form.time = '';
    form.timezone = '';
  } else {
    oneShot.value = false;
    form.runAt = '';
    form.daysOfWeek = value.DaysOfWeek || ['MON'];
    form.time = value.Time || '08:00';
    form.timezone = value.Timezone || 'America/Sao_Paulo';
  }

  selectedTemplateKey.value = '';
  csvReport.value = null;
  status.show = false;
  status.msg = '';
  status.ok = true;

  nextTick(() => {
    recomputeRequiredPlaceholders();
    if (form.channel === 'whatsapp' && form.agent) {
      loadTemplates();
    } else {
      templates.value = [];
      selectedTemplateKey.value = '';
    }
  });
}

async function loadAgents() {
  agentsLoading.value = true;
  try {
    const { data } = await axios.get(AGENTS_API);
    agents.value = Array.isArray(data?.items)
      ? data.items.map(item => ({
          id: item.id,
          label: item.label,
          number: item.number,
        }))
      : [];
  } catch (error) {
    useAlert(text.alerts.agents);
  } finally {
    agentsLoading.value = false;
  }
}

async function loadTemplates() {
  if (form.channel !== 'whatsapp' || !form.agent) {
    templates.value = [];
    selectedTemplateKey.value = '';
    return;
  }
  templatesLoading.value = true;
  templates.value = [];
  selectedTemplateKey.value = '';
  try {
    const params = new URLSearchParams();
    params.set('agent', form.agent);
    const { data } = await axios.get(`${TEMPLATES_API}?${params.toString()}`);
    const items = Array.isArray(data?.items) ? data.items : [];
    templates.value = items.map((template, index) => ({
      key: `${template.name}::${template.language || template.locale || 'und'}::${index}`,
      name: template.name,
      title: `${template.name}${template.language ? ` (${template.language})` : ''}`,
      text: template.text || '',
      placeholders: Array.isArray(template.placeholders)
        ? template.placeholders.map(normalizeVarKey)
        : [],
    }));
  } catch (error) {
    useAlert(text.alerts.templates);
  } finally {
    templatesLoading.value = false;
  }
}

async function syncTemplates() {
  if (form.channel !== 'whatsapp' || !form.agent) return;
  templatesLoading.value = true;
  try {
    const params = new URLSearchParams();
    params.set('agent', form.agent);
    params.set('sync', '1');
    await axios.get(`${TEMPLATES_API}?${params.toString()}`);
    await loadTemplates();
    useAlert(text.alerts.syncSuccess);
  } catch (error) {
    useAlert(text.alerts.syncError);
  } finally {
    templatesLoading.value = false;
  }
}

function applyTemplate() {
  if (!selectedTemplate.value) return;
  form.payload.message = selectedTemplate.value.text || '';
  const merged = new Set(
    [
      ...form.customFields.map(normalizeVarKey),
      ...selectedTemplate.value.placeholders,
    ].filter(Boolean)
  );
  form.customFields = Array.from(merged);
  nextTick(recomputeRequiredPlaceholders);
}

function addCustomField() {
  const key = normalizeVarKey(newVariable.value);
  if (!key) return;
  if (!form.customFields.includes(key)) {
    form.customFields = [...form.customFields, key];
  }
  form.recipients = form.recipients.map(recipient => ({
    ...recipient,
    vars: {
      ...(recipient.vars || {}),
      [key]: recipient.vars?.[key] || '',
    },
  }));
  newVariable.value = '';
}

function addRecipient() {
  const vars = Object.fromEntries(
    form.customFields.map(field => [normalizeVarKey(field), ''])
  );
  form.recipients = [
    ...form.recipients,
    {
      name: '',
      phone: form.channel === 'whatsapp' ? '' : undefined,
      email: form.channel === 'email' ? '' : undefined,
      vars,
    },
  ];
  nextTick(recomputeRequiredPlaceholders);
}

function removeRecipient(index) {
  form.recipients = form.recipients.filter(
    (_, recipientIndex) => recipientIndex !== index
  );
  nextTick(recomputeRequiredPlaceholders);
}

function detectDelimiter(headerLine) {
  const commaCount = (headerLine.match(/,/g) || []).length;
  const semicolonCount = (headerLine.match(/;/g) || []).length;
  return semicolonCount > commaCount ? ';' : ',';
}

function parseCsvText(textValue) {
  const lines = textValue.split(/\r?\n/).filter(line => line.trim() !== '');
  if (!lines.length) return { headers: [], rows: [], originalHeaders: [] };
  const delimiter = detectDelimiter(lines[0]);
  const split = line => {
    const output = [];
    let buffer = '';
    let inQuotes = false;
    for (let index = 0; index < line.length; index += 1) {
      const char = line[index];
      if (char === '"') {
        if (line[index + 1] === '"') {
          buffer += '"';
          index += 1;
        } else {
          inQuotes = !inQuotes;
        }
      } else if (char === delimiter && !inQuotes) {
        output.push(buffer);
        buffer = '';
      } else {
        buffer += char;
      }
    }
    output.push(buffer);
    return output.map(cell => cell.trim());
  };

  const originalHeaders = split(lines[0]);
  const headers = originalHeaders.map(normalizeHeader);
  const rows = lines.slice(1).map(split);
  return { headers, rows, originalHeaders };
}

function handleCsvUpload(event) {
  const file = event.target.files?.[0];
  if (!file) return;
  const reader = new FileReader();
  reader.onload = () => {
    try {
      const textValue = reader.result?.toString() || '';
      const { headers, rows, originalHeaders } = parseCsvText(textValue);
      if (!headers.length || !rows.length) {
        csvReport.value = { ok: false, msg: text.recipients.csv.empty };
        return;
      }

      const headerIndex = new Map(
        headers.map((header, index) => [header, index])
      );
      const nameKey = ['name', 'nome'].find(key => headerIndex.has(key));
      const emailKey = ['email', 'e-mail', 'e_mail'].find(key =>
        headerIndex.has(key)
      );
      const phoneKey = ['phone', 'telefone', 'celular', 'whatsapp'].find(key =>
        headerIndex.has(key)
      );
      const needsEmail = form.channel === 'email';
      const hasRequiredHeaders = needsEmail
        ? nameKey && emailKey
        : nameKey && phoneKey;
      if (!hasRequiredHeaders) {
        csvReport.value = {
          ok: false,
          msg: needsEmail
            ? text.recipients.csv.invalidEmail
            : text.recipients.csv.invalidWhatsapp,
        };
        return;
      }

      const knownHeaders = new Set(
        [nameKey, emailKey, phoneKey].filter(Boolean)
      );
      const variableColumns = headers
        .map((header, index) => ({ header, index }))
        .filter(({ header }) => header && !knownHeaders.has(header));

      const newFieldNames = variableColumns.map(({ index, header }) =>
        normalizeVarKey(originalHeaders[index] || header)
      );
      const mergedFields = Array.from(
        new Set([...form.customFields.map(normalizeVarKey), ...newFieldNames])
      );
      form.customFields = mergedFields;

      const existingKeys = new Set(
        form.recipients
          .map(recipient =>
            needsEmail ? recipient.email?.toLowerCase() : recipient.phone
          )
          .filter(Boolean)
      );

      let imported = 0;
      let skipped = 0;
      const freshRecipients = [];

      rows.forEach(row => {
        const name = row[headerIndex.get(nameKey)]?.trim();
        const email = emailKey ? row[headerIndex.get(emailKey)]?.trim() : '';
        const phone = phoneKey ? row[headerIndex.get(phoneKey)]?.trim() : '';

        if (!name && !email && !phone) {
          skipped += 1;
          return;
        }

        const vars = Object.fromEntries(
          variableColumns.map(({ index, header }) => [
            normalizeVarKey(originalHeaders[index] || header),
            (row[index] || '').trim(),
          ])
        );

        if (needsEmail) {
          if (!email || !isValidEmail(email)) {
            skipped += 1;
            return;
          }
          const key = email.toLowerCase();
          if (existingKeys.has(key)) {
            skipped += 1;
            return;
          }
          existingKeys.add(key);
          freshRecipients.push({ name, email, vars });
          imported += 1;
        } else {
          if (!phone || !isLikelyPhone(phone)) {
            skipped += 1;
            return;
          }
          const normalizedPhone = phone.replace(/\s+/g, '');
          if (existingKeys.has(normalizedPhone)) {
            skipped += 1;
            return;
          }
          existingKeys.add(normalizedPhone);
          freshRecipients.push({ name, phone: normalizedPhone, vars });
          imported += 1;
        }
      });

      form.recipients = [...form.recipients, ...freshRecipients];
      csvReport.value = {
        ok: true,
        msg: text.recipients.csv.summary(
          imported,
          skipped,
          form.recipients.length
        ),
      };
      event.target.value = '';
      nextTick(recomputeRequiredPlaceholders);
    } catch (error) {
      csvReport.value = { ok: false, msg: text.recipients.csv.error };
    }
  };
  reader.readAsText(file, 'utf-8');
}

function downloadCsvTemplate() {
  const needsEmail = form.channel === 'email';
  const headers = needsEmail ? ['name', 'email'] : ['name', 'phone'];
  const rows = needsEmail
    ? [
        ['Ana', 'ana@exemplo.com'],
        ['Bruno', 'bruno@exemplo.com'],
      ]
    : [
        ['Ana', '+5532987654321'],
        ['Bruno', '+5531999999999'],
      ];
  const csv = [
    headers.join(','),
    ...rows.map(row =>
      row.map(value => `"${String(value).replace(/"/g, '""')}"`).join(',')
    ),
  ].join('\n');
  const blob = new Blob([csv], { type: 'text/csv;charset=utf-8;' });
  const link = document.createElement('a');
  link.href = URL.createObjectURL(blob);
  link.download = needsEmail
    ? 'recipients_email_template.csv'
    : 'recipients_whatsapp_template.csv';
  document.body.appendChild(link);
  link.click();
  document.body.removeChild(link);
  URL.revokeObjectURL(link.href);
}

function buildPayload() {
  const base = {
    name: form.name,
    channel: form.channel,
    agent: form.channel === 'whatsapp' ? form.agent || undefined : undefined,
    recipients: form.recipients,
    payload:
      form.channel === 'email'
        ? {
            subject: form.payload?.subject || text.content.subjectPlaceholder,
            text: form.payload?.text || '',
            html: form.payload?.html || '',
          }
        : {
            message: form.payload?.message || 'Olá {{name}}!',
            messagesByDay:
              !oneShot.value && form.channel === 'whatsapp'
                ? form.daysOfWeek.reduce((acc, day) => {
                    const value = (
                      form.payload?.messagesByDay?.[day] || ''
                    ).trim();
                    if (value) acc[day] = value;
                    return acc;
                  }, {})
                : undefined,
          },
    enabled: Boolean(form.enabled),
    startAt: form.startAt?.trim() || undefined,
    endAt: form.endAt?.trim() || undefined,
  };

  if (oneShot.value) {
    return {
      ...base,
      runAt: form.runAt,
    };
  }

  return {
    ...base,
    daysOfWeek: form.daysOfWeek,
    time: form.time,
    timezone: form.timezone || 'America/Sao_Paulo',
  };
}

async function submit() {
  if (!isValid.value || submitting.value) return;
  submitting.value = true;
  status.show = false;
  try {
    const payload = buildPayload();
    if (isEdit.value) {
      await api.put(`/${encodeURIComponent(form.name)}`, payload);
      status.show = true;
      status.ok = true;
      status.msg = text.status.updated;
    } else {
      await api.post('', payload);
      status.show = true;
      status.ok = true;
      status.msg = text.status.created;
    }
    emit('saved');
  } catch (error) {
    status.show = true;
    status.ok = false;
    status.msg = text.status.error;
  } finally {
    submitting.value = false;
  }
}

watch(
  () => props.value,
  newValue => {
    hydrateFromValue(newValue);
  },
  { immediate: true }
);

watch(
  () => form.channel,
  channel => {
    if (!channel) {
      form.agent = '';
      form.recipients = form.recipients.map(recipient => ({
        name: recipient.name || '',
        email: undefined,
        phone: undefined,
        vars: recipient.vars || {},
      }));
      form.payload = { message: 'Olá {{name}}!', messagesByDay: {} };
      templates.value = [];
      selectedTemplateKey.value = '';
      return;
    }
    form.recipients = form.recipients.map(recipient => ({
      name: recipient.name || '',
      email: channel === 'email' ? recipient.email || '' : undefined,
      phone: channel === 'whatsapp' ? recipient.phone || '' : undefined,
      vars: recipient.vars || {},
    }));

    if (channel === 'email') {
      form.payload = {
        subject: form.payload.subject || '',
        text: form.payload.text || '',
        html: form.payload.html || '',
      };
      templates.value = [];
      selectedTemplateKey.value = '';
    } else {
      form.payload = {
        message: form.payload.message || 'Olá {{name}}!',
        messagesByDay: form.payload.messagesByDay || {},
      };
      if (form.agent) {
        loadTemplates();
      } else {
        templates.value = [];
        selectedTemplateKey.value = '';
      }
    }

    nextTick(recomputeRequiredPlaceholders);
  }
);

watch(
  () => form.daysOfWeek.slice(),
  days => {
    if (form.channel !== 'whatsapp' || oneShot.value) return;
    const messagesByDay = { ...(form.payload.messagesByDay || {}) };
    days.forEach(day => {
      if (!Object.prototype.hasOwnProperty.call(messagesByDay, day)) {
        messagesByDay[day] = '';
      }
    });
    Object.keys(messagesByDay).forEach(day => {
      if (!days.includes(day)) {
        delete messagesByDay[day];
      }
    });
    form.payload.messagesByDay = messagesByDay;
    nextTick(recomputeRequiredPlaceholders);
  }
);

watch(
  () => form.customFields.map(normalizeVarKey),
  keys => {
    const normalizedKeys = keys.filter(Boolean);
    form.recipients = form.recipients.map(recipient => {
      const nextVars = { ...(recipient.vars || {}) };
      Object.keys(nextVars).forEach(key => {
        if (!normalizedKeys.includes(normalizeVarKey(key))) {
          delete nextVars[key];
        }
      });
      normalizedKeys.forEach(key => {
        if (!Object.prototype.hasOwnProperty.call(nextVars, key)) {
          nextVars[key] = '';
        }
      });
      return { ...recipient, vars: nextVars };
    });
  }
);

watch(
  () => form.agent,
  value => {
    if (form.channel !== 'whatsapp') return;
    if (value) {
      loadTemplates();
    } else {
      templates.value = [];
      selectedTemplateKey.value = '';
    }
  }
);

watch(
  () => oneShot.value,
  () => {
    if (oneShot.value) {
      form.daysOfWeek = [];
      form.time = '';
      form.timezone = '';
    } else {
      form.runAt = '';
      if (!form.daysOfWeek.length) form.daysOfWeek = ['MON'];
      if (!form.time) form.time = '08:00';
      if (!form.timezone) form.timezone = 'America/Sao_Paulo';
    }
    nextTick(recomputeRequiredPlaceholders);
  }
);

onMounted(() => {
  loadAgents();
  if (form.channel === 'whatsapp' && form.agent) loadTemplates();
  recomputeRequiredPlaceholders();
});
</script>

<template>
  <form
    class="flex max-h-[72vh] flex-col gap-8 overflow-y-auto pr-1"
    @submit.prevent="submit"
  >
    <section class="grid gap-6 md:grid-cols-2">
      <div class="flex flex-col gap-2">
        <label
          class="text-xs font-semibold uppercase tracking-wide text-n-slate-9"
        >
          {{ text.name.label }}
        </label>
        <input
          v-model.trim="form.name"
          :disabled="isEdit"
          class="rounded-xl border border-n-weak bg-n-background px-4 py-3 text-sm text-n-slate-12 focus:border-n-brand focus:outline-none"
          :placeholder="text.name.placeholder"
          required
        />
      </div>
      <div class="flex flex-col gap-2">
        <label
          class="text-xs font-semibold uppercase tracking-wide text-n-slate-9"
        >
          {{ text.channel.label }}
        </label>
        <select
          v-model="form.channel"
          class="rounded-xl px-4 py-3 text-sm focus:outline-none"
          :class="[
            canSelectChannel
              ? 'border border-n-weak bg-n-background text-n-slate-12 focus:border-n-brand'
              : 'border border-n-ruby-8 bg-n-solid-2 text-n-slate-9 cursor-not-allowed',
          ]"
          :disabled="!canSelectChannel"
          :title="!canSelectChannel ? text.tooltips.channel : ''"
        >
          <option disabled value="">
            {{ text.channel.placeholder }}
          </option>
          <option value="whatsapp">{{ text.channel.options.whatsapp }}</option>
          <option value="email">{{ text.channel.options.email }}</option>
        </select>
      </div>
    </section>

    <section v-if="form.channel === 'whatsapp'" class="space-y-3">
      <label
        class="text-xs font-semibold uppercase tracking-wide text-n-slate-9"
      >
        {{ text.agent.label }}
      </label>
      <select
        v-model="form.agent"
        class="w-full rounded-xl px-4 py-3 text-sm md:w-80 focus:outline-none"
        :class="[
          agentsLoading || !canSelectAgent
            ? 'border border-n-ruby-8 bg-n-solid-2 text-n-slate-9 cursor-not-allowed'
            : 'border border-n-weak bg-n-background text-n-slate-12 focus:border-n-brand',
        ]"
        :disabled="agentsLoading || !canSelectAgent"
        :title="!canSelectAgent ? text.tooltips.agent : ''"
      >
        <option value="" disabled>{{ text.agent.placeholder }}</option>
        <option
          v-for="whatsAgent in agents"
          :key="whatsAgent.id"
          :value="whatsAgent.number"
        >
          {{ whatsAgent.label }}
        </option>
      </select>
      <p class="text-xs text-n-slate-10">
        {{ text.agent.hint }}
      </p>
    </section>

    <section class="space-y-4">
      <header class="flex flex-wrap items-center justify-between gap-3">
        <div class="space-y-1">
          <h2 class="text-base font-semibold text-n-slate-12">
            {{ text.recipients.title }}
          </h2>
          <p class="text-xs text-n-slate-10">{{ text.recipients.subtitle }}</p>
        </div>
        <div class="flex flex-wrap items-center gap-2">
          <input
            v-model="newVariable"
            class="w-full rounded-xl border border-n-weak bg-n-background px-4 py-2 text-xs text-n-slate-12 focus:border-n-brand focus:outline-none md:w-48"
            :placeholder="text.customField.placeholder"
            @keyup.enter.prevent="addCustomField"
          />
          <Button
            variant="ghost"
            color="slate"
            size="sm"
            icon="i-lucide-list-plus"
            :label="text.customField.add"
            type="button"
            @click="addCustomField"
          />
          <div class="relative">
            <button
              type="button"
              class="rounded-xl border border-n-weak bg-n-background px-4 py-2 text-xs font-medium text-n-slate-11 shadow-sm hover:bg-n-solid-1"
            >
              {{ text.recipients.import }}
              <input
                type="file"
                accept=".csv,text/csv"
                class="absolute inset-0 h-full w-full cursor-pointer opacity-0"
                @change="handleCsvUpload"
              />
            </button>
          </div>
          <Button
            variant="ghost"
            color="slate"
            size="sm"
            icon="i-lucide-download"
            :label="text.recipients.download"
            type="button"
            @click="downloadCsvTemplate"
          />
          <Button
            color="blue"
            size="sm"
            icon="i-lucide-user-plus"
            :label="text.recipients.add"
            type="button"
            @click="addRecipient"
          />
        </div>
      </header>

      <div class="rounded-3xl border border-n-container bg-n-solid-2 p-4">
        <div class="grid gap-3">
          <div
            v-for="(recipient, index) in form.recipients"
            :key="`recipient-${index}`"
            class="flex flex-col gap-4 rounded-2xl border border-n-weak bg-n-solid-1 p-4"
          >
            <div class="grid gap-3 md:grid-cols-2">
              <div class="flex flex-col gap-2">
                <label
                  class="text-xs font-semibold uppercase tracking-wide text-n-slate-9"
                >
                  {{ text.recipients.name }}
                </label>
                <input
                  v-model="recipient.name"
                  class="rounded-xl border border-n-weak bg-transparent px-4 py-2 text-sm focus:border-n-brand focus:outline-none"
                  :placeholder="text.recipients.namePlaceholder"
                />
              </div>
              <div class="flex flex-col gap-2">
                <label
                  class="text-xs font-semibold uppercase tracking-wide text-n-slate-9"
                >
                  {{
                    form.channel === 'email'
                      ? text.recipients.email
                      : text.recipients.phone
                  }}
                </label>
                <input
                  v-if="form.channel === 'email'"
                  v-model="recipient.email"
                  class="rounded-xl border border-n-weak bg-transparent px-4 py-2 text-sm focus:border-n-brand focus:outline-none"
                  :placeholder="text.recipients.emailPlaceholder"
                  type="email"
                />
                <input
                  v-else
                  v-model="recipient.phone"
                  class="rounded-xl border border-n-weak bg-transparent px-4 py-2 text-sm focus:border-n-brand focus:outline-none"
                  :placeholder="text.recipients.phonePlaceholder"
                />
              </div>
            </div>

            <div
              v-if="form.customFields.length"
              class="grid gap-3 md:grid-cols-2"
            >
              <div
                v-for="field in form.customFields"
                :key="`${field}-${index}`"
                class="flex flex-col gap-2"
              >
                <label
                  class="text-xs font-semibold uppercase tracking-wide text-n-slate-9"
                >
                  {{ field }}
                </label>
                <input
                  v-model="recipient.vars[field]"
                  class="rounded-xl border border-n-weak bg-transparent px-4 py-2 text-sm focus:border-n-brand focus:outline-none"
                  :placeholder="field"
                />
              </div>
            </div>

            <div class="flex justify-between">
              <Button
                variant="ghost"
                color="ruby"
                size="xs"
                icon="i-lucide-trash"
                :label="text.recipients.remove"
                type="button"
                @click="removeRecipient(index)"
              />
            </div>
          </div>
        </div>

        <p v-if="!form.recipients.length" class="mt-4 text-sm text-n-slate-10">
          {{ text.recipients.empty }}
        </p>
        <p
          v-if="csvReport"
          class="mt-3 text-sm"
          :class="csvReport.ok ? 'text-n-teal-11' : 'text-n-ruby-11'"
        >
          {{ csvReport.msg }}
        </p>
      </div>
    </section>

    <section class="space-y-6">
      <header class="space-y-1">
        <h2 class="text-base font-semibold text-n-slate-12">
          {{ text.content.title }}
        </h2>
        <p class="text-xs text-n-slate-10">{{ text.content.description }}</p>
      </header>

      <div v-if="form.channel === 'email'" class="grid gap-4">
        <div class="flex flex-col gap-2">
          <label
            class="text-xs font-semibold uppercase tracking-wide text-n-slate-9"
          >
            {{ text.content.subject }}
          </label>
          <input
            v-model="form.payload.subject"
            class="rounded-xl border border-n-weak bg-n-background px-4 py-3 text-sm text-n-slate-12 focus:border-n-brand focus:outline-none"
            :placeholder="text.content.subjectPlaceholder"
          />
        </div>
        <div class="flex flex-col gap-2">
          <label
            class="text-xs font-semibold uppercase tracking-wide text-n-slate-9"
          >
            {{ text.content.text }}
          </label>
          <textarea
            v-model="form.payload.text"
            rows="4"
            class="rounded-xl border border-n-weak bg-n-background px-4 py-3 text-sm text-n-slate-12 focus:border-n-brand focus:outline-none"
            :placeholder="text.content.textPlaceholder"
            @input="recomputeRequiredPlaceholders"
          />
        </div>
        <div class="flex flex-col gap-2">
          <label
            class="text-xs font-semibold uppercase tracking-wide text-n-slate-9"
          >
            {{ text.content.html }}
          </label>
          <textarea
            v-model="form.payload.html"
            rows="4"
            class="rounded-xl border border-n-weak bg-n-background px-4 py-3 text-sm text-n-slate-12 focus:border-n-brand focus:outline-none"
            :placeholder="text.content.htmlPlaceholder"
            @input="recomputeRequiredPlaceholders"
          />
        </div>
      </div>

      <div v-else class="space-y-4">
        <div class="flex flex-wrap items-center gap-2">
          <select
            v-model="selectedTemplateKey"
            class="w-full rounded-xl border border-n-weak bg-n-background px-4 py-3 text-sm text-n-slate-12 focus:border-n-brand focus:outline-none md:w-60"
            :disabled="templatesLoading || !templates.length"
          >
            <option value="" disabled>
              {{
                templatesLoading
                  ? text.content.templateSelect.loading
                  : templates.length
                    ? text.content.templateSelect.placeholder
                    : text.content.templateSelect.empty
              }}
            </option>
            <option
              v-for="template in templates"
              :key="template.key"
              :value="template.key"
            >
              {{ template.title }}
            </option>
          </select>
          <Button
            variant="ghost"
            color="slate"
            size="sm"
            icon="i-lucide-rotate-cw"
            :label="text.content.templateActions.reload"
            type="button"
            :disabled="templatesLoading"
            @click="loadTemplates"
          />
          <Button
            variant="ghost"
            color="slate"
            size="sm"
            icon="i-lucide-refresh-cw"
            :label="text.content.templateActions.sync"
            type="button"
            :disabled="templatesLoading"
            @click="syncTemplates"
          />
          <Button
            color="blue"
            size="sm"
            icon="i-lucide-sparkles"
            :label="text.content.templateActions.apply"
            type="button"
            :disabled="!selectedTemplate"
            @click="applyTemplate"
          />
        </div>

        <p
          v-if="selectedTemplate?.placeholders.length"
          class="text-xs text-n-slate-10"
        >
          {{ text.content.templatePlaceholders }}
          <span
            v-for="placeholder in selectedTemplate.placeholders"
            :key="placeholder"
            class="ml-1 inline-flex items-center rounded-full bg-n-solid-2 px-2 py-0.5 text-[11px]"
          >
            {{ placeholder }}
          </span>
        </p>

        <div class="flex flex-col gap-2">
          <label
            class="text-xs font-semibold uppercase tracking-wide text-n-slate-9"
          >
            {{ text.content.message }}
          </label>
          <textarea
            v-model="form.payload.message"
            rows="4"
            class="rounded-xl border border-n-weak bg-n-background px-4 py-3 text-sm text-n-slate-12 focus:border-n-brand focus:outline-none"
            :placeholder="text.content.textPlaceholder"
            @input="recomputeRequiredPlaceholders"
          />
        </div>

        <div
          v-if="placeholderGlobalError"
          class="rounded-xl bg-n-ruby-2 px-4 py-3 text-sm text-n-ruby-11"
        >
          {{ placeholderGlobalError }}
        </div>
      </div>
    </section>

    <section class="space-y-6">
      <header class="space-y-1">
        <h2 class="text-base font-semibold text-n-slate-12">
          {{ text.schedule.title }}
        </h2>
        <p class="text-xs text-n-slate-10">{{ text.schedule.description }}</p>
      </header>

      <div class="flex items-center gap-3">
        <input
          id="oneshot"
          v-model="oneShot"
          type="checkbox"
          class="h-4 w-4 rounded border border-n-weak text-n-brand focus:ring-n-brand"
        />
        <label for="oneshot" class="text-sm text-n-slate-12">{{
          text.schedule.runOnce
        }}</label>
      </div>

      <div v-if="oneShot" class="grid gap-4 md:grid-cols-2">
        <div class="flex flex-col gap-2">
          <label
            class="text-xs font-semibold uppercase tracking-wide text-n-slate-9"
          >
            {{ text.schedule.runAt }}
          </label>
          <input
            v-model="form.runAt"
            class="rounded-xl border border-n-weak bg-n-background px-4 py-3 text-sm text-n-slate-12 focus:border-n-brand focus:outline-none"
            :placeholder="text.schedule.runAtPlaceholder"
          />
        </div>
        <div class="flex flex-col gap-2">
          <label
            class="text-xs font-semibold uppercase tracking-wide text-n-slate-9"
          >
            {{ text.schedule.timezone }}
          </label>
          <input
            :value="text.schedule.timezoneStatic"
            disabled
            class="rounded-xl border border-dashed border-n-weak bg-n-solid-2 px-4 py-3 text-sm text-n-slate-10"
          />
        </div>
      </div>

      <div v-else class="grid gap-4 md:grid-cols-3">
        <div class="flex flex-col gap-3">
          <span
            class="text-xs font-semibold uppercase tracking-wide text-n-slate-9"
          >
            {{ text.schedule.days }}
          </span>
          <div class="grid grid-cols-2 gap-2">
            <label
              v-for="day in DAYS_OF_WEEK"
              :key="day.value"
              class="inline-flex items-center gap-2 rounded-xl border border-n-weak bg-n-solid-1 px-3 py-2 text-xs font-medium text-n-slate-11"
            >
              <input
                v-model="form.daysOfWeek"
                type="checkbox"
                :value="day.value"
                class="h-4 w-4 rounded border border-n-weak text-n-brand focus:ring-n-brand"
              />
              {{ day.label }}
            </label>
          </div>
        </div>
        <div class="flex flex-col gap-2">
          <label
            class="text-xs font-semibold uppercase tracking-wide text-n-slate-9"
          >
            {{ text.schedule.time }}
          </label>
          <input
            v-model="form.time"
            class="rounded-xl border border-n-weak bg-n-background px-4 py-3 text-sm text-n-slate-12 focus:border-n-brand focus:outline-none"
            :placeholder="text.schedule.timePlaceholder"
          />
        </div>
        <div class="flex flex-col gap-2">
          <label
            class="text-xs font-semibold uppercase tracking-wide text-n-slate-9"
          >
            {{ text.schedule.timezone }}
          </label>
          <input
            v-model="form.timezone"
            class="rounded-xl border border-n-weak bg-n-background px-4 py-3 text-sm text-n-slate-12 focus:border-n-brand focus:outline-none"
            :placeholder="text.schedule.timezonePlaceholder"
          />
        </div>
      </div>

      <div
        v-if="form.channel === 'whatsapp' && !oneShot"
        class="space-y-3 rounded-3xl border border-n-container bg-n-solid-2 p-4"
      >
        <h3 class="text-sm font-semibold text-n-slate-12">
          {{ text.schedule.dayMessagesTitle }}
        </h3>
        <div class="grid gap-4 md:grid-cols-2">
          <div
            v-for="day in form.daysOfWeek"
            :key="day"
            class="flex flex-col gap-2 rounded-2xl border border-n-weak bg-n-solid-1 p-3"
          >
            <label
              class="text-xs font-semibold uppercase tracking-wide text-n-slate-9"
            >
              {{
                DAYS_OF_WEEK.find(option => option.value === day)?.label || day
              }}
            </label>
            <textarea
              v-model="form.payload.messagesByDay[day]"
              rows="3"
              class="rounded-xl border border-n-weak bg-transparent px-4 py-3 text-sm text-n-slate-12 focus:border-n-brand focus:outline-none"
              :placeholder="
                text.schedule.messageLabel(
                  DAYS_OF_WEEK.find(option => option.value === day)?.label ||
                    day
                )
              "
              @input="recomputeRequiredPlaceholders"
            />
            <p
              v-if="!(form.payload.messagesByDay[day] || '').trim()"
              class="text-xs text-n-ruby-11"
            >
              {{
                text.schedule.missingMessage(
                  DAYS_OF_WEEK.find(option => option.value === day)?.label ||
                    day
                )
              }}
            </p>
          </div>
        </div>
        <p
          v-if="placeholderDaysError"
          class="rounded-xl bg-n-ruby-2 px-3 py-2 text-xs text-n-ruby-11"
        >
          {{ placeholderDaysError }}
        </p>
      </div>
    </section>

    <section class="grid gap-4 md:grid-cols-2">
      <div class="flex flex-col gap-2">
        <label
          class="text-xs font-semibold uppercase tracking-wide text-n-slate-9"
        >
          {{ text.timeframe.start }}
        </label>
        <input
          v-model="form.startAt"
          class="rounded-xl border border-n-weak bg-n-background px-4 py-3 text-sm text-n-slate-12 focus:border-n-brand focus:outline-none"
          :placeholder="text.timeframe.startPlaceholder"
        />
      </div>
      <div class="flex flex-col gap-2">
        <label
          class="text-xs font-semibold uppercase tracking-wide text-n-slate-9"
        >
          {{ text.timeframe.end }}
        </label>
        <input
          v-model="form.endAt"
          class="rounded-xl border border-n-weak bg-n-background px-4 py-3 text-sm text-n-slate-12 focus:border-n-brand focus:outline-none"
          :placeholder="text.timeframe.endPlaceholder"
        />
      </div>
      <p
        v-if="!validDateRange"
        class="md:col-span-2 rounded-xl bg-n-ruby-2 px-4 py-2 text-sm text-n-ruby-11"
      >
        {{ text.timeframe.error }}
      </p>
    </section>

    <label
      class="flex items-center gap-3 rounded-2xl border border-n-weak bg-n-solid-2 px-4 py-3 text-sm"
    >
      <input
        v-model="form.enabled"
        type="checkbox"
        class="h-4 w-4 rounded border border-n-weak text-n-brand focus:ring-n-brand"
      />
      {{ text.toggle }}
    </label>

    <div class="flex flex-col gap-3">
      <Button
        color="blue"
        size="md"
        class="w-full"
        :is-loading="submitting"
        :disabled="submitting || !isValid"
        :label="text.actions.save"
        type="submit"
      />
      <Button
        variant="ghost"
        color="slate"
        size="sm"
        :label="text.actions.cancel"
        type="button"
        @click="emit('close')"
      />
    </div>

    <div
      v-if="status.show"
      class="rounded-2xl border px-4 py-3"
      :class="
        status.ok
          ? 'border-n-teal-8 bg-n-teal-2 text-n-teal-11'
          : 'border-n-ruby-8 bg-n-ruby-2 text-n-ruby-11'
      "
    >
      {{ status.msg }}
    </div>
  </form>
</template>
