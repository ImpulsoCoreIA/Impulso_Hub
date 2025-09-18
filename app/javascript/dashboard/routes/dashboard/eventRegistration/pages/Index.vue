<script setup>
import {
  computed,
  nextTick,
  onBeforeUnmount,
  onMounted,
  reactive,
  ref,
  watch,
} from 'vue';
import axios from 'axios';

import { useAlert } from 'dashboard/composables';
import Button from 'dashboard/components-next/button/Button.vue';
import Dialog from 'dashboard/components-next/dialog/Dialog.vue';

import EventForm from '../components/EventForm.vue';

const API_BASE =
  'https://f4wzfjousg.execute-api.us-east-1.amazonaws.com/schedules';
const LOCAL_TIMEZONE = 'America/Sao_Paulo';
const WEEKDAY_LABEL = {
  SUN: 'Dom',
  MON: 'Seg',
  TUE: 'Ter',
  WED: 'Qua',
  THU: 'Qui',
  FRI: 'Sex',
  SAT: 'Sáb',
};

const text = Object.freeze({
  fallbackDash: '—',
  header: {
    title: 'Agendamentos',
    description:
      'Gerencie disparos recorrentes por WhatsApp ou Email. Crie, organize e acompanhe o status das suas automações de envio em um só lugar.',
  },
  stats: {
    total: 'Total',
    enabled: 'Ativos',
    disabled: 'Inativos',
  },
  filters: {
    searchPlaceholder: 'Buscar por nome, canal ou expressão',
    channels: {
      all: 'Todos',
      whatsapp: 'WhatsApp',
      email: 'Email',
    },
    statuses: {
      all: 'Todos',
      enabled: 'Ativos',
      disabled: 'Inativos',
    },
  },
  empty: {
    icon: '⏱️',
    title: 'Nenhum agendamento encontrado',
    description:
      'Crie o primeiro agendamento ou ajuste os filtros de busca para visualizar registros existentes.',
    action: 'Criar agendamento',
  },
  buttons: {
    refresh: 'Recarregar',
    create: 'Novo agendamento',
    createCompact: 'Criar agendamento',
    preview: 'Prévia',
    removeRecipient: 'Remover',
    toggleRecipientsShow: 'Ver destinatários',
    toggleRecipientsHide: 'Ocultar destinatários',
    toggleEnable: 'Ativar agendamento',
    toggleDisable: 'Desativar agendamento',
    editSchedule: 'Editar agendamento',
    deleteSchedule: 'Excluir agendamento',
  },
  card: {
    statusEnabled: 'Ativo',
    statusDisabled: 'Inativo',
    channelEmail: 'Email',
    channelWhatsapp: 'WhatsApp',
    labels: {
      model: 'Tipo:',
      when: 'Frequência:',
      timeframe: 'Vigência:',
      recipients: 'Destinatários:',
    },
    types: {
      runAt: 'Pontual',
      recurring: 'Recorrente',
    },
  },
  preview: {
    title: 'Prévia do envio',
    fallbackSubject: 'Mensagem',
    empty: '(sem conteúdo)',
    close: 'Fechar',
  },
  delete: {
    title: 'Remover agendamento',
    description:
      'Esta ação não pode ser desfeita. O agendamento será removido definitivamente.',
    confirm: 'Excluir',
    cancel: 'Cancelar',
    question: name =>
      name ? `Confirma a exclusão de ${name}?` : 'Confirma a exclusão?',
  },
  pagination: {
    previous: 'Anterior',
    next: 'Próxima',
    info: (current, total) => `Página ${current} de ${total}`,
  },
  alerts: {
    loadError: 'Falha ao carregar agendamentos. Tente novamente.',
    toggleError: 'Não foi possível atualizar o status do agendamento.',
    deleteSuccess: 'Agendamento removido com sucesso.',
    deleteError: 'Falha ao remover agendamento. Tente novamente.',
  },
});

const api = axios.create({ baseURL: API_BASE });

const schedules = ref([]);
const loading = ref(false);
const page = ref(1);
const pageSize = 6;
const filters = reactive({
  search: '',
  channel: 'all',
  status: 'all',
});

const expandedIds = ref(new Set());
const togglingId = ref('');
const deletingId = ref('');
const deleteTarget = ref(null);
const selectedSchedule = ref(null);
const previewContent = reactive({ title: '', body: '' });

const formDialogRef = ref(null);
const previewDialogRef = ref(null);
const deleteDialogRef = ref(null);

const formDialogOpen = ref(false);
const previewDialogOpen = ref(false);
const deleteDialogOpen = ref(false);

const channelOptions = [
  { value: 'all', label: text.filters.channels.all, icon: 'i-lucide-layers' },
  {
    value: 'whatsapp',
    label: text.filters.channels.whatsapp,
    icon: 'i-lucide-phone',
  },
  { value: 'email', label: text.filters.channels.email, icon: 'i-lucide-mail' },
];

const statusOptions = [
  {
    value: 'all',
    label: text.filters.statuses.all,
    badge: 'bg-n-alpha-3 text-n-slate-11',
  },
  {
    value: 'enabled',
    label: text.filters.statuses.enabled,
    badge: 'bg-n-teal-3 text-n-teal-11',
  },
  {
    value: 'disabled',
    label: text.filters.statuses.disabled,
    badge: 'bg-n-slate-3 text-n-slate-11',
  },
];

function formatDateTime(value) {
  if (!value) return text.fallbackDash;
  try {
    const iso =
      typeof value === 'string' && value.includes(' ')
        ? value.replace(' ', 'T')
        : value;
    const date = new Date(iso);
    if (Number.isNaN(date.getTime())) return value;
    return date.toLocaleString('pt-BR', {
      timeZone: LOCAL_TIMEZONE,
      year: 'numeric',
      month: '2-digit',
      day: '2-digit',
      hour: '2-digit',
      minute: '2-digit',
    });
  } catch (error) {
    return value;
  }
}

function humanRecurring(item) {
  // Para agendamentos pontuais
  if (item.runAt) {
    return formatDateTime(item.runAt);
  }
  
  // Para agendamentos recorrentes
  const days = (item.daysOfWeek || []);
  const time = item.time || text.fallbackDash;
  
  if (!days.length) return text.fallbackDash;
  
  // Detectar padrões comuns
  const daysCount = days.length;
  const isWeekdays = daysCount === 5 && 
    ['MON', 'TUE', 'WED', 'THU', 'FRI'].every(d => days.includes(d));
  const isWeekend = daysCount === 2 && 
    ['SAT', 'SUN'].every(d => days.includes(d));
  const isDaily = daysCount === 7;
  
  if (isDaily) {
    return `Todos os dias às ${time}`;
  } else if (isWeekdays) {
    return `Dias úteis às ${time}`;
  } else if (isWeekend) {
    return `Finais de semana às ${time}`;
  } else if (daysCount === 1) {
    const dayLabel = WEEKDAY_LABEL[days[0]] || days[0];
    return `Toda ${dayLabel} às ${time}`;
  } else {
    const daysLabel = days
      .map(day => WEEKDAY_LABEL[day] || day)
      .join(', ');
    return `${daysLabel} às ${time}`;
  }
}

function renderTemplate(template, variables) {
  if (!template) return '';
  const normalized = { ...(variables || {}) };
  if (normalized.name && !normalized.nome) normalized.nome = normalized.name;
  if (normalized.nome && !normalized.name) normalized.name = normalized.nome;
  return template.replace(
    /\{\{\s*([a-zA-Z0-9_\-.]+)\s*\}\}/g,
    (_, key) => normalized[key] ?? ''
  );
}

function dayKeyFor(timezone) {
  try {
    const formatter = new Intl.DateTimeFormat('en-US', {
      timeZone: timezone || 'UTC',
      weekday: 'short',
    });
    return formatter.format(new Date()).toUpperCase().slice(0, 3);
  } catch (error) {
    return 'MON';
  }
}

function mapApiResponse(rows) {
  return rows.map(row => ({
    eventId: row.Name,
    channel: row.Channel,
    recipients: row.Recipients || [],
    payload: row.Payload || {},
    enabled: Boolean(row.Enabled),
    scheduleExpression: row.ScheduleExpression || '',
    timezone: row.Timezone || null,
    daysOfWeek: row.DaysOfWeek || [],
    time: row.Time || null,
    runAt: row.RunAt || null,
    startAt: row.StartAt || null,
    endAt: row.EndAt || null,
    agent: row.Agent || null,
  }));
}

function prepareEditPayload(schedule) {
  return {
    Name: schedule.eventId,
    Channel: schedule.channel,
    Recipients: JSON.parse(JSON.stringify(schedule.recipients || [])),
    Payload: JSON.parse(JSON.stringify(schedule.payload || {})),
    Enabled: schedule.enabled,
    ScheduleExpression: schedule.scheduleExpression,
    Timezone: schedule.timezone,
    DaysOfWeek: schedule.daysOfWeek,
    Time: schedule.time,
    RunAt: schedule.runAt,
    StartAt: schedule.startAt,
    EndAt: schedule.endAt,
    Agent: schedule.agent || null,
  };
}

function isExpanded(id) {
  return expandedIds.value.has(id);
}

function toggleRecipients(id) {
  const next = new Set(expandedIds.value);
  if (next.has(id)) {
    next.delete(id);
  } else {
    next.add(id);
  }
  expandedIds.value = next;
}

async function loadSchedules() {
  loading.value = true;
  try {
    const { data } = await api.get('');
    let items = [];
    if (Array.isArray(data?.items)) {
      items = data.items;
    } else if (Array.isArray(data)) {
      items = data;
    }
    schedules.value = mapApiResponse(items);
  } catch (error) {
    useAlert(text.alerts.loadError);
    schedules.value = [];
  } finally {
    loading.value = false;
  }
}

function openForm(schedule) {
  selectedSchedule.value = schedule ? prepareEditPayload(schedule) : null;
  formDialogOpen.value = true;
  nextTick(() => {
    formDialogRef.value?.open();
  });
}

function resetFormDialogState() {
  selectedSchedule.value = null;
  formDialogOpen.value = false;
}

function hideFormDialog() {
  if (!formDialogOpen.value) {
    resetFormDialogState();
    return;
  }
  formDialogRef.value?.close();
  resetFormDialogState();
}

async function handleSaved() {
  hideFormDialog();
  await loadSchedules();
}

function openPreview(schedule, recipient) {
  if (!schedule) return;
  const channel = schedule.channel;
  const variables = {
    name: recipient.name || '',
    email: recipient.email || '',
    phone: recipient.phone || '',
    ...(recipient.vars || {}),
  };

  if (channel === 'email') {
    const subject = renderTemplate(
      schedule.payload?.subject || text.preview.fallbackSubject,
      variables
    );
    const body = renderTemplate(
      schedule.payload?.text || schedule.payload?.html || '',
      variables
    );
    previewContent.title = subject || schedule.eventId;
    previewContent.body = body || text.preview.empty;
  } else {
    const dayKey = dayKeyFor(schedule.timezone);
    const byDay = schedule.payload?.messagesByDay || {};
    const fallback = schedule.payload?.message || '';
    const message = renderTemplate(byDay[dayKey] || fallback, variables);
    previewContent.title = schedule.eventId;
    previewContent.body = message || text.preview.empty;
  }

  previewDialogOpen.value = true;
  nextTick(() => {
    previewDialogRef.value?.open();
  });
}

function resetPreviewDialogState() {
  previewContent.title = '';
  previewContent.body = '';
  previewDialogOpen.value = false;
}

function hidePreviewDialog() {
  if (!previewDialogOpen.value) {
    resetPreviewDialogState();
    return;
  }
  previewDialogRef.value?.close();
  resetPreviewDialogState();
}

function resetDeleteDialogState() {
  deleteTarget.value = null;
  deleteDialogOpen.value = false;
}

function openDeleteDialog(schedule) {
  deleteTarget.value = schedule;
  deleteDialogOpen.value = true;
  nextTick(() => {
    deleteDialogRef.value?.open();
  });
}

function hideDeleteDialog() {
  if (!deleteDialogOpen.value) {
    resetDeleteDialogState();
    return;
  }
  deleteDialogRef.value?.close();
  resetDeleteDialogState();
}

async function toggleEnabled(schedule) {
  if (!schedule?.eventId || togglingId.value) return;
  togglingId.value = schedule.eventId;
  try {
    await api.patch(`/${encodeURIComponent(schedule.eventId)}`, {
      enabled: !schedule.enabled,
    });
    schedule.enabled = !schedule.enabled;
  } catch (error) {
    useAlert(text.alerts.toggleError);
  } finally {
    togglingId.value = '';
  }
}

async function confirmDelete() {
  const target = deleteTarget.value;
  if (!target?.eventId) return;
  deletingId.value = target.eventId;
  try {
    await api.delete(`/${encodeURIComponent(target.eventId)}`);
    schedules.value = schedules.value.filter(
      item => item.eventId !== target.eventId
    );
    useAlert(text.alerts.deleteSuccess);
    hideDeleteDialog();
  } catch (error) {
    useAlert(text.alerts.deleteError);
  } finally {
    deletingId.value = '';
  }
}

function handleKeydown(event) {
  if (event.key !== 'Escape') return;
  if (formDialogOpen.value) hideFormDialog();
  if (previewDialogOpen.value) hidePreviewDialog();
  if (deleteDialogOpen.value) hideDeleteDialog();
}

const stats = computed(() => {
  const total = schedules.value.length;
  const enabled = schedules.value.filter(item => item.enabled).length;
  const disabled = total - enabled;
  return [
    { label: text.stats.total, value: total, icon: 'i-lucide-calendar-clock' },
    { label: text.stats.enabled, value: enabled, icon: 'i-lucide-play-circle' },
    {
      label: text.stats.disabled,
      value: disabled,
      icon: 'i-lucide-pause-circle',
    },
  ];
});

const filteredSchedules = computed(() => {
  const query = filters.search.trim().toLowerCase();
  return schedules.value
    .filter(item => {
      if (filters.channel !== 'all') {
        return (item.channel || '').toLowerCase() === filters.channel;
      }
      return true;
    })
    .filter(item => {
      if (filters.status === 'all') return true;
      const wantEnabled = filters.status === 'enabled';
      return Boolean(item.enabled) === wantEnabled;
    })
    .filter(item => {
      if (!query) return true;
      return [item.eventId, item.channel, item.scheduleExpression]
        .map(value => (value || '').toString().toLowerCase())
        .some(value => value.includes(query));
    });
});

const totalPages = computed(() =>
  Math.max(1, Math.ceil(filteredSchedules.value.length / pageSize))
);

const pagedSchedules = computed(() => {
  const start = (page.value - 1) * pageSize;
  return filteredSchedules.value.slice(start, start + pageSize);
});

const goToPreviousPage = () => {
  if (page.value <= 1) return;
  page.value -= 1;
};

const goToNextPage = () => {
  if (page.value >= totalPages.value) return;
  page.value += 1;
};

watch(
  () => filters.search,
  () => {
    page.value = 1;
  }
);

watch(
  () => filters.channel,
  () => {
    page.value = 1;
  }
);

watch(
  () => filters.status,
  () => {
    page.value = 1;
  }
);

watch(filteredSchedules, newList => {
  const maxPage = Math.max(1, Math.ceil(newList.length / pageSize));
  if (page.value > maxPage) page.value = maxPage;
});

onMounted(async () => {
  await loadSchedules();
  window.addEventListener('keydown', handleKeydown);
});

onBeforeUnmount(() => {
  window.removeEventListener('keydown', handleKeydown);
});
</script>

<template>
  <div
    class="mx-auto flex min-h-screen w-full max-w-6xl flex-col gap-8 px-6 py-10 text-n-slate-12"
  >
    <header class="flex flex-wrap items-start justify-between gap-6">
      <div class="flex flex-col gap-2">
        <h1 class="text-2xl font-semibold">
          {{ text.header.title }}
        </h1>
        <p class="max-w-2xl text-sm text-n-slate-10">
          {{ text.header.description }}
        </p>
      </div>
      <div class="flex flex-wrap items-center gap-3">
        <Button
          variant="faded"
          color="slate"
          size="sm"
          icon="i-lucide-rotate-cw"
          :label="text.buttons.refresh"
          :is-loading="loading"
          :disabled="loading"
          @click="loadSchedules"
        />
        <Button
          color="blue"
          size="sm"
          icon="i-lucide-plus"
          :label="text.buttons.create"
          @click="openForm()"
        />
      </div>
    </header>

    <section class="grid gap-4 md:grid-cols-3">
      <div
        v-for="card in stats"
        :key="card.label"
        class="flex items-center gap-4 rounded-2xl border border-n-container bg-n-solid-1 px-5 py-4 shadow-sm"
      >
        <span
          class="flex h-12 w-12 items-center justify-center rounded-xl bg-n-solid-2 text-xl text-n-blue-text"
        >
          <i :class="card.icon" />
        </span>
        <div class="flex flex-col">
          <span class="text-xs uppercase tracking-wide text-n-slate-9">
            {{ card.label }}
          </span>
          <span class="text-2xl font-semibold text-n-slate-12 tabular-nums">
            {{ card.value }}
          </span>
        </div>
      </div>
    </section>

    <section
      class="rounded-3xl border border-n-container bg-n-solid-1 p-6 shadow-xl shadow-n-solid-3/10"
    >
      <div
        class="flex flex-col gap-4 lg:flex-row lg:items-center lg:justify-between"
      >
        <div class="flex w-full items-center gap-3 rounded-full border border-transparent bg-n-solid-2 px-4 py-1.5 lg:max-w-md">
          <i class="i-lucide-search text-sm text-n-slate-10 flex-shrink-0" />
          <input
            v-model="filters.search"
            type="search"
            class="w-full bg-transparent text-xs font-medium text-n-slate-12 placeholder:text-n-slate-10 focus:outline-none"
            :placeholder="text.filters.searchPlaceholder"
          />
        </div>

        <div class="flex w-full flex-wrap items-center gap-3 lg:justify-end">
          <div
            class="flex flex-wrap items-center gap-2 rounded-full border border-n-container bg-n-solid-2 px-1 py-1"
          >
            <button
              v-for="option in channelOptions"
              :key="option.value"
              type="button"
              class="flex items-center gap-2 rounded-full px-3 py-1.5 text-xs font-medium transition"
              :class="[
                filters.channel === option.value
                  ? 'bg-n-solid-3 text-n-slate-12 shadow-sm'
                  : 'text-n-slate-10 hover:bg-n-solid-3',
              ]"
              @click="filters.channel = option.value"
            >
              <i :class="option.icon" class="text-sm" />
              {{ option.label }}
            </button>
          </div>

          <div
            class="flex flex-wrap items-center gap-2 rounded-full border border-n-container bg-n-solid-2 px-1 py-1"
          >
            <button
              v-for="option in statusOptions"
              :key="option.value"
              type="button"
              class="flex items-center gap-2 rounded-full px-3 py-1.5 text-xs font-medium transition"
              :class="[
                filters.status === option.value
                  ? 'bg-n-solid-3 text-n-slate-12 shadow-sm'
                  : 'text-n-slate-10 hover:bg-n-solid-3',
              ]"
              @click="filters.status = option.value"
            >
              <span
                class="inline-flex h-2 w-2 rounded-full"
                :class="option.badge"
              />
              {{ option.label }}
            </button>
          </div>
        </div>
      </div>

      <div class="mt-6 space-y-6">
        <div v-if="loading" class="grid gap-4 sm:grid-cols-2 xl:grid-cols-3">
          <div
            v-for="skeleton in 6"
            :key="skeleton"
            class="h-48 rounded-2xl border border-n-container bg-n-solid-2/60 animate-pulse"
          />
        </div>

        <div
          v-else-if="!pagedSchedules.length"
          class="flex flex-col items-center justify-center gap-3 rounded-3xl border border-dashed border-n-container bg-n-solid-2 px-12 py-16 text-center"
        >
          <span class="text-4xl">{{ text.empty.icon }}</span>
          <h2 class="text-xl font-semibold text-n-slate-12">
            {{ text.empty.title }}
          </h2>
          <p class="max-w-md text-sm text-n-slate-10">
            {{ text.empty.description }}
          </p>
          <Button
            color="blue"
            icon="i-lucide-plus"
            :label="text.empty.action"
            @click="openForm()"
          />
        </div>

        <div v-else class="grid gap-4 sm:grid-cols-2 xl:grid-cols-3">
          <article
            v-for="item in pagedSchedules"
            :key="item.eventId"
            class="flex h-full flex-col gap-4 rounded-3xl border border-n-container bg-n-solid-2 p-5 shadow-lg shadow-n-solid-2/20 transition hover:-translate-y-0.5 hover:shadow-xl"
            :class="{ 'opacity-70': !item.enabled }"
          >
            <header class="flex items-start justify-between gap-3">
              <div class="min-w-0 space-y-2">
                <div class="flex items-center gap-2">
                  <h3
                    class="truncate text-base font-semibold"
                    :title="item.eventId"
                  >
                    {{ item.eventId }}
                  </h3>
                  <span
                    class="inline-flex items-center gap-1 rounded-full px-2 py-0.5 text-[11px] font-semibold"
                    :class="
                      item.enabled
                        ? 'bg-n-teal-2 text-n-teal-11'
                        : 'bg-n-slate-3 text-n-slate-11'
                    "
                  >
                    {{
                      item.enabled
                        ? text.card.statusEnabled
                        : text.card.statusDisabled
                    }}
                  </span>
                  <span
                    class="rounded-full border border-n-weak px-2 py-0.5 text-[11px] text-n-slate-11"
                  >
                    {{
                      item.channel === 'email'
                        ? text.card.channelEmail
                        : text.card.channelWhatsapp
                    }}
                  </span>
                </div>
                <dl class="space-y-2 text-xs text-n-slate-10">
                  <div class="flex items-center gap-2">
                    <dt class="font-medium text-n-slate-9">
                      {{ text.card.labels.model }}
                    </dt>
                    <dd class="text-n-slate-12">
                      {{
                        item.runAt
                          ? text.card.types.runAt
                          : text.card.types.recurring
                      }}
                    </dd>
                  </div>
                  <div class="flex items-center gap-2">
                    <dt class="font-medium text-n-slate-9">
                      {{ text.card.labels.when }}
                    </dt>
                    <dd class="text-n-slate-12">
                      {{ humanRecurring(item) }}
                    </dd>
                  </div>
                  <div class="flex items-center gap-2">
                    <dt class="font-medium text-n-slate-9">
                      {{ text.card.labels.timeframe }}
                    </dt>
                    <dd class="text-n-slate-12">
                      <span>
                        {{ formatDateTime(item.startAt) }}
                        {{ text.fallbackDash }}
                        {{ formatDateTime(item.endAt) }}
                      </span>
                    </dd>
                  </div>
                  <div class="flex items-center gap-2">
                    <dt class="font-medium text-n-slate-9">
                      {{ text.card.labels.recipients }}
                    </dt>
                    <dd class="text-n-slate-12">
                      {{ item.recipients.length }}
                    </dd>
                  </div>
                </dl>
              </div>
              <div class="flex flex-col gap-2">
                <Button
                  variant="ghost"
                  color="slate"
                  size="xs"
                  icon="i-lucide-power"
                  :is-loading="togglingId === item.eventId"
                  :aria-label="
                    item.enabled
                      ? text.buttons.toggleDisable
                      : text.buttons.toggleEnable
                  "
                  @click="toggleEnabled(item)"
                />
                <Button
                  variant="ghost"
                  color="slate"
                  size="xs"
                  icon="i-lucide-pencil"
                  :aria-label="text.buttons.editSchedule"
                  @click="openForm(item)"
                />
                <Button
                  variant="ghost"
                  color="ruby"
                  size="xs"
                  icon="i-lucide-trash"
                  :aria-label="text.buttons.deleteSchedule"
                  @click="openDeleteDialog(item)"
                />
              </div>
            </header>

            <footer class="mt-auto space-y-3">
              <Button
                v-if="item.recipients.length"
                variant="ghost"
                color="slate"
                size="sm"
                justify="start"
                class="w-full"
                :icon="
                  isExpanded(item.eventId)
                    ? 'i-lucide-chevron-down'
                    : 'i-lucide-chevron-right'
                "
                :label="
                  isExpanded(item.eventId)
                    ? text.buttons.toggleRecipientsHide
                    : text.buttons.toggleRecipientsShow
                "
                @click="toggleRecipients(item.eventId)"
              />

              <transition name="fade">
                <div
                  v-if="isExpanded(item.eventId)"
                  class="space-y-2 rounded-2xl border border-n-weak bg-n-solid-1/70 p-4"
                >
                  <div
                    v-for="(recipient, index) in item.recipients"
                    :key="`${item.eventId}-${index}`"
                    class="flex items-start justify-between gap-4 rounded-xl border border-n-weak bg-n-solid-2 px-4 py-3"
                  >
                    <div class="space-y-1">
                      <p class="text-sm font-medium text-n-slate-12">
                        {{ recipient.name || '(sem nome)' }}
                      </p>
                      <p class="text-xs text-n-slate-10">
                        {{
                      item.channel === 'email'
                        ? recipient.email
                        : recipient.phone
                    }}
                  </p>
                </div>
                <Button
                  variant="ghost"
                  color="slate"
                  size="xs"
                  icon="i-lucide-eye"
                  :label="text.buttons.preview"
                  @click="openPreview(item, recipient)"
                />
              </div>
            </div>
          </transition>
        </footer>
      </article>
    </div>

    <div
      v-if="!loading && filteredSchedules.length > pageSize"
      class="flex items-center justify-center gap-4"
    >
      <Button
        variant="ghost"
        color="slate"
        size="sm"
        icon="i-lucide-arrow-left"
        :label="text.pagination.previous"
        :disabled="page <= 1"
        @click="goToPreviousPage"
      />
      <span class="text-sm text-n-slate-10">
        {{ text.pagination.info(page, totalPages) }}
      </span>
      <Button
        variant="ghost"
        color="slate"
        size="sm"
        icon="i-lucide-arrow-right"
        :label="text.pagination.next"
        :disabled="page >= totalPages"
        @click="goToNextPage"
      />
    </div>
  </div>
</section>

      <Dialog
        ref="formDialogRef"
        width="4xl"
        :overflow-y-auto="true"
        :title="
          selectedSchedule ? text.buttons.editSchedule : text.buttons.create
        "
        :show-confirm-button="false"
        :show-cancel-button="false"
        @close="resetFormDialogState"
      >
        <EventForm
          :value="selectedSchedule"
          @saved="handleSaved"
          @close="hideFormDialog"
        />
      </Dialog>

      <Dialog
        ref="previewDialogRef"
        width="xl"
        :title="text.preview.title"
        :confirm-button-label="text.preview.close"
        :show-cancel-button="false"
        @confirm="hidePreviewDialog"
        @close="resetPreviewDialogState"
      >
        <div class="space-y-3 text-sm text-n-slate-12">
          <h3 class="text-base font-semibold">{{ previewContent.title }}</h3>
          <pre
            class="max-h-80 overflow-y-auto rounded-xl border border-n-weak bg-n-solid-2 p-4 text-xs whitespace-pre-wrap"
          >
            {{ previewContent.body }}
          </pre>
        </div>
      </Dialog>

      <Dialog
        ref="deleteDialogRef"
        width="sm"
        type="alert"
        :title="text.delete.title"
        :description="text.delete.description"
        :confirm-button-label="text.delete.confirm"
        :cancel-button-label="text.delete.cancel"
        :is-loading="Boolean(deletingId)"
        @confirm="confirmDelete"
        @close="resetDeleteDialogState"
      >
        <p class="text-sm text-n-slate-11">
          {{ text.delete.question(deleteTarget ? deleteTarget.eventId : '') }}
        </p>
      </Dialog>
    </div>
  </template>

  <style scoped>
  .fade-enter-active,
  .fade-leave-active {
    transition: opacity 0.2s ease;
  }
  .fade-enter-from,
  .fade-leave-to {
    opacity: 0;
  }
  </style>
