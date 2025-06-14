<template>
  <div class="header">
    <div class="header-top">
      <div class="title-container">
        <h1 class="title" @click="startEditingTitle" v-if="!isEditingTitle">
          {{ boardTitle }}
          <span class="edit-icon">⚙️</span>
        </h1>
        <div v-else class="title-input-container">
          <input
            ref="titleInput"
            v-model="editingTitle"
            @keyup.enter="saveBoardTitle"
            @blur="saveBoardTitle"
            @keyup.esc="cancelEditTitle"
            class="title-input"
          />
        </div>
      </div>
      <div class="header-buttons">
        <button @click="$emit('config-click')" class="json-button">
          CONFIGURAR
        </button>
      </div>
    </div>

    <!-- Buscador y filtros -->
    <div class="search-filters-section">
      <div class="search-container">
        <input :value="searchTerm" @input="handleSearch" type="text" placeholder="Buscar por título o ID..."
          class="search-input" />
        <button v-if="searchTerm" @click="clearSearch" class="clear-search-button">×</button>
      </div>

      <div class="filters-container">
        <div class="filter-group">
          <label class="filter-label">Productos:</label>
          <div class="multiselect-container">
            <div @click="showProductDropdown = !showProductDropdown" class="multiselect-trigger">
              <span v-if="selectedProducts.length === 0">Todos los productos</span>
              <span v-else>{{ selectedProducts.length }} seleccionado(s)</span>
              <span class="dropdown-arrow">▼</span>
            </div>
            <div v-if="showProductDropdown" class="multiselect-dropdown">
              <div v-for="product in availableProducts" :key="product" @click="toggleProduct(product)"
                class="multiselect-option" :class="{ 'selected': selectedProducts.includes(product) }">
                <input type="checkbox" :checked="selectedProducts.includes(product)" @click.stop />
                {{ product }}
              </div>
            </div>
          </div>
        </div>

        <div class="filter-group">
          <label class="filter-label">Prioridad:</label>
          <div class="multiselect-container">
            <div @click="showPriorityDropdown = !showPriorityDropdown" class="multiselect-trigger">
              <span v-if="selectedPriorities.length === 0">Todas las prioridades</span>
              <span v-else>{{ selectedPriorities.length }} seleccionada(s)</span>
              <span class="dropdown-arrow">▼</span>
            </div>
            <div v-if="showPriorityDropdown" class="multiselect-dropdown">
              <div v-for="priority in availablePriorities" :key="priority.value"
                @click="togglePriority(priority.value)" class="multiselect-option"
                :class="{ 'selected': selectedPriorities.includes(priority.value) }">
                <input type="checkbox" :checked="selectedPriorities.includes(priority.value)" @click.stop />
                <span class="priority-indicator" :class="priority.class"></span>
                {{ priority.label }}
              </div>
            </div>
          </div>
        </div>
      </div>

      <button @click="$emit('add-task')" class="add-button">
        + Añadir Tarea
      </button>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, nextTick, defineProps, defineEmits } from 'vue';

const emit = defineEmits([
  'update:boardTitle',
  'update:searchTerm',
  'update:selectedProducts',
  'update:selectedPriorities',
  'search',
  'config-click',
  'add-task'
]);

const props = defineProps({
  boardTitle: {
    type: String,
    required: true
  },
  availableProducts: {
    type: Array,
    default: () => []
  },
  availablePriorities: {
    type: Array,
    default: () => []
  },
  selectedProducts: {
    type: Array,
    default: () => []
  },
  selectedPriorities: {
    type: Array,
    default: () => []
  },
  searchTerm: {
    type: String,
    default: ''
  }
});



const isEditingTitle = ref(false);
const editingTitle = ref('');
const titleInput = ref(null);
const showProductDropdown = ref(false);
const showPriorityDropdown = ref(false);

const startEditingTitle = () => {
  editingTitle.value = props.boardTitle;
  isEditingTitle.value = true;
  nextTick(() => {
    titleInput.value.focus();
  });
};

const saveBoardTitle = () => {
  if (editingTitle.value.trim()) {
    emit('update:boardTitle', editingTitle.value.trim());
  }
  isEditingTitle.value = false;
};

const cancelEditTitle = () => {
  isEditingTitle.value = false;
};

const handleSearch = (event) => {
  emit('update:searchTerm', event.target.value);
  emit('search', event.target.value);
};

const clearSearch = () => {
  emit('update:searchTerm', '');
  emit('search', '');
};

const toggleProduct = (product) => {
  const newProducts = [...props.selectedProducts];
  const index = newProducts.indexOf(product);
  if (index === -1) {
    newProducts.push(product);
  } else {
    newProducts.splice(index, 1);
  }
  emit('update:selectedProducts', newProducts);};

const togglePriority = (priority) => {
  const newPriorities = [...props.selectedPriorities];
  const index = newPriorities.indexOf(priority);
  if (index === -1) {
    newPriorities.push(priority);
  } else {
    newPriorities.splice(index, 1);
  }
  emit('update:selectedPriorities', newPriorities);
};

// Cerrar dropdowns al hacer clic fuera
const handleClickOutside = (event) => {
  if (!event.target.closest('.multiselect-container')) {
    showProductDropdown.value = false;
    showPriorityDropdown.value = false;
  }
};

onMounted(() => {
  document.addEventListener('click', handleClickOutside);
});
</script>

<style scoped>
@import '../assets/css/style.css';
</style>


