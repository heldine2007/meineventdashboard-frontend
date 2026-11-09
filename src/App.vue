
<script setup>
import { ref, computed, onMounted, watch } from 'vue'
import EventList from './components/EventList.vue'

// ==================================
// EVENTS UND STATUS
// ==================================

const events = ref([])
const loading = ref(true)
const error = ref('')

const selectedDate = ref('')
const maxPrice = ref(100)
const maxDistance = ref(50)

const activeTab = ref('discover')
const selectedEvent = ref(null)
const notice = ref('')

// Backend-Adresse
const API_URL = (
    import.meta.env.VITE_API_URL || 'http://localhost:8080'
).replace(/\/$/, '')

// ==================================
// EVENTS AUS DER DATENBANK LADEN
// ==================================

async function loadEvents() {
  loading.value = true
  error.value = ''

  try {
    const response = await fetch(`${API_URL}/events`)

    if (!response.ok) {
      throw new Error(`Fehler beim Laden: HTTP ${response.status}`)
    }

    events.value = await response.json()

  } catch (err) {
    console.error('Fehler beim Laden der Events:', err)
    error.value = 'Die Events konnten nicht geladen werden.'

  } finally {
    loading.value = false
  }
}

// ==================================
// NEUES EVENT IN DER DATENBANK SPEICHERN
// ==================================

async function createEvent(newEvent) {
  try {
    const response = await fetch(`${API_URL}/events`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json'
      },
      body: JSON.stringify(newEvent)
    })

    if (!response.ok) {
      throw new Error(
          `Event konnte nicht gespeichert werden: HTTP ${response.status}`
      )
    }

    const savedEvent = await response.json()

    events.value.push(savedEvent)

    notice.value = 'Event wurde erfolgreich gespeichert!'

    return savedEvent

  } catch (err) {
    console.error('Fehler beim Speichern:', err)
    notice.value = 'Das Event konnte nicht gespeichert werden.'
    throw err
  }
}

// ==================================
// NEUES EVENT ERSTELLEN
// ==================================

const showCreateForm = ref(false)
const savingEvent = ref(false)
const formError = ref('')

// Daten des neuen Events
const newEvent = ref({
  name: '',
  firstDate: '',
  lastDate: '',
  currentTime: '',
  price: 0,
  distance: 0,
  pass: false,
  category: '',
  description: '',
  imageUrl: ''
})

// Datum von YYYY-MM-DD in DD.MM.YYYY umwandeln
function formatDateForBackend(value) {
  const [year, month, day] = value.split('-')
  return `${day}.${month}.${year}`
}

// Formular absenden
async function submitNewEvent() {
  formError.value = ''

  // Datum prüfen
  if (!newEvent.value.firstDate || !newEvent.value.lastDate) {
    formError.value = 'Bitte gib ein Start- und Enddatum ein.'
    return
  }

  if (newEvent.value.lastDate < newEvent.value.firstDate) {
    formError.value = 'Das Enddatum darf nicht vor dem Startdatum liegen.'
    return
  }

  savingEvent.value = true

  try {
    const eventToSave = {
      name: newEvent.value.name.trim(),
      firstDate: formatDateForBackend(newEvent.value.firstDate),
      lastDate: formatDateForBackend(newEvent.value.lastDate),
      currentTime: newEvent.value.currentTime,
      price: Number(newEvent.value.price),
      distance: Number(newEvent.value.distance),
      pass: newEvent.value.pass,
      category: newEvent.value.category,
      description: newEvent.value.description,
      imageUrl: newEvent.value.imageUrl
    }

    await createEvent(eventToSave)

    // Formular zurücksetzen
    newEvent.value = {
      name: '',
      firstDate: '',
      lastDate: '',
      currentTime: '',
      price: 0,
      distance: 0,
      pass: false,
      category: '',
      description: '',
      imageUrl: ''
    }

    showCreateForm.value = false

  } catch (err) {
    formError.value =
        'Das Event konnte nicht gespeichert werden. Bitte versuche es erneut.'

  } finally {
    savingEvent.value = false
  }
}

// Beim Öffnen der Anwendung Events laden
onMounted(() => {
  loadEvents()
})

// ==================================
// FAVORITEN UND AUSGEBLENDETE EVENTS
// ==================================

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

// Favoriten im localStorage speichern
watch(savedIds, value => {
  localStorage.setItem(
      'event-dashboard-saved',
      JSON.stringify(value)
  )
}, { deep: true })

// Ausgeblendete Events im localStorage speichern
watch(hiddenIds, value => {
  localStorage.setItem(
      'event-dashboard-hidden',
      JSON.stringify(value)
  )
}, { deep: true })

// ==================================
// AUSGEBLENDETE EVENTS
// ==================================

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

// ==================================
// FILTER
// ==================================

function toISODate(value) {
  if (!value) return ''

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

// ==================================
// GESPEICHERTE EVENTS
// ==================================

const savedEvents = computed(() => {
  return visibleEvents.value.filter(event =>
      savedIds.value.includes(event.id)
  )
})

function toggleSaved(event) {
  if (savedIds.value.includes(event.id)) {
    savedIds.value = savedIds.value.filter(
        id => id !== event.id
    )

    notice.value = 'Event aus deinen gespeicherten Events entfernt.'

  } else {
    savedIds.value = [...savedIds.value, event.id]

    notice.value = 'Event wurde gespeichert!'
  }
}

// ==================================
// EVENTS AUSBLENDEN
// ==================================

function hideEvent(event) {
  hiddenIds.value = [
    ...new Set([...hiddenIds.value, event.id])
  ]

  savedIds.value = savedIds.value.filter(
      id => id !== event.id
  )

  if (selectedEvent.value?.id === event.id) {
    selectedEvent.value = null
  }

  notice.value = 'Event ausgeblendet. Es wird nicht mehr angezeigt.'
}

// ==================================
// FILTER ZURÜCKSETZEN
// ==================================

function resetFilters() {
  selectedDate.value = ''
  maxPrice.value = 100
  maxDistance.value = 50
}
</script>

<template>
  <main class="dashboard">

    <!-- Kopfbereich -->
    <header class="hero">
      <p class="eyebrow">ENTDECKE DEINE STADT</p>
      <h1>Mein Event Dashboard</h1>
      <p>Finde Konzerte, Festivals und Erlebnisse, die zu dir passen.</p>
    </header>

    <!-- Navigation -->
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

    <!-- Benachrichtigungen -->
    <p v-if="notice" class="notice" role="status">
      {{ notice }}

      <button
          @click="notice = ''"
          aria-label="Meldung schließen"
      >
        ✕
      </button>
    </p>

    <!-- ==================================
         EVENTS ENTDECKEN
         ================================== -->

    <template v-if="activeTab === 'discover'">

      <!-- ==================================
           NEUES EVENT HINZUFÜGEN
           ================================== -->

      <section class="create-event-section">

        <div class="create-event-header">
          <div>
            <h2>Eigenes Event hinzufügen</h2>
            <p>Trage ein neues Event ein und speichere es in der Datenbank.</p>
          </div>

          <button
              class="create-toggle"
              @click="showCreateForm = !showCreateForm"
          >
            {{ showCreateForm ? 'Formular schließen' : '+ Event hinzufügen' }}
          </button>
        </div>

        <!-- Eingabeformular -->
        <form
            v-if="showCreateForm"
            class="create-event-form"
            @submit.prevent="submitNewEvent"
        >

          <div class="form-grid">

            <!-- Eventname -->
            <label>
              Eventname *
              <input
                  v-model="newEvent.name"
                  type="text"
                  placeholder="z. B. Sommerfestival Berlin"
                  required
              />
            </label>

            <!-- Kategorie -->
            <label>
              Kategorie *
              <select v-model="newEvent.category" required>
                <option value="" disabled>Bitte auswählen</option>
                <option value="Konzert">Konzert</option>
                <option value="Festival">Festival</option>
                <option value="Kultur">Kultur</option>
                <option value="Sport">Sport</option>
                <option value="Freizeit">Freizeit</option>
                <option value="Comedy">Comedy</option>
                <option value="Ausstellung">Ausstellung</option>
                <option value="Sonstiges">Sonstiges</option>
              </select>
            </label>

            <!-- Startdatum -->
            <label>
              Startdatum *
              <input
                  v-model="newEvent.firstDate"
                  type="date"
                  required
              />
            </label>

            <!-- Enddatum -->
            <label>
              Enddatum *
              <input
                  v-model="newEvent.lastDate"
                  type="date"
                  :min="newEvent.firstDate"
                  required
              />
            </label>

            <!-- Uhrzeit -->
            <label>
              Uhrzeit *
              <input
                  v-model="newEvent.currentTime"
                  type="time"
                  required
              />
            </label>

            <!-- Preis -->
            <label>
              Preis in Euro *
              <input
                  v-model.number="newEvent.price"
                  type="number"
                  min="0"
                  step="0.01"
                  required
              />
            </label>

            <!-- Entfernung -->
            <label>
              Entfernung in km *
              <input
                  v-model.number="newEvent.distance"
                  type="number"
                  min="0"
                  step="0.1"
                  required
              />
            </label>

            <!-- Bild-URL -->
            <label>
              Bild-URL (optional)
              <input
                  v-model="newEvent.imageUrl"
                  type="url"
                  placeholder="https://beispiel.de/bild.jpg"
              />
            </label>

          </div>

          <!-- Beschreibung -->
          <label class="description-label">
            Beschreibung
            <textarea
                v-model="newEvent.description"
                rows="4"
                placeholder="Beschreibe das Event ..."
            ></textarea>
          </label>

          <!-- Pass / Ticket -->
          <label class="pass-label">
            <input
                v-model="newEvent.pass"
                type="checkbox"
            />
            Event beinhaltet einen Pass oder ein Ticket
          </label>

          <!-- Fehlermeldung -->
          <p v-if="formError" class="form-error">
            {{ formError }}
          </p>

          <!-- Buttons -->
          <div class="form-actions">

            <button
                type="submit"
                class="submit-event"
                :disabled="savingEvent"
            >
              {{ savingEvent ? 'Wird gespeichert ...' : '✓ Event speichern' }}
            </button>

            <button
                type="button"
                class="cancel-event"
                @click="showCreateForm = false"
            >
              Abbrechen
            </button>

          </div>

          <p class="required-hint">
            * Pflichtfelder
          </p>

        </form>

      </section>

      <!-- FILTER -->
      <section class="filters">

        <h2>Finde dein Event</h2>

        <div class="filter-grid">

          <label>
            Datum
            <input
                v-model="selectedDate"
                type="date"
            />
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

        <button
            class="reset"
            @click="resetFilters"
        >
          Filter zurücksetzen
        </button>

      </section>

    </template>

    <!-- ==================================
         ERGEBNISSE
         ================================== -->

    <section class="results">

      <!-- Events entdecken -->
      <template v-if="activeTab === 'discover'">

        <h2>Events entdecken</h2>

        <p v-if="!loading && !error">
          {{ filteredEvents.length }} Events gefunden
        </p>

        <p v-if="loading">
          Events werden geladen ...
        </p>

        <p v-else-if="error" class="error">
          {{ error }}
        </p>

        <EventList
            v-else
            :events="filteredEvents"
            :saved-ids="savedIds"
            @select="selectedEvent = $event"
            @save="toggleSaved"
            @hide="hideEvent"
        />

      </template>

      <!-- Gespeicherte Events -->
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

      <!-- Ausgeblendete Events -->
      <template v-else-if="activeTab === 'hidden'">

        <h2>Ausgeblendete Events</h2>

        <p>
          Hier findest du Events, die du ausgeblendet hast.
          Du kannst sie jederzeit wiederherstellen.
        </p>

        <p
            v-if="hiddenEvents.length === 0"
            class="empty-hidden"
        >
          Du hast momentan keine Events ausgeblendet.
        </p>

        <div
            v-else
            class="hidden-list"
        >

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

    <!-- ==================================
         EVENT-DETAILFENSTER
         ================================== -->

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

          <p class="eyebrow">
            {{ selectedEvent.category }}
          </p>

          <h2>{{ selectedEvent.name }}</h2>

          <p>
            <strong>Datum:</strong>
            {{ selectedEvent.firstDate }}

            <template v-if="selectedEvent.lastDate !== selectedEvent.firstDate">
              – {{ selectedEvent.lastDate }}
            </template>
          </p>

          <p>
            <strong>Uhrzeit:</strong>
            {{ selectedEvent.currentTime }} Uhr
          </p>

          <p>
            <strong>Preis:</strong>
            {{ Number(selectedEvent.price).toFixed(2) }} €
          </p>

          <p>
            <strong>Entfernung:</strong>
            {{ selectedEvent.distance }} km
          </p>

          <p class="description">
            {{ selectedEvent.description }}
          </p>

          <div class="detail-actions">

            <button
                class="save-detail"
                @click="toggleSaved(selectedEvent)"
            >
              ✓

              {{ savedIds.includes(selectedEvent.id)
                ? 'Aus Favoriten entfernen'
                : 'Event speichern' }}
            </button>

            <button
                class="hide-detail"
                @click="hideEvent(selectedEvent)"
            >
              Nicht mehr anzeigen
            </button>

          </div>

        </div>

      </article>

    </div>

  </main>
</template>

<style>

/* Allgemeine Einstellungen */

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

/* Hauptbereich */

.dashboard {
  max-width: 1150px;
  margin: auto;
  padding: 28px 22px;
}

/* Kopfbereich */

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

/* Navigation und Tabs */

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
  transition: background 0.2s ease, color 0.2s ease;
}

.tabs button:hover {
  background: #edf1fc;
}

.tabs button.active {
  background: #25356b;
  color: white;
}

.tabs button.active:hover {
  background: #1b2854;
}

/* Filter und Ergebnisse */

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
  transition: background 0.2s ease;
}

.reset:hover {
  background: #1b2854;
}

/* Rückmeldungen */

.notice {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 10px;
  padding: 12px 16px;
  border-radius: 10px;
  background: #e2f7e9;
  color: #176b45;
}

.notice button {
  border: 0;
  background: transparent;
  color: inherit;
}

.error {
  color: #b42318;
}

/* ==================================
   NEUES EVENT ERSTELLEN
   ================================== */

.create-event-section {
  background: white;
  padding: 24px;
  border-radius: 16px;
  margin-bottom: 24px;
  border: 1px solid #e1e6f0;
}

.create-event-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 20px;
  flex-wrap: wrap;
}

.create-event-header h2 {
  margin-top: 0;
  margin-bottom: 8px;
}

.create-event-header p {
  color: #606779;
  margin-bottom: 0;
}

.create-toggle {
  padding: 12px 18px;
  border: none;
  border-radius: 9px;
  background: #25356b;
  color: white;
  font-weight: bold;
  transition: background 0.2s ease;
}

.create-toggle:hover {
  background: #1b2854;
}

.create-event-form {
  margin-top: 24px;
  padding-top: 22px;
  border-top: 1px solid #e1e6f0;
}

.form-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(220px, 1fr));
  gap: 20px;
}

.form-grid label,
.description-label {
  display: flex;
  flex-direction: column;
  gap: 8px;
  font-weight: bold;
}

.form-grid input,
.form-grid select,
.description-label textarea {
  width: 100%;
  padding: 11px;
  border: 1px solid #d9deea;
  border-radius: 8px;
  background: white;
  color: #202536;
}

.form-grid input:focus,
.form-grid select:focus,
.description-label textarea:focus {
  outline: 2px solid #5368a6;
  border-color: #5368a6;
}

.description-label {
  margin-top: 22px;
}

.description-label textarea {
  resize: vertical;
  font-family: Arial, sans-serif;
  line-height: 1.5;
}

.pass-label {
  display: flex;
  align-items: center;
  gap: 10px;
  margin-top: 20px;
  font-weight: normal;
}

.pass-label input {
  width: 18px;
  height: 18px;
}

.form-actions {
  display: flex;
  flex-wrap: wrap;
  gap: 12px;
  margin-top: 22px;
}

.submit-event,
.cancel-event {
  padding: 12px 18px;
  border: none;
  border-radius: 9px;
  font-weight: bold;
}

.submit-event {
  background: #176b45;
  color: white;
}

.submit-event:hover {
  background: #105536;
}

.submit-event:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

.cancel-event {
  background: #edf0f6;
  color: #25356b;
}

.cancel-event:hover {
  background: #dfe4ef;
}

.form-error {
  color: #b42318;
  margin-top: 16px;
}

.required-hint {
  color: #606779;
  font-size: 12px;
  margin-top: 14px;
}

/* ==================================
   AUSGEBLENDETE EVENTS
   ================================== */

.hidden-list {
  display: flex;
  flex-direction: column;
  gap: 26px;
  max-width: 900px;
  margin: 28px auto 12px;
}

.hidden-card {
  display: flex;
  align-items: center;
  gap: 22px;
  padding: 20px;
  background: #ffffff;
  border: 1px solid #e1e6f0;
  border-radius: 16px;
  box-shadow: 0 4px 14px rgba(0, 0, 0, 0.045);
  transition: box-shadow 0.2s ease, border-color 0.2s ease;
}

.hidden-card:hover {
  border-color: #cbd5e8;
  box-shadow: 0 6px 18px rgba(0, 0, 0, 0.07);
}

.hidden-image {
  display: block;
  width: 125px;
  height: 95px;
  flex: 0 0 125px;
  object-fit: cover;
  border-radius: 12px;
  background: #e9edf5;
}

.hidden-info {
  flex: 1;
  min-width: 0;
}

.hidden-info h3 {
  margin: 0 0 12px;
  font-size: 18px;
  line-height: 1.4;
  overflow-wrap: anywhere;
}

.hidden-info p {
  margin: 7px 0;
  color: #606779;
  font-size: 14px;
  line-height: 1.5;
}

/* Wiederherstellen-Button */

.restore-button {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: 8px;
  margin-top: 12px;
  padding: 11px 20px;
  border: none;
  border-radius: 999px;
  background: #2563eb;
  color: #ffffff;
  font-size: 14px;
  font-weight: 600;
  line-height: 1.4;
  box-shadow: 0 2px 5px rgba(37, 99, 235, 0.15);
  transition:
      background 0.2s ease,
      box-shadow 0.2s ease,
      transform 0.2s ease;
}

.restore-button:hover {
  background: #1d4ed8;
  box-shadow: 0 4px 10px rgba(37, 99, 235, 0.25);
  transform: translateY(-1px);
}

.restore-button:focus-visible {
  outline: 3px solid #93c5fd;
  outline-offset: 3px;
}

/* Leere Liste */

.empty-hidden {
  margin: 28px 0 8px;
  padding: 24px;
  border-radius: 14px;
  background: #f4f6fb;
  color: #606779;
  text-align: center;
  line-height: 1.6;
}

/* ==================================
   EVENT-DETAILFENSTER
   ================================== */

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
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.12);
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

.save-detail:hover {
  background: #105536;
}

.hide-detail {
  background: #fff0f0;
  color: #b42318;
}

.hide-detail:hover {
  background: #ffe0e0;
}

/* ==================================
   RESPONSIVE DESIGN
   ================================== */

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

  .tabs {
    gap: 8px;
  }

  .tabs button {
    padding: 10px 12px;
    font-size: 14px;
  }

  /* Ausgeblendete Events auf dem Smartphone */

  .hidden-list {
    gap: 20px;
    margin-top: 22px;
  }

  .hidden-card {
    flex-direction: column;
    align-items: stretch;
    gap: 14px;
    padding: 16px;
  }

  .hidden-image {
    width: 100%;
    height: 145px;
    flex: none;
  }

  .hidden-info h3 {
    font-size: 17px;
  }

  .restore-button {
    margin-top: 10px;
    padding: 11px 18px;
    align-self: flex-start;
  }

  .empty-hidden {
    padding: 20px 14px;
  }

  /* Detailfenster auf kleinen Bildschirmen */

  .modal-backdrop {
    padding: 12px;
  }

  .detail-image {
    height: 200px;
  }

  .detail-content {
    padding: 18px;
  }

  .detail-actions {
    flex-direction: column;
  }

  .save-detail,
  .hide-detail {
    width: 100%;
  }

  /* Neues Event-Formular auf Smartphones */

  .create-event-section {
    padding: 16px;
  }

  .create-event-header {
    align-items: stretch;
  }

  .create-toggle {
    width: 100%;
  }

  .form-grid {
    grid-template-columns: 1fr;
  }

  .form-actions {
    flex-direction: column;
  }

  .submit-event,
  .cancel-event {
    width: 100%;
  }
}

</style>