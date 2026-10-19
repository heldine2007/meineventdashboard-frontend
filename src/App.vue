<script setup>
import { ref, computed, onMounted, watch } from 'vue'
import EventList from './components/EventList.vue'

const events = ref([])
const loading = ref(true)
const error = ref('')

const selectedDate = ref('')
const maxPrice = ref(100)
const maxDistance = ref(50)

const activeTab = ref('discover')
const selectedEvent = ref(null)
const notice = ref('')

const hiddenEvents = computed(() => {
  return events.value.filter(event =>
      hiddenIds.value.includes(event.id)
  )
})

function restoreEvent(event) {
  hiddenIds.value = hiddenIds.value.filter(
      id => id !== event.id
  )

  notice.value = `"${event.name}" wurde wiederhergestellt.`
}

// Entscheidungen bleiben im Browser erhalten.
function readStorage(key) {
  try {
    const value = JSON.parse(localStorage.getItem(key) || '[]')
    return Array.isArray(value) ? value : []
  } catch {
    return []
  }
}

const savedIds = ref(readStorage('event-dashboard-saved'))
const hiddenIds = ref(readStorage('event-dashboard-hidden'))

watch(savedIds, value => {
  localStorage.setItem('event-dashboard-saved', JSON.stringify(value))
}, { deep: true })

watch(hiddenIds, value => {
  localStorage.setItem('event-dashboard-hidden', JSON.stringify(value))
}, { deep: true })

onMounted(async () => {
  try {
    const response = await fetch('http://localhost:8080/')

    if (!response.ok) {
      throw new Error('HTTP ' + response.status)
    }

    events.value = await response.json()
  } catch (e) {
    error.value = 'Events konnten nicht geladen werden. Bitte prüfe, ob dein Backend läuft.'
    console.error(e)
  } finally {
    loading.value = false
  }
})

function toISODate(value) {
  const [day, month, year] = value.split('.')
  return `${year}-${month}-${day}`
}

const visibleEvents = computed(() => {
  return events.value.filter(event =>
      !hiddenIds.value.includes(event.id)
  )
})

const filteredEvents = computed(() => {
  return visibleEvents.value.filter(event => {
    const dateMatches =
        !selectedDate.value ||
        (
            toISODate(event.firstDate) <= selectedDate.value &&
            toISODate(event.lastDate) >= selectedDate.value
        )

    const priceMatches =
        Number(event.price) <= Number(maxPrice.value)

    const distanceMatches =
        Number(event.distance) <= Number(maxDistance.value)

    return dateMatches && priceMatches && distanceMatches
  })
})

const savedEvents = computed(() => {
  return visibleEvents.value.filter(event =>
      savedIds.value.includes(event.id)
  )
})

function toggleSaved(event) {
  if (savedIds.value.includes(event.id)) {
    savedIds.value = savedIds.value.filter(id => id !== event.id)
    notice.value = 'Event aus deinen gespeicherten Events entfernt.'
  } else {
    savedIds.value = [...savedIds.value, event.id]
    notice.value = 'Event wurde gespeichert!'
  }
}

function hideEvent(event) {
  hiddenIds.value = [...new Set([...hiddenIds.value, event.id])]
  savedIds.value = savedIds.value.filter(id => id !== event.id)

  if (selectedEvent.value?.id === event.id) {
    selectedEvent.value = null
  }

  notice.value = 'Event ausgeblendet. Es wird nicht mehr angezeigt.'
}

function resetFilters() {
  selectedDate.value = ''
  maxPrice.value = 100
  maxDistance.value = 50
}
</script>

<template>
  <main class="dashboard">
    <header class="hero">
      <p class="eyebrow">ENTDECKE DEINE STADT</p>
      <h1>Mein Event Dashboard</h1>
      <p>Finde Konzerte, Festivals und Erlebnisse, die zu dir passen.</p>
    </header>

    <nav class="tabs">
      <button
          :class="{ active: activeTab === 'discover' }"
          @click="activeTab = 'discover'"
      >
        Events entdecken
      </button>

      <button
          :class="{ active: activeTab === 'saved' }"
          @click="activeTab = 'saved'"
      >
        ✓ Meine gespeicherten Events ({{ savedEvents.length }})
      </button>

      <button
          :class="{ active: activeTab === 'hidden' }"
          @click="activeTab = 'hidden'"
      >
        Ausgeblendete Events ({{ hiddenEvents.length }})
      </button>
    </nav>

    <p v-if="notice" class="notice" role="status">
      {{ notice }}
      <button @click="notice = ''" aria-label="Meldung schließen">✕</button>
    </p>

    <template v-if="activeTab === 'discover'">
      <section class="filters">
        <h2>Finde dein Event</h2>

        <div class="filter-grid">
          <label>
            Datum
            <input v-model="selectedDate" type="date" />
          </label>

          <label>
            Maximaler Preis: {{ maxPrice }} €
            <input
                v-model.number="maxPrice"
                type="range"
                min="0"
                max="100"
                step="1"
            />
          </label>

          <label>
            Maximale Entfernung: {{ maxDistance }} km
            <input
                v-model.number="maxDistance"
                type="range"
                min="1"
                max="50"
                step="1"
            />
          </label>
        </div>

        <button class="reset" @click="resetFilters">
          Filter zurücksetzen
        </button>
      </section>
    </template>

    <section class="results">
      <template v-if="activeTab === 'discover'">
        <h2>Events entdecken</h2>

        <p v-if="!loading && !error">
          {{ filteredEvents.length }} Events gefunden
        </p>

        <p v-if="loading">Events werden geladen ...</p>
        <p v-else-if="error" class="error">{{ error }}</p>

        <EventList
            v-else
            :events="filteredEvents"
            :saved-ids="savedIds"
            @select="selectedEvent = $event"
            @save="toggleSaved"
            @hide="hideEvent"
        />
      </template>

      <template v-else-if="activeTab === 'saved'">
        <h2>Meine gespeicherten Events</h2>
        <p>Hier findest du deine persönlichen Favoriten.</p>

        <EventList
            :events="savedEvents"
            :saved-ids="savedIds"
            @select="selectedEvent = $event"
            @save="toggleSaved"
            @hide="hideEvent"
        />
      </template>

      <template v-else-if="activeTab === 'hidden'">
        <h2>Ausgeblendete Events</h2>

        <p>
          Hier findest du Events, die du ausgeblendet hast.
          Du kannst sie jederzeit wiederherstellen.
        </p>

        <p v-if="hiddenEvents.length === 0" class="empty-hidden">
          Du hast momentan keine Events ausgeblendet.
        </p>

        <div v-else class="hidden-list">
          <article
              v-for="event in hiddenEvents"
              :key="event.id"
              class="hidden-card"
          >
            <img
                v-if="event.imageUrl"
                :src="event.imageUrl"
                :alt="event.name"
                class="hidden-image"
            />

            <div class="hidden-info">
              <h3>{{ event.name }}</h3>

              <p>
                {{ event.firstDate }}
                <template v-if="event.lastDate !== event.firstDate">
                  – {{ event.lastDate }}
                </template>
                · {{ event.currentTime }} Uhr
              </p>

              <p>
                {{ Number(event.price).toFixed(2) }} €
                · {{ event.distance }} km entfernt
              </p>

              <button
                  class="restore-button"
                  @click="restoreEvent(event)"
              >
                ↩ Event wiederherstellen
              </button>
            </div>
          </article>
        </div>
      </template>
    </section>

    <!-- Detailfenster -->
    <div
        v-if="selectedEvent"
        class="modal-backdrop"
        @click.self="selectedEvent = null"
        @keydown.esc="selectedEvent = null"
    >
      <article class="modal">
        <button
            class="close"
            @click="selectedEvent = null"
            aria-label="Detailansicht schließen"
        >
          ✕
        </button>

        <img
            v-if="selectedEvent.imageUrl"
            class="detail-image"
            :src="selectedEvent.imageUrl"
            :alt="selectedEvent.name"
        />

        <div class="detail-content">
          <p class="eyebrow">{{ selectedEvent.category }}</p>
          <h2>{{ selectedEvent.name }}</h2>

          <p>
            <strong>Datum:</strong>
            {{ selectedEvent.firstDate }}
            <template v-if="selectedEvent.lastDate !== selectedEvent.firstDate">
              – {{ selectedEvent.lastDate }}
            </template>
          </p>

          <p><strong>Uhrzeit:</strong> {{ selectedEvent.currentTime }} Uhr</p>
          <p><strong>Preis:</strong> {{ Number(selectedEvent.price).toFixed(2) }} €</p>
          <p><strong>Entfernung:</strong> {{ selectedEvent.distance }} km</p>

          <p class="description">{{ selectedEvent.description }}</p>

          <div class="detail-actions">
            <button class="save-detail" @click="toggleSaved(selectedEvent)">
              ✓
              {{ savedIds.includes(selectedEvent.id)
                ? 'Aus Favoriten entfernen'
                : 'Event speichern' }}
            </button>

            <button class="hide-detail" @click="hideEvent(selectedEvent)">
              Nicht mehr anzeigen
            </button>
          </div>
        </div>
      </article>
    </div>
  </main>
</template>

<style>
* {
  box-sizing: border-box;
}

body {
  margin: 0;
  background: #f4f6fb;
  color: #202536;
  font-family: Arial, sans-serif;
}

button,
input {
  font: inherit;
}

button {
  cursor: pointer;
}

.dashboard {
  max-width: 1150px;
  margin: auto;
  padding: 28px 22px;
}

.hero {
  padding: 36px;
  border-radius: 20px;
  color: white;
  background: linear-gradient(130deg, #25356b, #5368a6);
  margin-bottom: 24px;
}

.hero h1 {
  margin: 8px 0;
  font-size: clamp(28px, 5vw, 42px);
}

.eyebrow {
  font-size: 12px;
  letter-spacing: 2px;
  font-weight: bold;
}

.tabs {
  display: flex;
  gap: 10px;
  flex-wrap: wrap;
  margin-bottom: 20px;
}

.tabs button {
  border: 1px solid #d9deea;
  padding: 12px 16px;
  border-radius: 10px;
  background: white;
  color: #25356b;
}

.tabs button.active {
  background: #25356b;
  color: white;
}

.filters,
.results {
  background: white;
  padding: 24px;
  border-radius: 16px;
  margin-bottom: 24px;
}

.filter-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(190px, 1fr));
  gap: 22px;
}

.filter-grid label {
  display: flex;
  flex-direction: column;
  gap: 12px;
  font-weight: bold;
}

.filter-grid input {
  width: 100%;
}

.reset {
  margin-top: 22px;
  padding: 10px 14px;
  border: 0;
  border-radius: 8px;
  background: #25356b;
  color: white;
}

.notice {
  display: flex;
  justify-content: space-between;
  gap: 10px;
  padding: 12px 16px;
  border-radius: 10px;
  background: #e2f7e9;
  color: #176b45;
}

.notice button {
  border: 0;
  background: transparent;
}

.error {
  color: #b42318;
}

.modal-backdrop {
  position: fixed;
  inset: 0;
  z-index: 10;
  background: rgba(16, 24, 40, 0.7);
  padding: 20px;
  display: grid;
  place-items: center;
  overflow-y: auto;
}

.modal {
  position: relative;
  width: min(650px, 100%);
  max-height: 90vh;
  overflow-y: auto;
  border-radius: 18px;
  background: white;
}

.close {
  position: absolute;
  right: 12px;
  top: 12px;
  z-index: 1;
  width: 38px;
  height: 38px;
  border: 0;
  border-radius: 50%;
  background: white;
  font-size: 18px;
}

.detail-image {
  display: block;
  width: 100%;
  height: 270px;
  object-fit: cover;
}

.detail-content {
  padding: 24px;
}

.description {
  line-height: 1.7;
  white-space: pre-line;
}

.detail-actions {
  display: flex;
  flex-wrap: wrap;
  gap: 10px;
  margin-top: 22px;
}

.save-detail,
.hide-detail {
  padding: 12px 15px;
  border: 0;
  border-radius: 9px;
}

.save-detail {
  background: #176b45;
  color: white;
}

.hide-detail {
  background: #fff0f0;
  color: #b42318;
}

@media (max-width: 600px) {
  .dashboard {
    padding: 14px;
  }

  .hero {
    padding: 24px;
  }

  .filters,
  .results {
    padding: 16px;
  }

  .hidden-list {
    display: grid;
    gap: 16px;
    margin-top: 20px;
  }

  .hidden-card {
    display: flex;
    align-items: center;
    gap: 18px;
    padding: 16px;
    border: 1px solid #e6e8ef;
    border-radius: 14px;
    background: #fafbfe;
  }

  .hidden-image {
    width: 140px;
    height: 110px;
    flex-shrink: 0;
    object-fit: cover;
    border-radius: 10px;
  }

  .hidden-info {
    flex: 1;
    min-width: 0;
  }

  .hidden-info h3 {
    margin-top: 0;
  }

  .hidden-info p {
    color: #606779;
    font-size: 14px;
  }

  .restore-button {
    padding: 10px 14px;
    border: 0;
    border-radius: 9px;
    background: #176b45;
    color: white;
    font-weight: bold;
    cursor: pointer;
  }

  .restore-button:hover {
    background: #105536;
  }

  .empty-hidden {
    padding: 20px;
    border-radius: 10px;
    background: #f4f6fb;
    color: #606779;
  }

  @media (max-width: 600px) {
    .hidden-card {
      align-items: flex-start;
      flex-direction: column;
    }

    .hidden-image {
      width: 100%;
      height: 180px;
    }
  }
}
</style>
