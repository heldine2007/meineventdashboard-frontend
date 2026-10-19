<script setup>
defineProps({
  events: {
    type: Array,
    required: true
  },
  savedIds: {
    type: Array,
    default: () => []
  }
})

const emit = defineEmits([
  'select',
  'save',
  'hide'
])
</script>

<template>
  <section class="event-grid">
    <p v-if="events.length === 0" class="empty">
      Hier gibt es momentan keine Events.
    </p>

    <article
        v-for="event in events"
        :key="event.id"
        class="event-card"
    >
      <div
          class="event-image"
          @click="emit('select', event)"
          role="button"
          tabindex="0"
          @keydown.enter="emit('select', event)"
      >
        <img
            v-if="event.imageUrl"
            :src="event.imageUrl"
            :alt="event.name"
            loading="lazy"
        />
        <div v-else class="no-image">
          Kein Bild verfügbar
        </div>

        <span class="category">{{ event.category }}</span>
      </div>

      <div class="event-content">
        <h3
            class="event-title"
            @click="emit('select', event)"
        >
          {{ event.name }}
        </h3>

        <p class="event-date">
          {{ event.firstDate }}
          <template v-if="event.lastDate !== event.firstDate">
            – {{ event.lastDate }}
          </template>
          · {{ event.currentTime }} Uhr
        </p>

        <div class="event-meta">
          <span>
            {{ Number(event.price).toFixed(2) }} €
          </span>
          <span>{{ event.distance }} km entfernt</span>
        </div>

        <div class="actions">
          <button
              class="save-button"
              :class="{ saved: savedIds.includes(event.id) }"
              @click="emit('save', event)"
              :aria-label="'Event speichern: ' + event.name"
          >
            <span>✓</span>
            {{ savedIds.includes(event.id) ? 'Gespeichert' : 'Merken' }}
          </button>

          <button
              class="hide-button"
              @click="emit('hide', event)"
              :aria-label="'Event ausblenden: ' + event.name"
              title="Dieses Event nicht mehr anzeigen"
          >
            ✕
          </button>
        </div>
      </div>
    </article>
  </section>
</template>

<style scoped>
.event-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(240px, 1fr));
  gap: 20px;
}

.event-card {
  background: white;
  border: 1px solid #e6e8ef;
  border-radius: 16px;
  overflow: hidden;
  box-shadow: 0 4px 14px rgba(0, 0, 0, 0.05);
}

.event-image {
  height: 175px;
  position: relative;
  cursor: pointer;
  background: #e9edf5;
}

.event-image img {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.no-image {
  height: 100%;
  display: grid;
  place-items: center;
  color: #666;
}

.category {
  position: absolute;
  left: 12px;
  top: 12px;
  padding: 6px 10px;
  border-radius: 20px;
  background: white;
  font-size: 12px;
  font-weight: bold;
}

.event-content {
  padding: 16px;
}

.event-title {
  cursor: pointer;
  margin: 0 0 10px;
}

.event-date {
  color: #606779;
  font-size: 14px;
}

.event-meta {
  display: flex;
  justify-content: space-between;
  gap: 8px;
  font-size: 13px;
  margin: 16px 0;
}

.actions {
  display: flex;
  gap: 10px;
}

.save-button,
.hide-button {
  border: 0;
  border-radius: 9px;
  padding: 10px 12px;
  cursor: pointer;
  font-weight: bold;
}

.save-button {
  flex: 1;
  color: #176b45;
  background: #e2f7e9;
}

.save-button.saved {
  background: #176b45;
  color: white;
}

.hide-button {
  background: #fff0f0;
  color: #b42318;
}

.empty {
  color: #606779;
}
</style>
