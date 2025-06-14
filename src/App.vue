<template>
  <div class="kanban-container">
    <div class="kanban-wrapper">
      <KanbanHeader
        v-model:boardTitle="boardTitle"
        v-model:searchTerm="searchTerm"
        v-model:selectedProducts="selectedProducts"
        v-model:selectedPriorities="selectedPriorities"
        :available-products="availableProducts"
        :available-priorities="availablePriorities"
        @search="handleSearch"
        @config-click="showConfigModal = true"
        @add-task="openModal"
      />
     
      <!-- Modal configuracion-->
      <div v-if="showConfigModal" class="modal-overlay" @click="showConfigModal = false">
        <div class="modal-content config-modal" @click.stop>
          <div class="modal-header">
            <h2 class="modal-title">Configuración</h2>
            <button @click="showConfigModal = false" class="close-button-red">×</button>
          </div>
          
          <!-- Pestañas de navegación -->
          <div class="settings-tabs">
            <button 
              @click="activeSettingsTab = 'columns'" 
              :class="['tab-button', { active: activeSettingsTab === 'columns' }]"
            >
              Columnas
            </button>
            <button 
              @click="activeSettingsTab = 'data'" 
              :class="['tab-button', { active: activeSettingsTab === 'data' }]"
            >
              Datos
            </button>
          </div>
          
          <!-- Contenido de Columnas -->
          <div v-if="activeSettingsTab === 'columns'" class="config-modal-content">
            <div class="column-types-container">
              <div class="config-actions" style="margin-top: 0; padding-top: 0; border: none; justify-content: space-between; margin-bottom: 15px;">
                <h3>Tipos de Columna</h3>
                <div>
                </div>
              </div>
              
              <div 
                class="column-types-list" 
                @dragover.prevent
                @drop="onColumnTypeDrop"
              >
                <div 
                  v-for="(type, index) in columnTypes" 
                  :key="type.id" 
                  class="column-type-item"
                  draggable="true"
                  @dragstart="onColumnTypeDragStart($event, index)"
                  @dragover.prevent="onColumnTypeDragOver($event, index)"
                  @dragend="onColumnTypeDragEnd"
                >
                  <div class="drag-handle" title="Arrastrar para reordenar">☰</div>
                  <div class="color-preview" :style="{ backgroundColor: type.color }"></div>
                  <input type="text" v-model="type.name" class="column-name-input" />
                  <input type="color" v-model="type.color" class="color-picker" />
                  <button @click="removeColumnType(index)" class="remove-column-btn" title="Eliminar columna" :disabled="columnTypes.length <= 1">
                    ×
                  </button>
                </div>
              </div>
              
              <div class="add-column-form">
                <h4>Agregar nuevo tipo de columna</h4>
                <div class="form-row">
                  <input 
                    type="text" 
                    v-model="newColumnType.name" 
                    placeholder="Nombre de la columna"
                    class="column-name-input"
                  />
                  <input 
                    type="color" 
                    v-model="newColumnType.color" 
                    class="color-picker"
                  />
                  <button 
                    @click="addColumnType" 
                    class="add-column-btn"
                    :disabled="!newColumnType.name.trim()"
                  >
                    Agregar
                  </button>
                </div>
              </div>
            </div>
            
            <div class="config-actions">
              <button @click="saveColumnTypes" class="save-btn">Guardar Cambios</button>
              <button @click="loadColumnTypes" class="cancel-btn">Cancelar</button>
            </div>
          </div>
          
          <!-- Contenido de Datos -->
          <div v-if="activeSettingsTab === 'data'" class="data-view-content">
            <div class="data-view-actions">
              <button @click="exportData" class="export-config-btn">
                Exportar Todos los Datos
              </button>
              <button @click="triggerFileInput" class="import-config-btn">
                Importar Datos
              </button>
              <input type="file" ref="fileInput" @change="importData" accept=".json" style="display: none" />
            </div>
            
            <div class="json-viewer">
              <pre class="json-display">{{ 
                JSON.stringify({
                  settings: {
                    title: boardTitle,
                    columnTypes: columnTypes
                  },
                  tasks: enhancedTasksWithComments,
                  comments: comments.value,
                  version: '1.0.0'
                }, null, 2) 
              }}</pre>
            </div>
            
            <div class="config-actions">
              <button @click="copyJsonToClipboard" class="copy-json-btn">
                {{ jsonCopied ? '¡Copiado!' : 'Copiar JSON' }}
              </button>
              <button @click="showConfigModal = false" class="cancel-btn">Cerrar</button>
            </div>
          </div>
        </div>
      </div>

      <!-- Modal JSON -->
      <div v-if="showJsonModal" class="modal-overlay" @click="showJsonModal = false">
        <div class="modal-content json-modal" @click.stop>
          <div class="modal-header">
            <h2 class="modal-title">Estado actual del JSON</h2>
            <div class="modal-header-actions">
              <button @click="copyJsonToClipboard" class="copy-button" :class="{ 'copied': jsonCopied }">
                {{ jsonCopied ? '✓ Copiado' : '📋 Copiar' }}
              </button>
              <button @click="showJsonModal = false" class="close-button-red">×</button>
            </div>
          </div>
          <div class="json-modal-content">
            <pre class="json-display-modal">{{ JSON.stringify({
              tasks: enhancedTasksWithComments,
              comments: comments.value,
              settings: {
                title_board: boardTitle,
                version: '1.0.0'
              }
            }, null, 2) }}</pre>
          </div>
        </div>
      </div>

      <!-- Modal para crear/editar tarea -->
      <div v-if="showModal" class="modal-overlay">
        <div class="modal-content modal-wide" @click.stop>
          <div class="modal-header">
            <h2 class="modal-title">{{ editingTask ? 'Editar Tarea' : 'Crear Nueva Tarea' }}</h2>
            <button @click="closeModal" class="close-button-red">×</button>
          </div>

          <form @submit.prevent="saveTask" class="modal-form">
            <div class="form-group">
              <label for="taskName" class="form-label">Nombre de la tarea *</label>
              <input id="taskName" v-model="newTask.name" type="text" class="form-input"
                placeholder="Ingresa el nombre de la tarea..." required />
            </div>

            <div class="form-group">
              <label for="taskDescription" class="form-label">Descripción <span class="optional-label">(opcional)</span></label>
              <textarea id="taskDescription" v-model="newTask.description" class="form-textarea"
                placeholder="Describe la tarea..." rows="3"></textarea>
            </div>

            <div class="form-group">
              <label for="taskProduct" class="form-label">Productos *</label>
              <div class="product-input-container">
                <div class="selected-products" v-if="newTask.products && newTask.products.length > 0">
                  <span v-for="product in newTask.products" :key="product" class="product-tag">
                    {{ product }}
                    <button type="button" @click="removeProduct(product)" class="remove-product">×</button>
                  </span>
                </div>
                <div class="product-autocomplete">
                  <input id="taskProduct" v-model="productSearchTerm" @input="filterProducts"
                    @focus="showProductSuggestions = true" @keydown.enter.prevent="addCurrentProduct"
                    @keydown.tab="addCurrentProduct" type="text" class="form-input"
                    placeholder="Buscar productos existentes... (presiona Enter para crear)" />
                  <div v-if="showProductSuggestions && filteredProductSuggestions.length > 0"
                    class="product-suggestions">
                    <div v-for="product in filteredProductSuggestions" :key="product" @click="addProduct(product)"
                      class="product-suggestion">
                      {{ product }}
                    </div>
                  </div>
                </div>
              </div>
            </div>

            <div class="form-group">
              <label for="taskPriority" class="form-label">Prioridad *</label>
              <select id="taskPriority" v-model="newTask.priority" class="form-input" required>
                <option value="">Selecciona una prioridad</option>
                <option v-for="priority in availablePriorities" :key="priority.value" :value="priority.value">
                  {{ priority.label }}
                </option>
              </select>
            </div>

            <div class="form-group">
              <label for="jiraLink" class="form-label">JIRA LINK <span class="optional-label">(opcional)</span></label>
              <div class="input-with-validation">
                <input 
                  id="jiraLink" 
                  v-model="newTask.jiraLink" 
                  type="url" 
                  class="form-input" 
                  :class="{ 'input-error': newTask.jiraLink && !isValidUrl(newTask.jiraLink) }"
                  placeholder="https://jira.company.com/browse/PROJ-123" 
                  @input="validateJiraLink"
                />
                <span v-if="newTask.jiraLink" class="validation-icon" :class="{ 'valid': isValidUrl(newTask.jiraLink), 'invalid': !isValidUrl(newTask.jiraLink) }">
                  {{ isValidUrl(newTask.jiraLink) ? '✓' : '✗' }}
                </span>
              </div>
              <div v-if="newTask.jiraLink && !isValidUrl(newTask.jiraLink)" class="form-error">
                Por favor, ingresa una URL válida
              </div>
              <div v-else class="form-hint">
                Deja vacío si no hay un enlace de Jira
              </div>
            </div>

            <div class="form-row">
              <div class="form-group">
                <label for="startDate" class="form-label">Fecha de inicio <span class="optional-label">(opcional)</span></label>
                <input id="startDate" v-model="newTask.startDate" type="date" class="form-input" />
              </div>
              <div class="form-group">
                <label for="endDate" class="form-label">Fecha de fin <span class="optional-label">(opcional)</span></label>
                <input id="endDate" v-model="newTask.endDate" type="date" class="form-input" :min="newTask.startDate" />
              </div>
            </div>

            <div class="form-group">
              <label for="parentTask" class="form-label">Tarea Padre <span
                  class="optional-label">(opcional)</span></label>
              <div class="select-container">
                <input id="parentTask" v-model="parentSearchTerm" @input="filterParentTasks"
                  @focus="showParentDropdown = true" @click="showParentDropdown = true" type="text" class="form-input"
                  placeholder="Buscar por nombre o ID..." autocomplete="off" />
                <div v-if="showParentDropdown && (filteredParentTasks.length > 0 || parentSearchTerm)" class="dropdown">
                  <div v-for="task in filteredParentTasks" :key="task.id" @click="selectParentTask(task)"
                    class="dropdown-item">
                    <div class="dropdown-item-content">
                      <span class="dropdown-item-name">{{ task.name }}</span>
                      <span class="dropdown-item-id">ID: {{ task.id }}</span>
                      <span class="dropdown-item-status">{{ getStatusLabel(task.status) }}</span>
                    </div>
                    <div v-if="task.description" class="dropdown-item-description">
                      {{ truncateText(task.description, 50) }}
                    </div>
                  </div>
                  <div v-if="filteredParentTasks.length === 0 && parentSearchTerm" class="dropdown-item-empty">
                    No se encontraron tareas
                  </div>
                </div>
              </div>
              <div v-if="newTask.parentId" class="selected-parent">
                <span>Tarea padre seleccionada: <strong>{{ getTaskById(newTask.parentId)?.name }}</strong> (ID: {{
                  newTask.parentId }})</span>
                <button type="button" @click="clearParentTask" class="clear-parent">×</button>
              </div>
            </div>

            <div class="modal-actions">
              <button type="button" @click="closeModal" class="cancel-button">
                Cancelar
              </button>
              <button type="submit"
                :disabled="!isFormValid" 
                class="submit-button">
                {{ editingTask ? 'Guardar Cambios' : 'Crear Tarea' }}
              </button>
            </div>
          </form>
        </div>
      </div>

      <!-- Modal combinado de comentarios y vista de tarea -->
      <div v-if="showCommentsModal" class="modal-overlay combined-modal-overlay" @click="closeCommentsModal">
        <div class="combined-modal-container" @click.stop>
          <!-- Botón de cierre único -->
          <button @click="closeCommentsModal" class="close-button-red combined-close-button">×</button>
          
          <div class="combined-modal-content">
            <!-- Sección de comentarios -->
            <div class="comments-section">
              <div class="modal-header">
                <h2 class="modal-title">Comentarios - {{ commentsTask?.name }}</h2>
              </div>

              <div class="comments-modal-content">
                <!-- Lista de comentarios -->
                <div class="comments-list">
                  <div v-if="getTaskComments(commentsTask?.id).length === 0" class="no-comments">
                    No hay comentarios aún. ¡Añade el primero!
                  </div>
                  <div v-for="comment in getTaskComments(commentsTask?.id)" :key="comment.id" class="comment-item" :class="[
                    getCommentMatrixClass(comment.important, comment.urgent),
                    { 'comment-completed': comment.isCompleted }
                  ]">
                    <div class="comment-header">
                      <div class="comment-header-left">
                        <span class="comment-priority" :class="getCommentPriorityClass(comment.important, comment.urgent)">
                          {{ getCommentPriorityLabel(comment.important, comment.urgent) }}
                        </span>
                        <span class="comment-date">{{ formatDateTime(comment.createdAt) }}</span>
                        <span v-if="comment.isCompleted && comment.completedAt" class="comment-completed-date">
                          ✓ Completado: {{ formatDateTime(comment.completedAt) }}
                        </span>
                      </div>
                      <div class="comment-actions-buttons">
                        <button @click="toggleCommentCompleted(comment.id)" class="comment-action-btn"
                          :class="{ 'completed': comment.isCompleted }"
                          :title="comment.isCompleted ? 'Marcar como pendiente' : 'Marcar como completado'">
                          {{ comment.isCompleted ? '✓' : '○' }}
                        </button>
                        <button @click="deleteComment(comment.id)" class="comment-action-btn delete"
                          title="Eliminar comentario">
                          🗑️
                        </button>
                      </div>
                    </div>
                    <div class="comment-text" :class="{ 'completed-text': comment.isCompleted }">
                      {{ comment.text }}
                    </div>
                  </div>
                </div>

                <!-- Formulario para nuevo comentario -->
                <div class="new-comment-form">
                  <div class="form-group">
                    <label class="form-label">Nuevo comentario</label>
                    <textarea v-model="newComment.text" class="form-textarea" placeholder="Escribe tu comentario..."
                      rows="3"></textarea>
                  </div>

                  <!-- Matriz de Eisenhower para comentarios -->
                  <div class="form-group">
                    <label class="form-label">Prioridad del comentario (Matriz de Eisenhower)</label>
                    <div class="eisenhower-matrix">
                      <div class="matrix-row">
                        <label class="matrix-option">
                          <input type="radio" v-model="newComment.matrixValue" value="important-urgent"
                            name="commentPriority" />
                          <span class="matrix-label important-urgent">
                            <strong>Importante y Urgente</strong>
                            <small>Hacer ahora</small>
                          </span>
                        </label>
                        <label class="matrix-option">
                          <input type="radio" v-model="newComment.matrixValue" value="important-not-urgent"
                            name="commentPriority" />
                          <span class="matrix-label important-not-urgent">
                            <strong>Importante y No urgente</strong>
                            <small>Planificar</small>
                          </span>
                        </label>
                      </div>
                      <div class="matrix-row">
                        <label class="matrix-option">
                          <input type="radio" v-model="newComment.matrixValue" value="not-important-urgent"
                            name="commentPriority" />
                          <span class="matrix-label not-important-urgent">
                            <strong>No importante y Urgente</strong>
                            <small>Delegar</small>
                          </span>
                        </label>
                        <label class="matrix-option">
                          <input type="radio" v-model="newComment.matrixValue" value="not-important-not-urgent"
                            name="commentPriority" />
                          <span class="matrix-label not-important-not-urgent">
                            <strong>No importante y No urgente</strong>
                            <small>Eliminar</small>
                          </span>
                        </label>
                      </div>
                    </div>
                  </div>

                  <div class="comment-actions">
                    <button @click="addComment" :disabled="!newComment.text.trim() || !newComment.matrixValue"
                      class="submit-button">
                      Añadir Comentario
                    </button>
                  </div>
                </div>
              </div>
            </div>

            <!-- Sección de vista de tarea -->
            <div class="task-view-section">
              <div class="modal-header">
                <h2 class="modal-title">{{ commentsTask?.name }}</h2>
              </div>
              <div class="view-modal-content">
                <div class="view-field">
                  <strong>ID:</strong> {{ commentsTask?.id }}
                </div>
                <div class="view-field">
                  <strong>Estado:</strong> {{ getStatusLabel(commentsTask?.status) }}
                </div>
                <div class="view-field">
                  <strong>Prioridad:</strong>
                  <span class="priority-badge" :class="getPriorityClass(commentsTask?.priority)">
                    {{ getPriorityLabel(commentsTask?.priority) }}
                  </span>
                </div>
                <div class="view-field">
                  <strong>Productos:</strong>
                  <span v-if="commentsTask?.products && commentsTask.products.length > 0">
                    {{ commentsTask.products.join(', ') }}
                  </span>
                  <span v-else>{{ commentsTask?.product || 'No especificado' }}</span>
                </div>
                <div v-if="commentsTask?.description" class="view-field">
                  <strong>Descripción:</strong> {{ commentsTask.description }}
                </div>
                <div v-if="commentsTask?.jiraLink" class="view-field">
                  <strong>JIRA LINK:</strong>
                  <a :href="commentsTask.jiraLink" target="_blank" rel="noopener noreferrer" class="jira-link">
                    {{ commentsTask.jiraLink }}
                  </a>
                </div>
                <div v-if="commentsTask?.parentId" class="view-field">
                  <strong>Tarea Padre:</strong>
                  <button @click="highlightTask(commentsTask.parentId)" class="link-button">
                    {{ getTaskById(commentsTask.parentId)?.name }} (ID: {{ commentsTask.parentId }})
                  </button>
                </div>
                <div v-if="getChildrenTasks(commentsTask?.id).length > 0" class="view-field">
                  <strong>Subtareas ({{ getChildrenCount(commentsTask.id) }}):</strong>
                  <div class="subtasks-list">
                    <div v-for="child in getChildrenTasks(commentsTask.id)" :key="child.id" class="subtask-item">
                      <button @click="highlightTask(child.id)" class="link-button">
                        {{ child.name }} ({{ getStatusLabel(child.status) }})
                      </button>
                    </div>
                  </div>
                </div>
                <div class="view-field">
                  <strong>Creada:</strong> {{ formatDateTime(commentsTask?.createdAt) }}
                </div>
                <div v-if="commentsTask?.updatedAt" class="view-field">
                  <strong>Actualizada:</strong> {{ formatDateTime(commentsTask.updatedAt) }}
                </div>
                <div class="task-actions-row">
                  <button @click="editTask(commentsTask)" class="action-button edit-button">
                    ✏️ Editar Tarea
                  </button>
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>

     
      <!-- Modal de confirmación de eliminación -->
      <div v-if="showDeleteModal" class="modal-overlay" @click="cancelDelete">
        <div class="modal-content delete-modal" @click.stop>
          <div class="modal-header">
            <h2 class="modal-title">{{ deleteModalTitle }}</h2>
            <button @click="cancelDelete" class="close-button-red">×</button>
          </div>
          <div class="delete-modal-content">
            <div class="delete-icon">⚠️</div>
            <p class="delete-message">{{ deleteModalMessage }}</p>
            <div v-if="taskToDelete && isParentTask(taskToDelete.id)" class="warning-box">
              <p><strong>⚠️ Atención:</strong> Esta tarea tiene {{ getChildrenCount(taskToDelete.id) }} subtarea(s).</p>
              <p>Si eliminas esta tarea padre, también se eliminarán todas sus subtareas.</p>
            </div>
          </div>
          <div class="modal-actions">
            <button @click="cancelDelete" class="cancel-button">
              Cancelar
            </button>
            <button @click="confirmDelete" class="delete-button">
              {{ deleteButtonText }}
            </button>
          </div>
        </div>
      </div>

      <!-- Menu contextual -->
      <div v-if="showContextMenu" :style="{ top: contextMenuY + 'px', left: contextMenuX + 'px' }" class="context-menu"
        @click.stop>
        <button @click="editTask(contextMenuTask)" class="context-menu-item">
          ✏️ Editar
        </button>
        <button @click="showDeleteConfirmation(contextMenuTask)" class="context-menu-item delete">
          🗑️ Eliminar
        </button>
      </div>

      <!-- Columnas del Kanban -->
      <div class="columns-container">
        <!-- Columnas dinámicas basadas en columnTypes -->
        <div v-for="(column, index) in columnTypes" :key="index" class="column">
          <div class="column-header" :class="`${column.id}-header`" :style="{ backgroundColor: column.color, color: getContrastColor(column.color) }">
            <h2 class="column-title">
              <div class="status-dot" :class="`${column.id}-dot`"></div>
              {{ column.name }}
            </h2>
          </div>
          <div 
            @drop="onDrop($event, column.id)" 
            @dragover.prevent 
            @dragenter.prevent 
            class="column-content"
            :class="{ 'drag-over': dragOverColumn === column.id }"
            :style="{ borderTopColor: column.color }"
          >
            <div 
              v-for="task in getFilteredTasksByStatus(column.id)" 
              :key="task.id" 
              :draggable="true"
              @dragstart="onDragStart($event, task)" 
              @dragend="onDragEnd" 
              @dblclick="viewTask(task)"
              @contextmenu.prevent="showContextMenuFor(task, $event)" 
              class="task-card" 
              :class="[
                `${column.id}-card`,
                { 
                  'parent-task': isParentTask(task.id),
                  'child-task': task.parentId,
                  'highlighted': highlightedTaskId === task.id
                }
              ]"
            >
              <div class="task-header">
                <h3 class="task-title">{{ task.name }}</h3>
                <div class="task-actions">
                  <button class="comments-button"
                    :title="`${getTaskComments(task.id).length} comentarios`">
                    💬 {{ getTaskComments(task.id).length }}
                  </button>
                  <div class="task-badges">
                    <div v-if="isParentTask(task.id)" class="parent-badge">Padre</div>
                    <div v-if="task.parentId" class="child-badge">Subtarea</div>
                  </div>
                </div>
              </div>
              <div class="task-meta">
                <span class="task-id">ID: {{ task.id }}</span>
                <span class="task-product">
                  {{ task.products && task.products.length > 0 ? task.products.join(', ') : (task.product || 'Sin producto') }}
                </span>
                <span class="priority-badge" :class="getPriorityClass(task.priority)">
                  {{ getPriorityLabel(task.priority) }}
                </span>
              </div>
              <div v-if="task.description" class="task-description">{{ task.description }}</div>
              <div v-if="task.jiraLink" class="jira-link-container">
                <a :href="task.jiraLink" target="_blank" rel="noopener noreferrer" class="jira-link-card" @click.stop>
                  🔗 JIRA
                </a>
              </div>
              <div v-if="task.parentId" class="parent-info">
                Padre:
                <button @click.stop="highlightTask(task.parentId)" class="link-button">
                  {{ getTaskById(task.parentId)?.name || 'Tarea no encontrada' }}
                </button>
              </div>
              <div v-if="isParentTask(task.id)" class="children-info">
                Subtareas: {{ getChildrenCount(task.id) }}
                <div class="children-links">
                  <button v-for="child in getChildrenTasks(task.id).slice(0, 2)" :key="child.id"
                    @click.stop="highlightTask(child.id)" class="link-button child-link">
                    {{ child.name }}
                  </button>
                  <span v-if="getChildrenTasks(task.id).length > 2">
                    +{{ getChildrenTasks(task.id).length - 2 }} más
                  </span>
                </div>
              </div>
              <div class="task-info">
                <span>{{ formatDate(task.createdAt) }}</span>
              </div>
            </div>
            <div v-if="getFilteredTasksByStatus(column.id).length === 0" class="empty-column">
              No hay tareas en {{ column.name.toLowerCase() }}
            </div>
          </div>

        </div>
      </div>

      <!-- Componente TreeNode como template separado -->
      <TreeNodeComponent v-if="false" :task="null" :level="0" />
    </div>
  </div>

</template>
<script setup>
import { ref, onMounted, watch, computed, defineComponent } from 'vue';
import KanbanHeader from './components/KanbanHeader.vue';
// Function to calculate contrast color (black or white) based on background color
const getContrastColor = (hexColor) => {
  if (!hexColor) return '#000000'; // Default to black if no color is provided
  const hex = hexColor.replace('#', '');
  const r = parseInt(hex.substring(0, 2), 16);
  const g = parseInt(hex.substring(2, 4), 16);
  const b = parseInt(hex.substring(4, 6), 16);
  const luminance = (0.2126 * r + 0.7152 * g + 0.0722 * b) / 255;
  return luminance > 0.5 ? '#000000' : '#ffffff';
};

// Definir TreeNodeComponent como un componente separado
const TreeNodeComponent = defineComponent({
  name: 'TreeNodeComponent',
  props: {
    task: {
      type: Object,
      required: true
    },
    level: {
      type: Number,
      default: 0
    }
  },
  template: `
    <div class="tree-node" :style="{ marginLeft: level * 40 + 'px' }">
      <div class="tree-task-card" :class="'status-' + task.status">
        <div class="tree-task-header">
          <h4 class="tree-task-title">{{ task.name }}</h4>
          <div class="tree-task-actions">
            <button @click="viewTaskFromTree(task)" class="tree-action-btn">👁️</button>
            <button @click="highlightTaskFromTree(task.id)" class="tree-action-btn">📍</button>
          </div>
        </div>
        <div class="tree-task-meta">
          <span class="task-id">ID: {{ task.id }}</span>
          <span class="task-product">{{ getTaskProducts(task) }}</span>
          <span class="priority-badge" :class="getPriorityClassFromTree(task.priority)">
            {{ getPriorityLabelFromTree(task.priority) }}
          </span>
          <span class="status-badge" :class="'status-' + task.status">
            {{ getStatusLabelFromTree(task.status) }}
          </span>
        </div>
        <div v-if="task.description" class="tree-task-description">
          {{ task.description }}
        </div>
      </div>
      
      <!-- Líneas de conexión y subtareas -->
      <div v-if="task.children && task.children.length > 0" class="tree-children">
        <div class="tree-line"></div>
        <TreeNodeComponent 
          v-for="child in task.children" 
          :key="child.id"
          :task="child" 
          :level="level + 1"
        />
      </div>
    </div>
  `,
  methods: {
    viewTaskFromTree(task) {
      this.$parent.viewTask(task)
    },
    highlightTaskFromTree(taskId) {
      this.$parent.highlightTask(taskId)
    },
    getPriorityClassFromTree(priority) {
      return this.$parent.getPriorityClass(priority)
    },
    getPriorityLabelFromTree(priority) {
      return this.$parent.getPriorityLabel(priority)
    },
    getStatusLabelFromTree(status) {
      return this.$parent.getStatusLabel(status)
    },
    getTaskProducts(task) {
      if (task.products && task.products.length > 0) {
        return task.products.join(', ')
      }
      return task.product || 'Sin producto'
    }
  }
})

// Estado para el arrastre de columnas
const draggedColumnIndex = ref(null);
const dragOverColumnIndex = ref(null);

// Estado reactivo
const boardTitle = ref('Tablero Kanban')
const tasks = ref([])
const comments = ref([])

// Computed property para combinar tareas con sus comentarios
const enhancedTasksWithComments = computed(() => {
  return tasks.value.map(task => {
    const taskComments = comments.value.filter(comment => comment.taskId === task.id)
    return {
      ...task,
      comments: taskComments
    }
  })
})


const draggedTask = ref(null)
const dragOverColumn = ref(null)

// Estado del modal
const showModal = ref(false)
const editingTask = ref(null)
const newTask = ref({
  name: '',
  description: '',
  products: [], // Cambiar de product a products (array)
  priority: '',
  jiraLink: '',
  parentId: null,
  startDate: null,
  endDate: null
})

// Estado del modal configuración
const showConfigModal = ref(false)
const columnTypes = ref([
])
const newColumnType = ref({ name: '', color: '#CCCCCC' })

// Estado del modal JSON
const showJsonModal = ref(false)
const jsonCopied = ref(false)
const activeSettingsTab = ref('columns') // 'columns' or 'data' for tab navigation

// Estado del modal de eliminación
const showDeleteModal = ref(false)
const taskToDelete = ref(null)

// Estado del modal de comentarios
const showCommentsModal = ref(false)
const commentsTask = ref(null)
const newComment = ref({
  text: '',
  matrixValue: 'not-important-not-urgent' // Valor por defecto
})

// Estado del buscador de tareas padre
const parentSearchTerm = ref('')
const showParentDropdown = ref(false)
const filteredParentTasks = ref([])

// Estado del buscador general
const searchTerm = ref('')
//const filteredTasks = ref([])

// Estado de filtros de producto
const selectedProducts = ref([])
const showProductDropdown = ref(false)

// Estado de filtros de prioridad
const selectedPriorities = ref([])
const showPriorityDropdown = ref(false)

// Estado del menú contextual
const showContextMenu = ref(false)
const contextMenuTask = ref(null)
const contextMenuX = ref(0)
const contextMenuY = ref(0)

// Estado de resaltado
const highlightedTaskId = ref(null)
const fileInput = ref(null)
let highlightTimer = null

function triggerFileInput() {
  fileInput.value.click()
}

async function importData(event) {
  const file = event.target.files[0]
  if (!file) return

  try {
    const fileContent = await readFileAsText(file)
    const data = JSON.parse(fileContent)

    // Verificar si el archivo tiene el formato antiguo (solo array de tareas)
    if (Array.isArray(data)) {
      // Formato antiguo: solo array de tareas
      const importedComments = []
      const tasksWithoutComments = data.map(task => {
        const { comments = [], ...taskWithoutComments } = task
        if (Array.isArray(comments)) {
          importedComments.push(...comments)
        }
        return taskWithoutComments
      })

      tasks.value = tasksWithoutComments
      comments.value = importedComments
    } else if (data.tasks && Array.isArray(data.tasks)) {
      // Nuevo formato: objeto con settings, tasks y comments
      const importedComments = []
      const tasksWithoutComments = data.tasks.map(task => {
        const { comments = [], ...taskWithoutComments } = task
        if (Array.isArray(comments)) {
          importedComments.push(...comments)
        }
        return taskWithoutComments
      })

      tasks.value = tasksWithoutComments
      comments.value = data.comments || []
      
      // Aplicar configuración si existe
      if (data.settings) {
        if (data.settings.title_board) {
          boardTitle.value = data.settings.title_board
        }
        // Importar configuración de columnas si existe
        if (data.settings.columnTypes && Array.isArray(data.settings.columnTypes)) {
          columnTypes.value = data.settings.columnTypes
        }
        saveSettings()
      }
    } else if (data.settings && data.settings.columnTypes && data.type === 'kanban-column-config') {
      // Solo configuración de columnas
      if (confirm('¿Importar configuración de columnas? Esto sobrescribirá la configuración actual.')) {
        columnTypes.value = data.settings.columnTypes
        if (data.settings.title) {
          boardTitle.value = data.settings.title
        }
        saveSettings()
      } else {
        return // Cancelar la importación si el usuario no confirma
      }
    } else {
      throw new Error('El archivo no contiene un formato válido')
    }

    // Guardar en localStorage
    saveTasks()
    saveComments()
    
    // Resetear el input para permitir cargar el mismo archivo de nuevo
    event.target.value = null

    // Mostrar notificación de éxito
    const notification = document.createElement('div')
    notification.className = 'import-notification success'
    notification.textContent = 'Datos importados correctamente'
    document.body.appendChild(notification)
    
    // Eliminar la notificación después de 3 segundos
    setTimeout(() => {
      notification.classList.add('fade-out')
      setTimeout(() => notification.remove(), 300)
    }, 3000)
    
  } catch (error) {
    console.error('Error importing data:', error)
    
    // Mostrar notificación de error
    const notification = document.createElement('div')
    notification.className = 'import-notification error'
    notification.textContent = 'Error al importar: ' + (error.message || 'Formato de archivo no válido')
    document.body.appendChild(notification)
    
    // Eliminar la notificación después de 5 segundos
    setTimeout(() => {
      notification.classList.add('fade-out')
      setTimeout(() => notification.remove(), 300)
    }, 5000)
  }
}

function readFileAsText(file) {
  return new Promise((resolve, reject) => {
    const reader = new FileReader()
    reader.onload = e => resolve(e.target.result)
    reader.onerror = error => reject(error)
    reader.readAsText(file)
  })
}

// Function to export data as JSON file
const exportData = () => {
  try {
    const dataToExport = {
      settings: {
        title_board: boardTitle.value,
        columnTypes: columnTypes.value
      },
      tasks: tasks.value,
      comments: comments.value,
      version: '1.0.0'
    }
    
    const dataStr = JSON.stringify(dataToExport, null, 2)
    const blob = new Blob([dataStr], { type: 'application/json' })
    const url = URL.createObjectURL(blob)
    
    const exportFileDefaultName = `tareas_${new Date().toISOString().split('T')[0]}.json`
    
    const link = document.createElement('a')
    link.href = url
    link.download = exportFileDefaultName
    document.body.appendChild(link)
    link.click()
    document.body.removeChild(link)
    URL.revokeObjectURL(url)
  } catch (error) {
    console.error('Error exporting data:', error)
    alert('Error al exportar los datos. Por favor, inténtalo de nuevo.')
  }
}

// Estado para autocompletado de productos
const productSearchTerm = ref('')
const showProductSuggestions = ref(false)
const filteredProductSuggestions = ref([])

// Validación de formulario
const isFormValid = computed(() => {
  if (newTask.value.jiraLink != "" && !isValidUrl(newTask.value.jiraLink)) {
    return false
  }
  return (
    newTask.value.name.trim() &&
    newTask.value.products.length > 0 &&
    newTask.value.priority
  )
})

// Prioridades disponibles
const availablePriorities = [
  { value: 'muy-bajo', label: 'Muy Bajo', class: 'priority-muy-bajo' },
  { value: 'bajo', label: 'Bajo', class: 'priority-bajo' },
  { value: 'medio', label: 'Medio', class: 'priority-medio' },
  { value: 'alto', label: 'Alto', class: 'priority-alto' },
  { value: 'muy-alto', label: 'Muy Alto', class: 'priority-muy-alto' }
]

// Computed para productos disponibles
const availableProducts = computed(() => {
  const products = [...new Set(tasks.value.map(task => task.products).flat().filter(Boolean))]
  return products.sort()
})

// Computed para el modal de eliminación
const deleteModalTitle = computed(() => {
  if (!taskToDelete.value) return ''
  return isParentTask(taskToDelete.value.id) ? 'Eliminar Tarea Padre' : 'Eliminar Tarea'
})

const deleteModalMessage = computed(() => {
  if (!taskToDelete.value) return ''
  return `¿Estás seguro de que quieres eliminar la tarea "${taskToDelete.value.name}"?`
})

const deleteButtonText = computed(() => {
  if (!taskToDelete.value) return 'Eliminar'
  return isParentTask(taskToDelete.value.id) ? 'Eliminar Todo' : 'Eliminar'
})

// Función para validar URL
function isValidUrl(string) {
  if (!string) return true // Campo opcional
  try {
    new URL(string)
    return true
  } catch (_) {
    return false
  }
}

// Funciones de prioridad
function getPriorityLabel(priority) {
  const priorityObj = availablePriorities.find(p => p.value === priority)
  return priorityObj ? priorityObj.label : priority
}

function getPriorityClass(priority) {
  const priorityObj = availablePriorities.find(p => p.value === priority)
  return priorityObj ? priorityObj.class : ''
}


// Funciones de comentarios con Matriz de Eisenhower
function getTaskComments(taskId) {
  return comments.value
    .filter(comment => comment.taskId === taskId)
    .sort((a, b) => new Date(b.createdAt) - new Date(a.createdAt))
}

function addComment() {
  if (!newComment.value.text.trim() || !commentsTask.value || !newComment.value.matrixValue) return

  // Convertir el valor de la matriz a booleanos
  const matrixMap = {
    'important-urgent': { important: true, urgent: true },
    'important-not-urgent': { important: true, urgent: false },
    'not-important-urgent': { important: false, urgent: true },
    'not-important-not-urgent': { important: false, urgent: false }
  }

  const { important, urgent } = matrixMap[newComment.value.matrixValue]

  const comment = {
    id: Date.now(),
    taskId: commentsTask.value.id,
    text: newComment.value.text.trim(),
    important: important,
    urgent: urgent,
    isCompleted: false, // Nuevo campo
    createdAt: new Date().toISOString()
  }

  comments.value.push(comment)
  newComment.value = {
    text: '',
    matrixValue: 'not-important-not-urgent'
  }
}

// Nuevas funciones para gestionar comentarios
function toggleCommentCompleted(commentId) {
  const comment = comments.value.find(c => c.id === commentId)
  if (comment) {
    comment.isCompleted = !comment.isCompleted
    comment.completedAt = comment.isCompleted ? new Date().toISOString() : null
  }
}

function deleteComment(commentId) {
  const commentIndex = comments.value.findIndex(c => c.id === commentId)
  if (commentIndex !== -1) {
    comments.value.splice(commentIndex, 1)
  }
}

function getCommentPriorityLabel(important, urgent) {
  if (important && urgent) return 'Importante y Urgente'
  if (important && !urgent) return 'Importante y No urgente'
  if (!important && urgent) return 'No importante y Urgente'
  return 'No importante y No urgente'
}

function getCommentPriorityClass(important, urgent) {
  if (important && urgent) return 'comment-priority-important-urgent'
  if (important && !urgent) return 'comment-priority-important-not-urgent'
  if (!important && urgent) return 'comment-priority-not-important-urgent'
  return 'comment-priority-not-important-not-urgent'
}

function getCommentMatrixClass(important, urgent) {
  if (important && urgent) return 'matrix-important-urgent'
  if (important && !urgent) return 'matrix-important-not-urgent'
  if (!important && urgent) return 'matrix-not-important-urgent'
  return 'matrix-not-important-not-urgent'
}

// Función para formatear fecha y hora
function formatDateTime(dateString) {
  const date = new Date(dateString)
  return date.toLocaleString('es-ES', {
    day: '2-digit',
    month: '2-digit',
    year: 'numeric',
    hour: '2-digit',
    minute: '2-digit'
  })
}

// Funciones para productos múltiples
function filterProducts() {
  const searchTerm = productSearchTerm.value?.toLowerCase()?.trim() || ''

  if (!searchTerm) {
    filteredProductSuggestions.value = availableProducts.value.slice(0, 5)
    return
  }

  filteredProductSuggestions.value = availableProducts.value.filter(product =>
    product.toLowerCase().includes(searchTerm) &&
    !newTask.value.products.includes(product)
  ).slice(0, 5)
}

function addProduct(product) {
  if (!newTask.value.products.includes(product)) {
    newTask.value.products.push(product)
  }
  productSearchTerm.value = ''
  showProductSuggestions.value = false
  filteredProductSuggestions.value = []
}

function addCurrentProduct() {
  const product = productSearchTerm.value.trim().toUpperCase()
  if (product && !newTask.value.products.includes(product)) {
    newTask.value.products.push(product)
    productSearchTerm.value = ''
    showProductSuggestions.value = false
    filteredProductSuggestions.value = []
  }
}

function removeProduct(product) {
  const index = newTask.value.products.indexOf(product)
  if (index > -1) {
    newTask.value.products.splice(index, 1)
  }
}

// Función para copiar JSON al portapapeles
async function copyJsonToClipboard() {
  try {
    const data = {
      tasks: tasks.value,
      comments: comments.value
    }
    await navigator.clipboard.writeText(JSON.stringify(data, null, 2))
    jsonCopied.value = true
    setTimeout(() => {
      jsonCopied.value = false
    }, 2000)
  } catch (err) {
    console.error('Error al copiar al portapapeles:', err)
    // Fallback para navegadores que no soportan clipboard API
    const textArea = document.createElement('textarea')
    const data = {
      tasks: tasks.value,
      comments: comments.value
    }
    textArea.value = JSON.stringify(data, null, 2)
    document.body.appendChild(textArea)
    textArea.select()
    document.execCommand('copy')
    document.body.removeChild(textArea)
    jsonCopied.value = true
    setTimeout(() => {
      jsonCopied.value = false
    }, 2000)
  }
}

// Funciones del modal de eliminación
function showDeleteConfirmation(task) {
  taskToDelete.value = task
  showDeleteModal.value = true
  showContextMenu.value = false
}

function cancelDelete() {
  showDeleteModal.value = false
  taskToDelete.value = null
}

function confirmDelete() {
  if (!taskToDelete.value) return

  const taskId = taskToDelete.value.id

  // Eliminar la tarea
  tasks.value = tasks.value.filter(t => t.id !== taskId)

  // Eliminar comentarios de la tarea
  comments.value = comments.value.filter(c => c.taskId !== taskId)

  // Si era una tarea padre, eliminar también las subtareas y sus comentarios
  if (isParentTask(taskId)) {
    const childTasks = tasks.value.filter(t => t.parentId === taskId)
    childTasks.forEach(child => {
      comments.value = comments.value.filter(c => c.taskId !== child.id)
    })
    tasks.value = tasks.value.filter(t => t.parentId !== taskId)
  }

  // Limpiar referencias padre de las subtareas (por si acaso)
  tasks.value.forEach(t => {
    if (t.parentId === taskId) {
      t.parentId = null
    }
  })

  showDeleteModal.value = false
  taskToDelete.value = null
}

// Cargar configuración
function loadSettings() {
  const savedSettings = localStorage.getItem('boardSettings')
  if (savedSettings) {
    try {
      const settings = JSON.parse(savedSettings)
      if (settings.title_board) {
        boardTitle.value = settings.title_board
      }
    } catch (e) {
      console.error('Error loading settings:', e)
    }
  }
}

// Guardar configuración
const saveSettings = () => {
  localStorage.setItem('kanbanSettings', JSON.stringify({
    title: boardTitle.value,
    columnTypes: columnTypes.value
  }))
}

// Cargar configuración de columnas
const loadColumnTypes = () => {
  const savedSettings = localStorage.getItem('kanbanSettings')
  if (savedSettings) {
    try {
      const settings = JSON.parse(savedSettings)
      if (settings.columnTypes) {
        columnTypes.value = settings.columnTypes
      }
    } catch (e) {
      console.error('Error al cargar configuración de columnas:', e)
    }
  }
}

// Agregar nuevo tipo de columna
const addColumnType = () => {
  if (newColumnType.value.name.trim()) {
    columnTypes.value.push({
      id: newColumnType.value.name.toLowerCase().replace(/\s+/g, '-'),
      name: newColumnType.value.name.trim(),
      color: newColumnType.value.color
    })
    newColumnType.value = { name: '', color: '#CCCCCC' }
  }
}

// Funciones para el reordenamiento de columnas en configuración
function onColumnTypeDragStart(event, index) {
  event.stopPropagation();
  event.dataTransfer.effectAllowed = 'move';
  event.dataTransfer.setData('text/plain', index);
  draggedColumnIndex.value = index;
  event.currentTarget.classList.add('dragging');
}

function onColumnTypeDragOver(event, index) {
  event.preventDefault();
  if (draggedColumnIndex.value === null) return;
  dragOverColumnIndex.value = index;
  
  // Actualizar clases visuales
  const items = document.querySelectorAll('.column-type-item');
  items.forEach((item, i) => {
    item.classList.toggle('drag-over', i === index);
  });
}

function onColumnTypeDrop(event) {
  event.preventDefault();
  
  if (draggedColumnIndex.value === null || dragOverColumnIndex.value === null) return;
  if (draggedColumnIndex.value === dragOverColumnIndex.value) return;
  
  // Prevenir cualquier acción adicional que pueda cerrar el menú
  event.stopImmediatePropagation();
  
  // Reordenar los tipos de columna
  const newColumnTypes = [...columnTypes.value];
  const [movedColumn] = newColumnTypes.splice(draggedColumnIndex.value, 1);
  newColumnTypes.splice(dragOverColumnIndex.value, 0, movedColumn);
  
  columnTypes.value = newColumnTypes;
  saveColumnTypes();
  
  // Resetear estados
  onColumnTypeDragEnd();
}

function onColumnTypeDragEnd() {
  // Remover clases de arrastre
  document.querySelectorAll('.column-type-item').forEach(el => {
    el.classList.remove('dragging', 'drag-over');
  });
  
  draggedColumnIndex.value = null;
  dragOverColumnIndex.value = null;
}

// Eliminar tipo de columna
function removeColumnType(index) {
  const column = columnTypes.value[index];
  
  // Verificar si hay tareas en esta columna
  const tasksInColumn = tasks.value.filter(task => task.status === column.id);
  
  if (tasksInColumn.length > 0) {
    showNotification(`No se puede eliminar la columna "${column.name}" porque tiene ${tasksInColumn.length} tarea(s) asignada(s).`, 'error');
    return;
  }
  
  if (columnTypes.value.length > 1) {
    columnTypes.value.splice(index, 1);
    saveColumnTypes();
    showNotification('Columna eliminada correctamente');
  } else {
    showNotification('Debe haber al menos una columna', 'error');
  }
}

// Guardar tipos de columna
const saveColumnTypes = () => {
  saveSettings()
  showConfigModal.value = false
}



// Función para mostrar notificaciones
function showNotification(message, type = 'success') {
  const notification = document.createElement('div')
  notification.className = `import-notification ${type}`
  notification.textContent = message
  document.body.appendChild(notification)
  
  // Eliminar la notificación después de 3 segundos
  setTimeout(() => {
    notification.classList.add('fade-out')
    setTimeout(() => notification.remove(), 300)
  }, 3000)
}

// Cargar tareas y configuración desde localStorage al montar el componente
onMounted(() => {
  loadSettings()
  loadTasks()
  loadColumnTypes()
  loadComments()
  // Cerrar dropdowns y menús al hacer click fuera
  document.addEventListener('click', (e) => {
    if (!e.target.closest('.select-container')) {
      showParentDropdown.value = false
    }
    if (!e.target.closest('.multiselect-container')) {
      showProductDropdown.value = false
      showPriorityDropdown.value = false
    }
    if (!e.target.closest('.context-menu')) {
      showContextMenu.value = false
    }
    if (!e.target.closest('.product-autocomplete')) {
      showProductSuggestions.value = false
    }
  })
})

// Observar cambios en las tareas para guardar automáticamente
watch(tasks, () => {
  saveTasks()
}, { deep: true })

// Observar cambios en los comentarios para guardar automáticamente
watch(comments, () => {
  saveComments()
}, { deep: true })

// Función para cargar tareas desde localStorage
function loadTasks() {
  try {
    const savedTasks = localStorage.getItem('kanban-tasks')
    if (savedTasks) {
      tasks.value = JSON.parse(savedTasks)
      // Migrar tareas antiguas con product a products
      tasks.value.forEach(task => {
        if (task.product && !task.products) {
          task.products = [task.product]
          delete task.product
        }
        if (!task.products) {
          task.products = []
        }
      })
    } else {
      // Tareas de ejemplo si no hay datos guardados
      tasks.value = []
    }
  } catch (error) {
    console.error('Error al cargar las tareas:', error)
    tasks.value = []
  }
}

function loadComments() {
  try {
    const savedComments = localStorage.getItem('kanban-comments')
    if (savedComments) {
      comments.value = JSON.parse(savedComments)
    } else {
      // Comentarios de ejemplo con Matriz de Eisenhower
      comments.value = [
        {
          id: 1,
          taskId: 1,
          text: 'Necesitamos revisar los requisitos de seguridad urgentemente',
          important: true,
          urgent: true,
          isCompleted: false,
          createdAt: new Date(Date.now() - 86400000).toISOString() // Ayer
        },
        {
          id: 2,
          taskId: 1,
          text: 'Ya está listo el diseño de la interfaz, podemos planificar la implementación',
          important: true,
          urgent: false,
          isCompleted: true,
          completedAt: new Date(Date.now() - 3600000).toISOString(), // Hace 1 hora
          createdAt: new Date().toISOString()
        },
        {
          id: 3,
          taskId: 2,
          text: 'Hay que responder al cliente sobre el timeline',
          important: false,
          urgent: true,
          isCompleted: false,
          createdAt: new Date().toISOString()
        }
      ]
    }
  } catch (error) {
    console.error('Error al cargar los comentarios:', error)
    comments.value = []
  }
}

// Función para guardar tareas en localStorage
function saveTasks() {
  try {
    localStorage.setItem('kanban-tasks', JSON.stringify(tasks.value))
  } catch (error) {
    console.error('Error al guardar las tareas:', error)
  }
}

// Función para guardar comentarios en localStorage
function saveComments() {
  try {
    localStorage.setItem('kanban-comments', JSON.stringify(comments.value))
  } catch (error) {
    console.error('Error al guardar los comentarios:', error)
  }
}

// Funciones del modal
function openModal() {
  editingTask.value = null
  showModal.value = true
  resetForm()
}

function closeModal() {
  showModal.value = false
  showParentDropdown.value = false
  showProductSuggestions.value = false
  editingTask.value = null
  resetForm()
}

function resetForm() {
  newTask.value = {
    name: '',
    description: '',
    products: [],
    priority: '',
    jiraLink: '',
    parentId: null
  }
  parentSearchTerm.value = ''
  productSearchTerm.value = ''
  filteredParentTasks.value = []
  filteredProductSuggestions.value = []
}

// Función para añadir/editar tarea
function saveTask() {
  if (!isFormValid.value) return

  if (editingTask.value) {
    // Editar tarea existente
    const taskIndex = tasks.value.findIndex(task => task.id === editingTask.value.id)
    if (taskIndex !== -1) {
      tasks.value[taskIndex] = {
        ...tasks.value[taskIndex],
        name: newTask.value.name.trim(),
        description: newTask.value.description.trim(),
        products: newTask.value.products.map(p => p.toUpperCase()),
        priority: newTask.value.priority,
        jiraLink: newTask.value.jiraLink.trim(),
        parentId: newTask.value.parentId,
        updatedAt: new Date().toISOString()
      }
    }
  } else {
    // Crear nueva tarea
    const task = {
      id: Date.now(),
      name: newTask.value.name.trim(),
      description: newTask.value.description.trim(),
      products: newTask.value.products.map(p => p.toUpperCase()),
      priority: newTask.value.priority,
      jiraLink: newTask.value.jiraLink.trim(),
      status: 'backlog',
      parentId: newTask.value.parentId,
      createdAt: new Date().toISOString()
    }
    tasks.value.push(task)
  }

  closeModal()
}

// Funciones para el buscador de tareas padre
function filterParentTasks() {
  const searchTerm = parentSearchTerm.value?.toLowerCase()?.trim() || ''

  if (!searchTerm) {
    filteredParentTasks.value = tasks.value
      .filter(task => !editingTask.value || task.id !== editingTask.value.id)
      .slice(0, 10)
    return
  }

  filteredParentTasks.value = tasks.value.filter(task => {
    if (!task || !task.name) return false
    if (editingTask.value && task.id === editingTask.value.id) return false

    const nameMatch = task.name.toLowerCase().includes(searchTerm)
    const idMatch = String(task.id).includes(searchTerm)
    const descriptionMatch = task.description &&
      typeof task.description === 'string' &&
      task.description.toLowerCase().includes(searchTerm)

    return nameMatch || idMatch || descriptionMatch
  }).slice(0, 10)
}

function selectParentTask(task) {
  if (!task || !task.id || !task.name) return

  newTask.value.parentId = task.id
  parentSearchTerm.value = `${task.name} (ID: ${task.id})`
  showParentDropdown.value = false
  filteredParentTasks.value = []
}

function clearParentTask() {
  newTask.value.parentId = null
  parentSearchTerm.value = ''
  filteredParentTasks.value = []
}

// Funciones de búsqueda y filtrado
function handleSearch() {
  // La búsqueda se aplica en getFilteredTasksByStatus
  console.log("Buscando:", searchTerm.value)
}



function getFilteredTasksByStatus(status) {
  return tasks.value.filter(task => {
    // Primero verificamos el estado
    if (task.status !== status) return false

    // Luego aplicamos el filtro de búsqueda si existe
    if (searchTerm.value) {
      const search = searchTerm.value.toLowerCase().trim()
      const nameMatch = task.name.toLowerCase().includes(search)
      const idMatch = String(task.id).includes(search)

      if (!nameMatch && !idMatch) return false
    }

    // Aplicamos el filtro de productos
    if (selectedProducts.value.length > 0) {
      const hasMatchingProduct = task.products && task.products.some(product =>
        selectedProducts.value.includes(product)
      )
      if (!hasMatchingProduct) return false
    }

    // Aplicamos el filtro de prioridades
    if (selectedPriorities.value.length > 0) {
      if (!selectedPriorities.value.includes(task.priority)) return false
    }

    return true
  })
}

function closeCommentsModal() {
  showCommentsModal.value = false
  commentsTask.value = null
  newComment.value = {
    text: '',
    matrixValue: 'not-important-not-urgent'
  }
}

// Funciones del menú contextual
function showContextMenuFor(task, event) {
  contextMenuTask.value = task
  contextMenuX.value = event.clientX
  contextMenuY.value = event.clientY
  showContextMenu.value = true
}

function editTask(task) {
  editingTask.value = task
  newTask.value = {
    name: task.name,
    description: task.description || '',
    products: task.products || [],
    priority: task.priority || '',
    jiraLink: task.jiraLink || '',
    parentId: task.parentId
  }
  if (task.parentId) {
    const parentTask = getTaskById(task.parentId)
    parentSearchTerm.value = parentTask ? `${parentTask.name} (ID: ${parentTask.id})` : ''
  }
  showModal.value = true
  showContextMenu.value = false
}

// Función para ver tarea
function viewTask(task) {
  commentsTask.value = task
  showCommentsModal.value = true
}

// Función para resaltar tarea
function highlightTask(taskId) {
  // Limpiar el temporizador anterior si existe
  if (highlightTimer) {
    clearTimeout(highlightTimer)
  }

  // Establecer la tarea resaltada
  highlightedTaskId.value = taskId

  // Configurar un nuevo temporizador de 5 segundos
  highlightTimer = setTimeout(() => {
    highlightedTaskId.value = null
    highlightTimer = null
  }, 5000)
}

// Funciones auxiliares
function truncateText(text, maxLength) {
  if (!text) return ''
  return text.length > maxLength ? text.substring(0, maxLength) + '...' : text
}

function getTaskById(id) {
  if (!id || !tasks.value || !Array.isArray(tasks.value)) return null
  return tasks.value.find(task => task && task.id === id) || null
}

function isParentTask(taskId) {
  return tasks.value.some(task => task.parentId === taskId)
}

function getChildrenCount(parentId) {
  return tasks.value.filter(task => task.parentId === parentId).length
}

function getChildrenTasks(parentId) {
  return tasks.value.filter(task => task.parentId === parentId)
}

function getStatusLabel(status) {
  const labels = {
    'backlog': 'Backlog',
    'paralyzed': 'Paralizada',
    'in-progress': 'En Progreso',
    'testing': 'Testing',
    'finalized': 'Finalizada'
  }
  return labels[status] || status
}

// Función para formatear fecha
function formatDate(dateString) {
  const date = new Date(dateString)
  return date.toLocaleDateString('es-ES', {
    day: '2-digit',
    month: '2-digit',
    year: 'numeric'
  })
}

// Funciones para drag and drop
function onDragStart(event, task) {
  draggedTask.value = task
  event.dataTransfer.effectAllowed = 'move'
  dragOverColumn.value = null
}

function onDragEnd() {
  draggedTask.value = null
  dragOverColumn.value = null
}

function onDrop(event, newStatus) {
  event.preventDefault()
  dragOverColumn.value = null

  if (draggedTask.value && draggedTask.value.status !== newStatus) {
    const taskIndex = tasks.value.findIndex(task => task.id === draggedTask.value.id)
    if (taskIndex !== -1) {
      tasks.value[taskIndex].status = newStatus
      tasks.value[taskIndex].updatedAt = new Date().toISOString()
    }
  }

  draggedTask.value = null
}
</script>



<style>
@import './assets/css/style.css';
</style>