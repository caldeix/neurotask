<template>
  <div class="kanban-container">
    <div class="kanban-wrapper">
      <!-- Header con buscador y filtros -->
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
            <button @click="showJsonModal = true" class="json-button">
              VER DATOS
            </button>
            <button @click="exportData" class="export-button">
              EXPORTAR DATOS
            </button>
            <button @click="triggerFileInput" class="import-button">
              IMPORTAR DATOS
            </button>
            <input type="file" ref="fileInput" @change="importData" accept=".json" style="display: none" />
          </div>
        </div>

        <!-- Buscador y filtros -->
        <div class="search-filters-section">
          <div class="search-container">
            <input v-model="searchTerm" @input="handleSearch" type="text" placeholder="Buscar por título o ID..."
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

          <button @click="openModal" class="add-button">
            + Añadir Tarea
          </button>
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
              settings: {
                title_board: boardTitle.value
              },
              tasks: enhancedTasksWithComments,
              comments: comments.value
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

      <!-- Modal de visualización -->
      <div v-if="showViewModal" class="modal-overlay" @click="showViewModal = false">
        <div class="modal-content" @click.stop>
          <div class="modal-header">
            <h2 class="modal-title">#{{ viewingTask?.id }} - {{ viewingTask?.name }} 
              <span class="priority-badge" :class="getPriorityClass(viewingTask?.priority)">
                {{ getPriorityLabel(viewingTask?.priority) }}
              </span>
              <span class="priority-badge" :class="getStatusClass(viewingTask?.status)">
                {{ getStatusLabel(viewingTask?.status) }}
              </span>
            </h2>
            <button @click="showViewModal = false" class="close-button-red">×</button>
          </div>
          <div class="view-modal-content">
            <div class="view-field">
              <strong>Estado:</strong> {{ getStatusLabel(viewingTask?.status) }}
            </div>
            <div class="view-field">
              <strong>Productos:</strong>
              <span v-if="viewingTask?.products && viewingTask.products.length > 0">
                {{ viewingTask.products.join(', ') }}
              </span>
              <span v-else>{{ viewingTask?.product || 'No especificado' }}</span>
            </div>
            <div v-if="viewingTask?.description" class="view-field">
              <strong>Descripción:</strong> {{ viewingTask.description }}
            </div>
            <div v-if="viewingTask?.jiraLink" class="view-field">
              <strong>JIRA LINK:</strong>
              <a :href="viewingTask.jiraLink" target="_blank" rel="noopener noreferrer" class="jira-link">
                {{ viewingTask.jiraLink }}
              </a>
            </div>
            <div v-if="viewingTask?.parentId" class="view-field">
              <strong>Tarea Padre:</strong>
              <button @click="highlightTask(viewingTask.parentId)" class="link-button">
                {{ getTaskById(viewingTask.parentId)?.name }} (ID: {{ viewingTask.parentId }})
              </button>
            </div>
            <div v-if="getChildrenTasks(viewingTask?.id).length > 0" class="view-field">
              <strong>Subtareas:</strong>
              <div class="subtasks-list">
                <button v-for="child in getChildrenTasks(viewingTask?.id)" :key="child.id"
                  @click="highlightTask(child.id)" class="link-button subtask-link">
                  {{ child.name }} (ID: {{ child.id }})
                </button>
              </div>
            </div>
            <div class="view-field">
              <strong>Comentarios:</strong> {{ getTaskComments(viewingTask?.id).length }}
            </div>
            <div class="view-field">
              <strong>Creado:</strong> {{ formatDate(viewingTask?.createdAt) }}
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
        <!-- Columna Backlog -->
        <div class="column">
          <div class="column-header backlog-header">
            <h2 class="column-title">
              <div class="status-dot backlog-dot"></div>
              Backlog
              <span class="task-count">({{ getFilteredTasksByStatus('backlog').length }})</span>
            </h2>
          </div>
          <div @drop="onDrop($event, 'backlog')" @dragover.prevent @dragenter.prevent class="column-content"
            :class="{ 'drag-over': dragOverColumn === 'backlog' }">
            <div v-for="task in getFilteredTasksByStatus('backlog')" :key="task.id" :draggable="true"
              @dragstart="onDragStart($event, task)" @dragend="onDragEnd" @dblclick="viewTask(task)"
              @contextmenu.prevent="showContextMenuFor(task, $event)" class="task-card backlog-card" :class="{
                'parent-task': isParentTask(task.id),
                'child-task': task.parentId,
                'highlighted': highlightedTaskId === task.id
              }">
              <div class="task-header">
                <h3 class="task-title">{{ task.name }}</h3>
                <div class="task-actions">
                  <button @click.stop="openCommentsModal(task)" class="comments-button"
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
              <div v-if="task.description" class="task-description">{{ truncateText(task.description, 50) }}</div>
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
            <div v-if="getFilteredTasksByStatus('backlog').length === 0" class="empty-column">
              No hay tareas en backlog
            </div>
          </div>
        </div>

        <!-- Columna Paralyzed -->
        <div class="column">
          <div class="column-header paralyzed-header">
            <h2 class="column-title">
              <div class="status-dot paralyzed-dot"></div>
              Paralyzed
              <span class="task-count">({{ getFilteredTasksByStatus('paralyzed').length }})</span>
            </h2>
          </div>
          <div @drop="onDrop($event, 'paralyzed')" @dragover.prevent @dragenter.prevent class="column-content"
            :class="{ 'drag-over': dragOverColumn === 'paralyzed' }">
            <div v-for="task in getFilteredTasksByStatus('paralyzed')" :key="task.id" :draggable="true"
              @dragstart="onDragStart($event, task)" @dragend="onDragEnd" @dblclick="viewTask(task)"
              @contextmenu.prevent="showContextMenuFor(task, $event)" class="task-card paralyzed-card" :class="{
                'parent-task': isParentTask(task.id),
                'child-task': task.parentId,
                'highlighted': highlightedTaskId === task.id
              }">
              <div class="task-header">
                <h3 class="task-title">{{ task.name }}</h3>
                <div class="task-actions">
                  <button @click.stop="openCommentsModal(task)" class="comments-button"
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
            <div v-if="getFilteredTasksByStatus('paralyzed').length === 0" class="empty-column">
              No hay tareas paralizadas
            </div>
          </div>
        </div>

        <!-- Columna In Progress -->
        <div class="column">
          <div class="column-header progress-header">
            <h2 class="column-title">
              <div class="status-dot progress-dot"></div>
              In Progress
              <span class="task-count">({{ getFilteredTasksByStatus('in-progress').length }})</span>
            </h2>
          </div>
          <div @drop="onDrop($event, 'in-progress')" @dragover.prevent @dragenter.prevent class="column-content"
            :class="{ 'drag-over': dragOverColumn === 'in-progress' }">
            <div v-for="task in getFilteredTasksByStatus('in-progress')" :key="task.id" :draggable="true"
              @dragstart="onDragStart($event, task)" @dragend="onDragEnd" @dblclick="viewTask(task)"
              @contextmenu.prevent="showContextMenuFor(task, $event)" class="task-card progress-card" :class="{
                'parent-task': isParentTask(task.id),
                'child-task': task.parentId,
                'highlighted': highlightedTaskId === task.id
              }">
              <div class="task-header">
                <h3 class="task-title">{{ task.name }}</h3>
                <div class="task-actions">
                  <button @click.stop="openCommentsModal(task)" class="comments-button"
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
            <div v-if="getFilteredTasksByStatus('in-progress').length === 0" class="empty-column">
              No hay tareas en progreso
            </div>
          </div>
        </div>

        <!-- Columna Testing -->
        <div class="column">
          <div class="column-header testing-header">
            <h2 class="column-title">
              <div class="status-dot testing-dot"></div>
              Testing
              <span class="task-count">({{ getFilteredTasksByStatus('testing').length }})</span>
            </h2>
          </div>
          <div @drop="onDrop($event, 'testing')" @dragover.prevent @dragenter.prevent class="column-content"
            :class="{ 'drag-over': dragOverColumn === 'testing' }">
            <div v-for="task in getFilteredTasksByStatus('testing')" :key="task.id" :draggable="true"
              @dragstart="onDragStart($event, task)" @dragend="onDragEnd" @dblclick="viewTask(task)"
              @contextmenu.prevent="showContextMenuFor(task, $event)" class="task-card testing-card" :class="{
                'parent-task': isParentTask(task.id),
                'child-task': task.parentId,
                'highlighted': highlightedTaskId === task.id
              }">
              <div class="task-header">
                <h3 class="task-title">{{ task.name }}</h3>
                <div class="task-actions">
                  <button @click.stop="openCommentsModal(task)" class="comments-button"
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
            <div v-if="getFilteredTasksByStatus('testing').length === 0" class="empty-column">
              No hay tareas en testing
            </div>
          </div>
        </div>

        <!-- Columna Finalized -->
        <div class="column">
          <div class="column-header finalized-header">
            <h2 class="column-title">
              <div class="status-dot finalized-dot"></div>
              Finalized
              <span class="task-count">({{ getFilteredTasksByStatus('finalized').length }})</span>
            </h2>
          </div>
          <div @drop="onDrop($event, 'finalized')" @dragover.prevent @dragenter.prevent class="column-content"
            :class="{ 'drag-over': dragOverColumn === 'finalized' }">
            <div v-for="task in getFilteredTasksByStatus('finalized')" :key="task.id" :draggable="true"
              @dragstart="onDragStart($event, task)" @dragend="onDragEnd" @dblclick="viewTask(task)"
              @contextmenu.prevent="showContextMenuFor(task, $event)" class="task-card finalized-card" :class="{
                'parent-task': isParentTask(task.id),
                'child-task': task.parentId,
                'highlighted': highlightedTaskId === task.id
              }">
              <div class="task-header">
                <h3 class="task-title">{{ task.name }}</h3>
                <div class="task-actions">
                  <button @click.stop="openCommentsModal(task)" class="comments-button"
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
            <div v-if="getFilteredTasksByStatus('finalized').length === 0" class="empty-column">
              No hay tareas finalizadas
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>

  <!-- Componente TreeNode como template separado -->
  <TreeNodeComponent v-if="false" :task="null" :level="0" />
</template>

<script setup>
import { ref, onMounted, watch, computed, defineComponent, nextTick } from 'vue'

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

// Estado reactivo
const tasks = ref([])
const comments = ref([])
const boardTitle = ref('Custom Board')
const isEditingTitle = ref(false)
const editingTitle = ref('')
const titleInput = ref(null)

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

// Estado del modal JSON
const showJsonModal = ref(false)
const jsonCopied = ref(false)

// Estado del modal de visualización
const showViewModal = ref(false)
const viewingTask = ref(null)

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
      if (data.settings && data.settings.title_board) {
        boardTitle.value = data.settings.title_board
        saveSettings()
      }
    } else {
      throw new Error('El archivo no contiene un formato válido')
    }

    // Guardar en localStorage
    saveTasks()
    saveComments()

    // Resetear el input para permitir cargar el mismo archivo de nuevo
    event.target.value = null

    alert('Datos importados correctamente')
  } catch (error) {
    console.error('Error importing data:', error)
    alert('Error al importar los datos: ' + (error.message || 'Formato de archivo no válido'))
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
function exportData() {
  try {
    const dataToExport = {
      settings: {
        title_board: boardTitle.value
      },
      tasks: tasks.value,
      comments: comments.value
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

// Estados disponibles
const availableStatuses = [
  { value: 'pendiente', label: 'Pendiente', class: 'status-pendiente' },
  { value: 'en-progreso', label: 'En Progreso', class: 'status-en-progreso' },
  { value: 'completado', label: 'Completado', class: 'status-completado' }
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

function getStatusClass(status) {
  const statusObj = availableStatuses.find(s => s.value === status)
  return statusObj ? statusObj.class : ''
}

function togglePriority(priority) {
  const index = selectedPriorities.value.indexOf(priority)
  if (index > -1) {
    selectedPriorities.value.splice(index, 1)
  } else {
    selectedPriorities.value.push(priority)
  }
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
function saveSettings() {
  const settings = {
    title_board: boardTitle.value
  }
  localStorage.setItem('boardSettings', JSON.stringify(settings))
}

// Manejar la edición del título
function startEditingTitle() {
  editingTitle.value = boardTitle.value
  isEditingTitle.value = true
  nextTick(() => {
    titleInput.value?.focus()
    titleInput.value?.select()
  })
}

function saveBoardTitle() {
  if (editingTitle.value.trim()) {
    boardTitle.value = editingTitle.value.trim()
    saveSettings()
  }
  isEditingTitle.value = false
}

function cancelEditTitle() {
  isEditingTitle.value = false
}

// Cargar tareas desde localStorage al montar el componente
onMounted(() => {
  loadSettings()
  loadTasks()
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
      tasks.value = [
        {
          id: 1,
          name: 'Implementar autenticación',
          description: 'Crear sistema de login y registro',
          products: ['LIGHTREVIEW'],
          priority: 'alto',
          jiraLink: 'https://jira.company.com/browse/LR-123',
          status: 'backlog',
          parentId: null,
          createdAt: new Date().toISOString()
        },
        {
          id: 2,
          name: 'Diseñar dashboard',
          description: 'Crear interfaz principal del usuario',
          products: ['DASHBOARD'],
          priority: 'medio',
          jiraLink: '',
          status: 'paralyzed',
          parentId: null,
          createdAt: new Date().toISOString()
        },
        {
          id: 3,
          name: 'Validación de formularios',
          description: 'Añadir validaciones al login',
          products: ['LIGHTREVIEW'],
          priority: 'bajo',
          jiraLink: 'https://jira.company.com/browse/LR-124',
          status: 'backlog',
          parentId: 1,
          createdAt: new Date().toISOString()
        },
        {
          id: 4,
          name: 'Pruebas unitarias',
          description: 'Crear tests para el sistema de autenticación',
          products: ['LIGHTREVIEW', 'API'],
          priority: 'muy-alto',
          jiraLink: 'https://jira.company.com/browse/LR-125',
          status: 'testing',
          parentId: null,
          createdAt: new Date().toISOString()
        }
      ]
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

function clearSearch() {
  searchTerm.value = ''
}

function toggleProduct(product) {
  const index = selectedProducts.value.indexOf(product)
  if (index > -1) {
    selectedProducts.value.splice(index, 1)
  } else {
    selectedProducts.value.push(product)
  }
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

function openCommentsModal(task) {
  commentsTask.value = task
  showCommentsModal.value = true
  newComment.value = {
    text: '',
    matrixValue: 'not-important-not-urgent'
  }
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
  viewingTask.value = task
  showViewModal.value = true
}

// Función para resaltar tarea
function highlightTask(taskId) {
  // Limpiar el temporizador anterior si existe
  if (highlightTimer) {
    clearTimeout(highlightTimer)
  }

  // Establecer la tarea resaltada
  highlightedTaskId.value = taskId
  showViewModal.value = false

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



<style scoped>
/* Form layout */
.form-row {
  display: flex;
  gap: 1rem;
  margin-bottom: 1rem;
}

.form-row .form-group {
  flex: 1;
  margin-bottom: 0;
}

/* Date inputs */
input[type="date"] {
  padding: 0.5rem;
  border: 1px solid #d1d5db;
  border-radius: 0.375rem;
  width: 100%;
  font-family: inherit;
}

input[type="date"]:focus {
  outline: none;
  border-color: #3b82f6;
  box-shadow: 0 0 0 2px rgba(59, 130, 246, 0.2);
}

/* Optional label */
.optional-label {
  color: #6b7280;
  font-size: 0.875rem;
  font-weight: normal;
}
/* Estilos base */
* {
  box-sizing: border-box;
}

.kanban-container {
  height: 100vh;
  background-color: #f8fafc;
  padding: 24px;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
  display: flex;
  flex-direction: column;
  overflow: hidden;
}

.kanban-wrapper {
  max-width: 100%;
  margin: 0 auto;
  display: flex;
  flex-direction: column;
  height: 100%;
  overflow: hidden;
}

/* Header mejorado */
.header {
  margin-bottom: 24px;
  flex-shrink: 0;
}

.header-top {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 20px;
}

.header-buttons {
  display: flex;
  gap: 16px;
}

.title-container {
  display: flex;
  align-items: center;
  position: relative;
}

.title {
  font-size: 2rem;
  font-weight: bold;
  color: #1f2937;
  margin: 0;
  cursor: pointer;
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 4px 8px;
  border-radius: 4px;
  transition: background-color 0.2s;
}

.title:hover {
  background-color: #f3f4f6;
}

.edit-icon {
  font-size: 1.2rem;
  opacity: 0.7;
  transition: opacity 0.2s;
}

.title:hover .edit-icon {
  opacity: 1;
}

.title-input {
  font-size: 2rem;
  font-weight: bold;
  color: #1f2937;
  border: 2px solid #3b82f6;
  border-radius: 4px;
  padding: 4px 8px;
  width: 300px;
  outline: none;
}

.title-input:focus {
  border-color: #2563eb;
  box-shadow: 0 0 0 2px rgba(59, 130, 246, 0.2);
}

.json-button,
.export-button,
.import-button {
  padding: 12px 24px;
  color: white;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  font-weight: 500;
  transition: background-color 0.2s;
  margin-left: 16px;
}

.json-button,
.export-button,
.import-button {
  background-color: #3b82f6;
}

.json-button:hover,
.export-button:hover,
.import-button:hover {
  background-color: #2563eb;
}

.tree-button {
  padding: 8px 16px;
  background-color: #10b981;
  color: white;
  border: none;
  border-radius: 6px;
  font-size: 14px;
  cursor: pointer;
  transition: background-color 0.2s;
  margin-left: 8px;
}

.tree-button:hover {
  background-color: #059669;
}

/* Estilos para la vista de árbol genealógico como página completa */
.tree-view-container {
  flex: 1;
  display: flex;
  flex-direction: column;
  overflow: hidden;
}

.tree-view-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 24px;
  padding-bottom: 16px;
  border-bottom: 2px solid #e5e7eb;
}

.tree-view-title {
  font-size: 2rem;
  font-weight: bold;
  color: #1e293b;
  margin: 0;
}

.back-button-main {
  padding: 12px 24px;
  background-color: #6b7280;
  color: white;
  border: none;
  border-radius: 8px;
  font-size: 16px;
  cursor: pointer;
  transition: background-color 0.2s;
  font-weight: 500;
}

.back-button-main:hover {
  background-color: #4b5563;
}

.tree-view-content {
  flex: 1;
  display: flex;
  flex-direction: column;
  overflow: hidden;
}

.parent-selector-main {
  margin-bottom: 32px;
  padding: 24px;
  background-color: white;
  border-radius: 12px;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
  border: 1px solid #e5e7eb;
}

.tree-selector {
  max-width: 400px;
}

.tree-container-main {
  flex: 1;
  background-color: white;
  border-radius: 12px;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
  border: 1px solid #e5e7eb;
  padding: 32px;
  overflow: auto;
}

.tree-wrapper-main {
  min-width: fit-content;
}

.no-parent-selected-main {
  flex: 1;
  display: flex;
  align-items: center;
  justify-content: center;
}

.empty-state {
  text-align: center;
  max-width: 400px;
}

.empty-icon {
  font-size: 4rem;
  margin-bottom: 16px;
}

.empty-state h3 {
  font-size: 1.5rem;
  font-weight: 600;
  color: #1e293b;
  margin: 0 0 12px 0;
}

.empty-state p {
  color: #6b7280;
  line-height: 1.6;
  margin: 0;
}

.search-filters-section {
  display: flex;
  gap: 16px;
  align-items: center;
  flex-wrap: wrap;
}

.search-container {
  flex: 1;
  min-width: 300px;
  position: relative;
}

.search-input {
  width: 100%;
  padding: 12px 16px;
  border: 2px solid #d1d5db;
  border-radius: 8px;
  font-size: 16px;
  outline: none;
  transition: border-color 0.2s;
  padding-right: 40px;
}

.search-input:focus {
  border-color: #3b82f6;
}

.clear-search-button {
  position: absolute;
  right: 12px;
  top: 50%;
  transform: translateY(-50%);
  background: none;
  border: none;
  color: #6b7280;
  font-size: 18px;
  cursor: pointer;
  padding: 4px;
  line-height: 1;
}

.clear-search-button:hover {
  color: #374151;
}

.filters-container {
  display: flex;
  gap: 16px;
  align-items: center;
}

.filter-group {
  display: flex;
  align-items: center;
  gap: 8px;
}

.filter-label {
  font-size: 14px;
  font-weight: 500;
  color: #1e293b;
  white-space: nowrap;
}

.multiselect-container {
  position: relative;
  min-width: 180px;
}

.multiselect-trigger {
  padding: 8px 12px;
  border: 2px solid #d1d5db;
  border-radius: 6px;
  background-color: white;
  cursor: pointer;
  display: flex;
  justify-content: space-between;
  align-items: center;
  font-size: 14px;
}

.multiselect-trigger:hover {
  border-color: #9ca3af;
}

.dropdown-arrow {
  font-size: 12px;
  color: #6b7280;
}

.multiselect-dropdown {
  position: absolute;
  top: 100%;
  left: 0;
  right: 0;
  background-color: white;
  border: 2px solid #d1d5db;
  border-top: none;
  border-radius: 0 0 6px 6px;
  max-height: 200px;
  overflow-y: auto;
  z-index: 10;
  box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.1);
}

.multiselect-option {
  padding: 8px 12px;
  cursor: pointer;
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 14px;
  transition: background-color 0.2s;
}

.multiselect-option:hover {
  background-color: #f9fafb;
}

.multiselect-option.selected {
  background-color: #eff6ff;
  color: #1e40af;
}

.add-button {
  padding: 12px 24px;
  background-color: #3b82f6;
  color: white;
  border: none;
  border-radius: 8px;
  font-size: 16px;
  font-weight: 500;
  cursor: pointer;
  transition: background-color 0.2s;
  white-space: nowrap;
}

.add-button:hover {
  background-color: #2563eb;
}

/* Estilos para prioridades */
.priority-indicator {
  width: 12px;
  height: 12px;
  border-radius: 50%;
  display: inline-block;
}

.priority-muy-bajo {
  background-color: #e5e7eb;
}

.priority-bajo {
  background-color: #60a5fa;
}

.priority-medio {
  background-color: #fbbf24;
}

.priority-alto {
  background-color: #f97316;
}

.priority-muy-alto {
  background-color: #ef4444;
}

.priority-badge {
  font-size: 10px;
  padding: 2px 6px;
  border-radius: 10px;
  font-weight: 500;
  white-space: nowrap;
}

.priority-badge.priority-muy-bajo {
  background-color: #f3f4f6;
  color: #6b7280;
}

.priority-badge.priority-bajo {
  background-color: #dbeafe;
  color: #1e40af;
}

.priority-badge.priority-medio {
  background-color: #fef3c7;
  color: #92400e;
}

.priority-badge.priority-alto {
  background-color: #fed7aa;
  color: #c2410c;
}

.priority-badge.priority-muy-alto {
  background-color: #fecaca;
  color: #dc2626;
}

/* Estilos para campos opcionales */
.optional-label {
  color: #9ca3af;
  font-weight: normal;
  font-size: 12px;
}

/* Estilos para JIRA LINK */
.form-error {
  color: #ef4444;
  font-size: 0.8rem;
  margin-top: 0.25rem;
}

.form-hint {
  color: #6b7280;
  font-size: 0.75rem;
  margin-top: 0.25rem;
  font-style: italic;
}

.input-with-validation {
  position: relative;
  display: flex;
  align-items: center;
}

.validation-icon {
  position: absolute;
  right: 10px;
  font-weight: bold;
  pointer-events: none;
}

.validation-icon.valid {
  color: #10b981; /* green-500 */
}

.validation-icon.invalid {
  color: #ef4444; /* red-500 */
}

.input-error {
  border-color: #ef4444 !important;
  padding-right: 30px;
}

.jira-link {
  color: #3b82f6;
  text-decoration: none;
  word-break: break-all;
}

.jira-link:hover {
  text-decoration: underline;
}

.jira-link-container {
  margin-bottom: 8px;
}

.jira-link-card {
  display: inline-flex;
  align-items: center;
  gap: 4px;
  padding: 4px 8px;
  background-color: #1e40af;
  color: white;
  text-decoration: none;
  border-radius: 4px;
  font-size: 11px;
  font-weight: 500;
  transition: background-color 0.2s;
}

.jira-link-card:hover {
  background-color: #1d4ed8;
  color: white;
}

/* Botón de comentarios */
.comments-button {
  background: none;
  border: 1px solid #d1d5db;
  color: #6b7280;
  padding: 4px 8px;
  border-radius: 4px;
  font-size: 11px;
  cursor: pointer;
  transition: all 0.2s;
  display: flex;
  align-items: center;
  gap: 2px;
}

.comments-button:hover {
  background-color: #f3f4f6;
  border-color: #9ca3af;
  color: #374151;
}

.task-actions {
  display: flex;
  align-items: center;
  gap: 8px;
}

/* Modal de comentarios */
.comments-modal {
  max-width: 700px;
  max-height: 80vh;
}

.comments-modal-content {
  padding: 0 24px 24px 24px;
  max-height: 60vh;
  overflow-y: auto;
}

.comments-list {
  max-height: 300px;
  overflow-y: auto;
  margin-bottom: 24px;
  border: 1px solid #e5e7eb;
  border-radius: 8px;
  padding: 16px;
}

.no-comments {
  text-align: center;
  color: #9ca3af;
  font-style: italic;
  padding: 32px 16px;
}

.comment-item {
  padding: 12px;
  border-radius: 8px;
  margin-bottom: 12px;
  border-left: 4px solid #e5e7eb;
}

.comment-item:last-child {
  margin-bottom: 0;
}

/* Estilos para la Matriz de Eisenhower en comentarios */
.comment-item.matrix-important-urgent {
  border-left-color: #dc2626;
  background-color: #fef2f2;
}

.comment-item.matrix-important-not-urgent {
  border-left-color: #f59e0b;
  background-color: #fffbeb;
}

.comment-item.matrix-not-important-urgent {
  border-left-color: #3b82f6;
  background-color: #eff6ff;
}

.comment-item.matrix-not-important-not-urgent {
  border-left-color: #6b7280;
}

/* Estilos para el modal combinado de comentarios y vista de tarea */
.combined-modal-overlay {
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 1000;
  background-color: rgba(0, 0, 0, 0.5);
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
}

.combined-modal-container {
  position: relative;
  background: white;
  border-radius: 12px;
  box-shadow: 0 5px 30px rgba(0, 0, 0, 0.3);
  width: 90%;
  max-width: 1400px;
  height: 90vh;
  max-height: 90vh;
  display: flex;
  flex-direction: column;
  overflow: hidden;
}

.combined-close-button {
  position: absolute;
  top: 15px;
  right: 15px;
  z-index: 10;
  width: 32px;
  height: 32px;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 20px;
  cursor: pointer;
  border: none;
  background: #ef4444;
  color: white;
  transition: all 0.2s ease;
}

.combined-close-button:hover {
  background: #dc2626;
  transform: scale(1.1);
  box-shadow: 0 4px 8px rgba(239, 68, 68, 0.4);
}

.combined-modal-content {
  display: flex;
  height: 100%;
  overflow: hidden;
}

.comments-section {
  flex: 1;
  overflow-y: auto;
  padding: 20px;
  border-right: 1px solid #e5e7eb;
  display: flex;
  flex-direction: column;
}

.comments-section .comments-modal-content {
  flex: 1;
  display: flex;
  flex-direction: column;
  padding: 0;
  max-height: none;
}

.comments-section .comments-list {
  flex: 1;
  max-height: none;
  margin-bottom: 20px;
}

.task-view-section {
  width: 400px;
  overflow-y: auto;
  padding: 20px;
  background-color: #f9fafb;
  border-left: 1px solid #e5e7eb;
}

.task-view-section .view-modal-content {
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.view-field {
  padding: 8px 0;
  border-bottom: 1px solid #e5e7eb;
}

.view-field:last-child {
  border-bottom: none;
}

.task-actions-row {
  margin-top: 20px;
  display: flex;
  justify-content: flex-end;
}

.action-button {
  padding: 8px 16px;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  font-weight: 500;
  transition: all 0.2s;
  display: flex;
  align-items: center;
  gap: 6px;
}

.edit-button {
  background-color: #3b82f6;
  color: white;
}

.edit-button:hover {
  background-color: #2563eb;
}

/* Ajustes responsivos para pantallas más pequeñas */
@media (max-width: 1024px) {
  .combined-modal-content {
    flex-direction: column;
  }
  
  .task-view-section {
    width: 100%;
    border-left: none;
    border-top: 1px solid #e5e7eb;
  }
  
  .comments-section {
    border-right: none;
  }
}

.comment-header {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  margin-bottom: 8px;
  gap: 8px;
}

.comment-header-left {
  display: flex;
  flex-direction: column;
  gap: 2px;
  flex: 1;
}

.comment-priority {
  font-size: 10px;
  padding: 2px 6px;
  border-radius: 10px;
  font-weight: 500;
}

.comment-priority-important-urgent {
  background-color: #fecaca;
  color: #dc2626;
}

.comment-priority-important-not-urgent {
  background-color: #fef3c7;
  color: #92400e;
}

.comment-priority-not-important-urgent {
  background-color: #dbeafe;
  color: #1e40af;
}

.comment-priority-not-important-not-urgent {
  background-color: #f3f4f6;
  color: #6b7280;
}

.comment-date {
  font-size: 12px;
  color: #6b7280;
}

.comment-completed-date {
  font-size: 10px;
  color: #10b981;
  font-weight: 500;
}

.comment-actions-buttons {
  display: flex;
  gap: 4px;
  flex-shrink: 0;
}

.comment-action-btn {
  background: none;
  border: 1px solid #d1d5db;
  color: #6b7280;
  padding: 4px 6px;
  border-radius: 4px;
  font-size: 12px;
  cursor: pointer;
  transition: all 0.2s;
  display: flex;
  align-items: center;
  justify-content: center;
  min-width: 24px;
  height: 24px;
}

.comment-action-btn:hover {
  background-color: #f3f4f6;
  border-color: #9ca3af;
}

.comment-action-btn.completed {
  background-color: #10b981;
  border-color: #10b981;
  color: white;
}

.comment-action-btn.completed:hover {
  background-color: #059669;
  border-color: #059669;
}

.comment-action-btn.delete:hover {
  background-color: #fef2f2;
  border-color: #fca5a5;
  color: #dc2626;
}

.comment-text {
  color: #1e293b;
  line-height: 1.5;
}

.comment-text.completed-text {
  text-decoration: line-through;
  color: #9ca3af;
}

.comment-item.comment-completed {
  opacity: 0.7;
  background-color: #f8f9fa !important;
}

.new-comment-form {
  border-top: 1px solid #e5e7eb;
  padding-top: 16px;
}

/* Matriz de Eisenhower para formulario */
.eisenhower-matrix {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 12px;
  margin-top: 8px;
}

.matrix-row {
  display: contents;
}

.matrix-option {
  display: flex;
  align-items: flex-start;
  gap: 8px;
  padding: 12px;
  border: 2px solid #e5e7eb;
  border-radius: 8px;
  cursor: pointer;
  transition: all 0.2s;
}

.matrix-option:hover {
  border-color: #9ca3af;
  background-color: #f9fafb;
}

.matrix-option input[type="radio"] {
  margin-top: 2px;
}

.matrix-option input[type="radio"]:checked+.matrix-label {
  font-weight: 600;
}

.matrix-label {
  display: flex;
  flex-direction: column;
  gap: 2px;
}

.matrix-label strong {
  font-size: 14px;
  line-height: 1.2;
}

.matrix-label small {
  font-size: 12px;
  color: #6b7280;
  font-style: italic;
}

.matrix-label.important-urgent strong {
  color: #dc2626;
}

.matrix-label.important-not-urgent strong {
  color: #f59e0b;
}

.matrix-label.not-important-urgent strong {
  color: #3b82f6;
}

.matrix-label.not-important-not-urgent strong {
  color: #6b7280;
}

.comment-actions {
  display: flex;
  justify-content: flex-end;
  margin-top: 16px;
}

/* Botón de copiar JSON */
.modal-header-actions {
  display: flex;
  align-items: center;
  gap: 12px;
}

.copy-button {
  padding: 6px 12px;
  background-color: #10b981;
  color: white;
  border: none;
  border-radius: 6px;
  font-size: 12px;
  cursor: pointer;
  transition: all 0.2s;
  display: flex;
  align-items: center;
  gap: 4px;
}

.copy-button:hover {
  background-color: #059669;
}

.copy-button.copied {
  background-color: #059669;
  transform: scale(0.95);
}

.back-button {
  padding: 6px 12px;
  background-color: #6b7280;
  color: white;
  border: none;
  border-radius: 6px;
  font-size: 12px;
  cursor: pointer;
  transition: background-color 0.2s;
}

.back-button:hover {
  background-color: #4b5563;
}

/* Cruz de cerrar roja y bonita */
.close-button-red {
  background: linear-gradient(135deg, #ef4444, #dc2626);
  border: none;
  color: white;
  width: 32px;
  height: 32px;
  border-radius: 50%;
  font-size: 18px;
  font-weight: bold;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all 0.2s;
  box-shadow: 0 2px 4px rgba(239, 68, 68, 0.3);
}

.close-button-red:hover {
  background: linear-gradient(135deg, #dc2626, #b91c1c);
  transform: scale(1.1);
  box-shadow: 0 4px 8px rgba(239, 68, 68, 0.4);
}

.close-button-red:active {
  transform: scale(0.95);
}

/* Modal JSON */
.json-modal {
  max-width: 800px;
  max-height: 80vh;
}

.json-modal-content {
  padding: 0 24px 24px 24px;
  max-height: 60vh;
  overflow-y: auto;
}

.json-display-modal {
  background-color: #f3f4f6;
  padding: 16px;
  border-radius: 8px;
  font-size: 12px;
  white-space: pre-wrap;
  margin: 0;
  font-family: 'Monaco', 'Menlo', 'Ubuntu Mono', monospace;
  line-height: 1.4;
}

/* Modal más ancho */
.modal-wide {
  max-width: 700px;
}

/* Modal de eliminación */
.delete-modal {
  max-width: 450px;
}

.delete-modal-content {
  padding: 0 24px 24px 24px;
  text-align: center;
}

.delete-icon {
  font-size: 48px;
  margin-bottom: 16px;
}

.delete-message {
  font-size: 16px;
  color: #1e293b;
  margin-bottom: 16px;
  line-height: 1.5;
}

.warning-box {
  background-color: #fef3c7;
  border: 1px solid #f59e0b;
  border-radius: 8px;
  padding: 16px;
  margin-top: 16px;
  text-align: left;
}

.warning-box p {
  margin: 0 0 8px 0;
  color: #92400e;
  font-size: 14px;
  line-height: 1.4;
}

.warning-box p:last-child {
  margin-bottom: 0;
}

.delete-button {
  padding: 10px 20px;
  background-color: #ef4444;
  color: white;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  transition: background-color 0.2s;
  font-weight: 500;
}

.delete-button:hover {
  background-color: #dc2626;
}

/* Modal de visualización */
.view-modal-content {
  padding: 0 24px 24px 24px;
}

.view-field {
  margin-bottom: 16px;
  line-height: 1.5;
}

.subtasks-list {
  margin-top: 8px;
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.subtask-link {
  align-self: flex-start;
}

/* Menú contextual */
.context-menu {
  position: fixed;
  background-color: white;
  border: 1px solid #e5e7eb;
  border-radius: 8px;
  box-shadow: 0 10px 15px -3px rgba(0, 0, 0, 0.1);
  z-index: 1000;
  min-width: 120px;
}

.context-menu-item {
  display: block;
  width: 100%;
  padding: 12px 16px;
  text-align: left;
  background: none;
  border: none;
  cursor: pointer;
  font-size: 14px;
  transition: background-color 0.2s;
}

.context-menu-item:hover {
  background-color: #f9fafb;
}

.context-menu-item.delete:hover {
  background-color: #fef2f2;
  color: #dc2626;
}

/* Botones de enlace */
.link-button {
  background: none;
  border: none;
  color: #3b82f6;
  cursor: pointer;
  text-decoration: underline;
  font-size: inherit;
  padding: 0;
  margin: 0;
}

.link-button:hover {
  color: #1d4ed8;
}

.child-link {
  font-size: 11px;
  margin-right: 8px;
}

.children-links {
  margin-top: 4px;
  font-size: 11px;
}

/* Modal (estilos existentes) */
.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background-color: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
  padding: 20px;
}

.modal-content {
  background-color: white;
  border-radius: 12px;
  box-shadow: 0 20px 25px -5px rgba(0, 0, 0, 0.1);
  width: 100%;
  max-width: 500px;
  max-height: 90vh;
  overflow-y: auto;
}

.modal-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 24px 24px 0 24px;
  border-bottom: 1px solid #e5e7eb;
  margin-bottom: 24px;
  position: relative;
}
.modal-header .close-button-red{
  position: absolute;
  top:0;
  right:0;
  margin: 10px;
}

.modal-title .priority-badge{
  position: absolute;
  top: 0;
  left: 0;
  padding: 4px 12px;
  border-radius: 0 0 10px 0;
  font-size: 14px;
}

.modal-title {
  font-size: 1.5rem;
  font-weight: 600;
  color: #1e293b;
  margin: 0;
}

.modal-title {
  display: flex;
  align-items: center;
  justify-content: space-between;
  width: 85%;
}

.modal-form {
  padding: 0 24px 24px 24px;
}

.form-group {
  margin-bottom: 20px;
}

.form-label {
  display: block;
  font-size: 14px;
  font-weight: 500;
  color: #1e293b;
  margin-bottom: 6px;
}

.form-input,
.form-textarea {
  width: 100%;
  padding: 12px 16px;
  border: 2px solid #d1d5db;
  border-radius: 8px;
  font-size: 16px;
  outline: none;
  transition: border-color 0.2s;
}

.form-input:focus,
.form-textarea:focus {
  border-color: #3b82f6;
}

.form-textarea {
  resize: vertical;
  min-height: 80px;
}

/* Select con buscador */
.select-container {
  position: relative;
}

.dropdown {
  position: absolute;
  top: 100%;
  left: 0;
  right: 0;
  background-color: white;
  border: 2px solid #d1d5db;
  border-top: none;
  border-radius: 0 0 8px 8px;
  max-height: 200px;
  overflow-y: auto;
  z-index: 10;
  box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.1);
}

.dropdown-item {
  padding: 12px 16px;
  cursor: pointer;
  border-bottom: 1px solid #f3f4f6;
  transition: background-color 0.2s;
}

.dropdown-item:hover {
  background-color: #f9fafb;
}

.dropdown-item:last-child {
  border-bottom: none;
}

.dropdown-item-content {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 4px;
  gap: 8px;
}

.dropdown-item-name {
  font-weight: 500;
  color: #1e293b;
  flex: 1;
}

.dropdown-item-id {
  font-size: 11px;
  color: #6b7280;
  background-color: #f3f4f6;
  padding: 2px 6px;
  border-radius: 8px;
}

.dropdown-item-status {
  font-size: 12px;
  color: #6b7280;
  background-color: #f3f4f6;
  padding: 2px 8px;
  border-radius: 12px;
}

.dropdown-item-description {
  font-size: 12px;
  color: #6b7280;
}

.dropdown-item-empty {
  padding: 12px 16px;
  color: #9ca3af;
  font-style: italic;
  text-align: center;
}

.selected-parent {
  margin-top: 8px;
  padding: 8px 12px;
  background-color: #f0f9ff;
  border: 1px solid #bfdbfe;
  border-radius: 6px;
  display: flex;
  justify-content: space-between;
  align-items: center;
  font-size: 14px;
}

.clear-parent {
  background: none;
  border: none;
  color: #6b7280;
  cursor: pointer;
  font-size: 18px;
  padding: 0;
  margin-left: 8px;
}

.clear-parent:hover {
  color: #374151;
}

.modal-actions {
  display: flex;
  gap: 12px;
  justify-content: flex-end;
  margin-top: 24px;
  padding-top: 20px;
  border-top: 1px solid #e5e7eb;
}

.cancel-button {
  padding: 10px 20px;
  background-color: #f3f4f6;
  color: #1e293b;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  transition: background-color 0.2s;
}

.cancel-button:hover {
  background-color: #e5e7eb;
}

.submit-button {
  padding: 10px 20px;
  background-color: #3b82f6;
  color: white;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  transition: background-color 0.2s;
}

.submit-button:hover:not(:disabled) {
  background-color: #2563eb;
}

.submit-button:disabled {
  background-color: #9ca3af;
  cursor: not-allowed;
}

/* Estilos para productos múltiples */
.product-input-container {
  border: 2px solid #d1d5db;
  border-radius: 8px;
  padding: 8px;
  background-color: white;
  transition: border-color 0.2s;
}

.product-input-container:focus-within {
  border-color: #3b82f6;
}

.selected-products {
  display: flex;
  flex-wrap: wrap;
  gap: 6px;
  margin-bottom: 8px;
}

.product-tag {
  background-color: #3b82f6;
  color: white;
  padding: 4px 8px;
  border-radius: 16px;
  font-size: 12px;
  font-weight: 500;
  display: flex;
  align-items: center;
  gap: 4px;
}

.remove-product {
  background: none;
  border: none;
  color: white;
  cursor: pointer;
  font-size: 14px;
  padding: 0;
  margin-left: 4px;
  line-height: 1;
}

.remove-product:hover {
  color: #fecaca;
}

.product-autocomplete {
  position: relative;
}

.product-autocomplete .form-input {
  border: none;
  padding: 8px 0;
  margin: 0;
}

.product-autocomplete .form-input:focus {
  border: none;
  outline: none;
}

.product-suggestions {
  position: absolute;
  top: 100%;
  left: 0;
  right: 0;
  background-color: white;
  border: 1px solid #d1d5db;
  border-radius: 6px;
  max-height: 150px;
  overflow-y: auto;
  z-index: 10;
  box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.1);
}

.product-suggestion {
  padding: 8px 12px;
  cursor: pointer;
  font-size: 14px;
  transition: background-color 0.2s;
  color: #1e293b;
}

.product-suggestion:hover {
  background-color: #f9fafb;
}

.product-suggestion:last-child {
  border-bottom: none;
}

/* Columnas - Layout optimizado para pantalla completa */
.columns-container {
  display: flex;
  gap: 16px;
  flex: 1;
  overflow: hidden;
  min-height: 0;
}

.column {
  flex: 1;
  background-color: white;
  border-radius: 12px;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
  border: 1px solid #e5e7eb;
  display: flex;
  flex-direction: column;
  min-width: 280px;
  overflow: hidden;
}

.column-header {
  padding: 16px;
  border-bottom: 1px solid #e5e7eb;
  flex-shrink: 0;
}

.column-title {
  font-size: 1.125rem;
  font-weight: 600;
  color: #1e293b;
  display: flex;
  align-items: center;
  gap: 8px;
  margin: 0;
}

.status-dot {
  width: 12px;
  height: 12px;
  border-radius: 50%;
}

.backlog-dot {
  background-color: #6b7280;
}

.paralyzed-dot {
  background-color: #f59e0b;
}

.progress-dot {
  background-color: #3b82f6;
}

.testing-dot {
  background-color: #f97316;
}

.finalized-dot {
  background-color: #10b981;
}

.task-count {
  font-size: 0.875rem;
  color: #6b7280;
  font-weight: normal;
}

.column-content {
  padding: 16px;
  flex: 1;
  overflow-y: auto;
  transition: background-color 0.2s;
}

.column-content.drag-over {
  background-color: #f0f9ff;
}

/* Tarjetas de tareas mejoradas */
.task-card {
  background-color: #f9fafb;
  padding: 16px;
  border-radius: 8px;
  border: 1px solid #e5e7eb;
  margin-bottom: 12px;
  cursor: move;
  transition: all 0.2s;
}

.task-card:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
}

.task-card:active {
  transform: scale(0.98);
}

.task-card.highlighted {
  border: 3px solid #ef4444 !important;
  box-shadow: 0 0 0 2px rgba(239, 68, 68, 0.2);
  animation: pulse 1s infinite;
}

@keyframes pulse {

  0%,
  100% {
    box-shadow: 0 0 0 2px rgba(239, 68, 68, 0.2);
  }

  50% {
    box-shadow: 0 0 0 4px rgba(239, 68, 68, 0.1);
  }
}

.backlog-card {
  border-left: 4px solid #6b7280;
}

.paralyzed-card {
  background-color: #fffbeb;
  border-left: 4px solid #f59e0b;
}

.progress-card {
  background-color: #eff6ff;
  border-left: 4px solid #3b82f6;
}

.testing-card {
  background-color: #fff7ed;
  border-left: 4px solid #f97316;
}

.finalized-card {
  background-color: #f0fdf4;
  border-left: 4px solid #10b981;
}

.parent-task {
  border-left-width: 6px !important;
}

.child-task {
  margin-left: 12px;
  border-left-style: dashed !important;
}

.task-header {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  margin-bottom: 8px;
}

.task-title {
  font-weight: 500;
  color: #1e293b;
  margin: 0;
  font-size: 1rem;
  flex: 1;
}

.task-badges {
  display: flex;
  gap: 4px;
}

.parent-badge,
.child-badge {
  font-size: 10px;
  padding: 2px 6px;
  border-radius: 10px;
  font-weight: 500;
  white-space: nowrap;
}

.parent-badge {
  background-color: #dbeafe;
  color: #1e40af;
}

.child-badge {
  background-color: #fef3c7;
  color: #92400e;
}

.task-meta {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 8px;
  font-size: 0.75rem;
  gap: 8px;
  flex-wrap: wrap;
}

.task-id {
  color: #6b7280;
  background-color: #f3f4f6;
  padding: 2px 6px;
  border-radius: 8px;
}

.task-product {
  color: #3b82f6;
  background-color: #eff6ff;
  padding: 2px 8px;
  border-radius: 12px;
  font-weight: 500;
}

.task-description {
  font-size: 0.875rem;
  color: #6b7280;
  margin-bottom: 8px;
  line-height: 1.4;
}

.parent-info,
.children-info {
  font-size: 0.75rem;
  color: #6b7280;
  margin-bottom: 8px;
  font-style: italic;
}

.task-info {
  display: flex;
  justify-content: space-between;
  align-items: center;
  font-size: 0.875rem;
  color: #6b7280;
}

.empty-column {
  text-align: center;
  color: #9ca3af;
  padding: 32px 16px;
  font-style: italic;
}

/* Estilos para el árbol genealógico */
.tree-node {
  position: relative;
  margin-bottom: 16px;
}

.tree-task-card {
  background-color: white;
  border: 2px solid #e5e7eb;
  border-radius: 8px;
  padding: 12px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
  transition: all 0.2s;
  max-width: 300px;
}

.tree-task-card:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.15);
}

.tree-task-header {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  margin-bottom: 8px;
}

.tree-task-title {
  font-size: 14px;
  font-weight: 600;
  color: #1e293b;
  margin: 0;
  flex: 1;
}

.tree-task-actions {
  display: flex;
  gap: 4px;
}

.tree-action-btn {
  background: none;
  border: 1px solid #d1d5db;
  color: #6b7280;
  padding: 2px 4px;
  border-radius: 4px;
  font-size: 10px;
  cursor: pointer;
  transition: all 0.2s;
}

.tree-action-btn:hover {
  background-color: #f3f4f6;
  border-color: #9ca3af;
}

.tree-task-meta {
  display: flex;
  flex-wrap: wrap;
  gap: 4px;
  margin-bottom: 8px;
  font-size: 10px;
}

.tree-task-description {
  font-size: 12px;
  color: #6b7280;
  line-height: 1.4;
}

.tree-children {
  position: relative;
  margin-left: 20px;
}

.tree-line {
  position: absolute;
  left: -20px;
  top: -8px;
  width: 20px;
  height: 20px;
  border-left: 2px solid #d1d5db;
  border-bottom: 2px solid #d1d5db;
  border-bottom-left-radius: 8px;
}

.tree-line::before {
  content: '';
  position: absolute;
  left: -2px;
  top: 20px;
  width: 2px;
  height: calc(100% - 20px);
  background-color: #d1d5db;
}

.status-badge {
  font-size: 10px;
  padding: 2px 6px;
  border-radius: 10px;
  font-weight: 500;
  white-space: nowrap;
}

.status-badge.status-backlog {
  background-color: #f3f4f6;
  color: #6b7280;
}

.status-badge.status-paralyzed {
  background-color: #fef3c7;
  color: #92400e;
}

.status-badge.status-in-progress {
  background-color: #dbeafe;
  color: #1e40af;
}

.status-badge.status-testing {
  background-color: #fed7aa;
  color: #c2410c;
}

.status-badge.status-finalized {
  background-color: #d1fae5;
  color: #065f46;
}

/* Colores de tarjetas según estado */
.tree-task-card.status-backlog {
  border-left: 4px solid #6b7280;
}

.tree-task-card.status-paralyzed {
  border-left: 4px solid #f59e0b;
  background-color: #fffbeb;
}

.tree-task-card.status-in-progress {
  border-left: 4px solid #3b82f6;
  background-color: #eff6ff;
}

.tree-task-card.status-testing {
  border-left: 4px solid #f97316;
  background-color: #fff7ed;
}

.tree-task-card.status-finalized {
  border-left: 4px solid #10b981;
  background-color: #f0fdf4;
}

/* Responsive */
@media (max-width: 1200px) {
  .columns-container {
    overflow-x: auto;
    flex-wrap: nowrap;
  }

  .column {
    min-width: 300px;
    flex-shrink: 0;
  }
}

@media (max-width: 768px) {
  .kanban-container {
    padding: 16px;
  }

  .search-filters-section {
    flex-direction: column;
    align-items: stretch;
  }

  .search-container {
    min-width: auto;
  }

  .filters-container {
    justify-content: space-between;
  }

  .columns-container {
    flex-direction: column;
    overflow-y: auto;
  }

  .column {
    min-width: auto;
    flex-shrink: 1;
    max-height: 400px;
  }

  .modal-overlay {
    padding: 10px;
  }

  .modal-content {
    max-height: 95vh;
  }

  .task-header {
    flex-direction: column;
    align-items: flex-start;
    gap: 8px;
  }

  .task-badges {
    margin-left: 0;
  }

  .modal-wide {
    max-width: 95%;
  }

  .comments-modal {
    max-width: 95%;
  }

  .eisenhower-matrix {
    grid-template-columns: 1fr;
  }

  .comment-header {
    flex-direction: column;
    align-items: flex-start;
    gap: 8px;
  }

  .comment-actions-buttons {
    align-self: flex-end;
  }

  .tree-view-header {
    flex-direction: column;
    gap: 16px;
    align-items: flex-start;
  }

  .tree-view-title {
    font-size: 1.5rem;
  }

  .tree-container-main {
    padding: 16px;
  }
}
</style>