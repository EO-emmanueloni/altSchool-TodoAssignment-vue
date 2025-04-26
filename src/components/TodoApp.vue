<template>
    <div class="app-container">
      <div class="app-content">
        <h1 class="app-title">Todo Application
            <h2>Emmanuel Oni
                (ALT/SOE/024/1197)</h2>
        </h1>
        
        <!-- Search and Filter -->
        <div class="search-filter-container">
          <div class="search-filter-wrapper">
            <div class="search-input-container">
              <input 
                v-model="searchQuery" 
                type="text" 
                placeholder="Search todos..." 
                class="search-input"
              />
            </div>
            <div class="filter-container">
              <select 
                v-model="statusFilter" 
                class="status-filter"
              >
                <option value="all">All</option>
                <option value="completed">Completed</option>
                <option value="incomplete">Incomplete</option>
              </select>
            </div>
          </div>
        </div>
        
        <!-- Add New Todo -->
        <div class="add-todo-container">
          <h2 class="add-todo-title">Add New Todo</h2>
          <form @submit.prevent="addNewTodo" class="add-todo-form">
            <input 
              v-model="newTodoTitle" 
              type="text" 
              placeholder="Enter a new todo..." 
              class="add-todo-input"
              :class="{ 'input-error': titleError }"
            />
            <button 
              type="submit" 
              class="add-todo-button"
            >
              Add Todo
            </button>
          </form>
          <p v-if="titleError" class="error-message">{{ titleError }}</p>
        </div>
        
        <!-- Loading State -->
        <div v-if="loading" class="loading-container">
          <div class="loading-spinner"></div>
        </div>
        
        <!-- Error State -->
        <div v-else-if="error" class="error-container">
          <strong class="error-title">Error!</strong>
          <span class="error-message-text"> {{ error }}</span>
        </div>
        
        <!-- Todo List -->
        <div v-else>
          <div v-if="filteredTodos.length === 0" class="empty-todos-message">
            No todos found matching your criteria.
          </div>
          
          <div v-else class="todos-container">
            <ul class="todos-list">
              <li v-for="todo in paginatedTodos" :key="todo.id" class="todo-item">
                <div class="todo-content">
                  <input 
                    type="checkbox" 
                    :checked="todo.completed" 
                    @change="toggleTodo(todo)"
                    class="todo-checkbox"
                  />
                  <span 
                    class="todo-title"
                    :class="{ 'todo-completed': todo.completed }"
                  >
                    {{ todo.title }}
                  </span>
                  <span 
                    class="todo-status"
                    :class="todo.completed ? 'status-completed' : 'status-pending'"
                  >
                    {{ todo.completed ? 'Completed' : 'Pending' }}
                  </span>
                </div>
              </li>
            </ul>
          </div>
          
          <!-- Pagination -->
          <div class="pagination-container">
            <div class="pagination-info">
              Showing {{ startIndex + 1 }}-{{ endIndex }} of {{ filteredTodos.length }} todos
            </div>
            <div class="pagination-controls">
              <button 
                @click="currentPage--" 
                :disabled="currentPage === 1"
                class="pagination-button"
                :class="currentPage === 1 ? 'button-disabled' : ''"
              >
                Previous
              </button>
              <button 
                v-for="page in totalPages" 
                :key="page" 
                @click="currentPage = page"
                class="pagination-button"
                :class="currentPage === page ? 'button-active' : ''"
              >
                {{ page }}
              </button>
              <button 
                @click="currentPage++" 
                :disabled="currentPage === totalPages"
                class="pagination-button"
                :class="currentPage === totalPages ? 'button-disabled' : ''"
              >
                Next
              </button>
            </div>
          </div>
        </div>
      </div>
    </div>
  </template>
  
  <script setup>
  import { ref, computed, onMounted, watch } from 'vue';
  
  // State
  const todos = ref([]);
  const loading = ref(true);
  const error = ref(null);
  const searchQuery = ref('');
  const statusFilter = ref('all');
  const currentPage = ref(1);
  const itemsPerPage = 10;
  const newTodoTitle = ref('');
  const titleError = ref('');
  const addingTodo = ref(false);
  
  // Fetch todos from API
  const fetchTodos = async () => {
    loading.value = true;
    error.value = null;
    
    try {
      const response = await fetch('https://jsonplaceholder.typicode.com/todos');
      if (!response.ok) {
        throw new Error('Failed to fetch todos');
      }
      
      const data = await response.json();
      todos.value = data;
    } catch (err) {
      error.value = err.message;
      console.error('Error fetching todos:', err);
    } finally {
      loading.value = false;
    }
  };
  
  // Toggle todo completion status
  const toggleTodo = (todo) => {
    // In a real app, you would update the server here
    todo.completed = !todo.completed;
  };
  
  // Add a new todo
  const addNewTodo = async () => {
    // Validate input
    if (!newTodoTitle.value.trim()) {
      titleError.value = 'Todo title cannot be empty';
      return;
    }
    
    titleError.value = '';
    addingTodo.value = true;
    
    try {
      // Create new todo object
      const newTodo = {
        title: newTodoTitle.value.trim(),
        completed: false,
        userId: 1, // Using a default userId
      };
      
      // Send POST request to API
      const response = await fetch('https://jsonplaceholder.typicode.com/todos', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
        },
        body: JSON.stringify(newTodo),
      });
      
      if (!response.ok) {
        throw new Error('Failed to create todo');
      }
      
      // Get the response data (with the id assigned by the server)
      const createdTodo = await response.json();
      
      // Add to the beginning of our todos list
      // Note: JSONPlaceholder doesn't actually save the new todo on their server,
      // but returns a mock response with an id
      todos.value = [createdTodo, ...todos.value];
      
      // Reset form
      newTodoTitle.value = '';
      
      // Reset to first page to show the new todo
      currentPage.value = 1;
      
    } catch (err) {
      console.error('Error creating todo:', err);
      error.value = 'Failed to create todo. Please try again.';
    } finally {
      addingTodo.value = false;
    }
  };
  
  // Filter todos based on search query and status filter
  const filteredTodos = computed(() => {
    let result = todos.value;
    
    // Apply search filter
    if (searchQuery.value) {
      const query = searchQuery.value.toLowerCase();
      result = result.filter(todo => 
        todo.title.toLowerCase().includes(query)
      );
    }
    
    // Apply status filter
    if (statusFilter.value !== 'all') {
      const isCompleted = statusFilter.value === 'completed';
      result = result.filter(todo => todo.completed === isCompleted);
    }
    
    return result;
  });
  
  // Pagination logic
  const totalPages = computed(() => {
    return Math.ceil(filteredTodos.value.length / itemsPerPage);
  });
  
  const startIndex = computed(() => {
    return (currentPage.value - 1) * itemsPerPage;
  });
  
  const endIndex = computed(() => {
    return Math.min(startIndex.value + itemsPerPage, filteredTodos.value.length);
  });
  
  const paginatedTodos = computed(() => {
    return filteredTodos.value.slice(startIndex.value, endIndex.value);
  });
  
  // Reset to first page when filters change or new todo is added
  watch([searchQuery, statusFilter], () => {
    currentPage.value = 1;
  });
  
  // Fetch todos on component mount
  onMounted(() => {
    fetchTodos();
  });
  </script>
  
  <style scoped>
  :root {
    --color-primary: #4f46e5;
    --color-white: #ffffff;
    --color-gray-50: #f9fafb;
    --color-gray-100: #f3f4f6;
    --color-gray-200: #e5e7eb;
    --color-gray-300: #d1d5db;
    --color-gray-500: #6b7280;
    --color-gray-700: #374151;
    --color-gray-800: #1f2937;
    --color-gray-900: #111827;
    --color-red-100: #fee2e2;
    --color-red-400: #f87171;
    --color-red-500: #ef4444;
    --color-red-600: #dc2626;
    --color-red-700: #b91c1c;
    --color-green-100: #d1fae5;
    --color-green-800: #065f46;
    --color-yellow-100: #fef3c7;
    --color-yellow-800: #92400e;
  }
  
  /* App Container */
  .app-container {
    min-height: 100vh;
    background-color: var(--color-gray-50);
    padding: 2rem 1rem;
  }
  
  .app-content {
    max-width: 64rem;
    margin: 0 auto;
  }
  
  .app-title {
    font-size: 1.875rem;
    font-weight: 700;
    color: var(--color-gray-800);
    margin-bottom: 2rem;
    text-align: center;
  }
  
  /* Search and Filter */
  .search-filter-container {
    margin-bottom: 1.5rem;
  }
  
  .search-filter-wrapper {
    display: flex;
    flex-direction: column;
    gap: 1rem;
  }
  
  @media (min-width: 640px) {
    .search-filter-wrapper {
      flex-direction: row;
      align-items: center;
    }
  }
  
  .search-input-container {
    flex-grow: 1;
  }
  
  .search-input {
    width: 100%;
    padding: 0.5rem 1rem;
    border: 1px solid var(--color-gray-300);
    border-radius: 0.375rem;
  }
  
  .search-input:focus {
    outline: none;
    box-shadow: 0 0 0 2px rgba(79, 70, 229, 0.2);
    border-color: var(--color-primary);
  }
  
  .status-filter {
    width: 100%;
    padding: 0.5rem 1rem;
    border: 1px solid var(--color-gray-300);
    border-radius: 0.375rem;
  }
  
  @media (min-width: 640px) {
    .status-filter {
      width: auto;
    }
  }
  
  .status-filter:focus {
    outline: none;
    box-shadow: 0 0 0 2px rgba(79, 70, 229, 0.2);
    border-color: var(--color-primary);
  }
  
  /* Add Todo Section */
  .add-todo-container {
    margin-bottom: 1.5rem;
    background-color: var(--color-white);
    border-radius: 0.5rem;
    box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
    padding: 1rem;
  }
  
  .add-todo-title {
    font-size: 1.125rem;
    font-weight: 500;
    color: var(--color-gray-800);
    margin-bottom: 0.75rem;
  }
  
  .add-todo-form {
    display: flex;
    flex-direction: column;
    gap: 0.75rem;
  }
  
  @media (min-width: 640px) {
    .add-todo-form {
      flex-direction: row;
    }
  }
  
  .add-todo-input {
    flex-grow: 1;
    padding: 0.5rem 1rem;
    border: 1px solid var(--color-gray-300);
    border-radius: 0.375rem;
  }
  
  .add-todo-input:focus {
    outline: none;
    box-shadow: 0 0 0 2px rgba(79, 70, 229, 0.2);
    border-color: var(--color-primary);
  }
  
  .input-error {
    border-color: var(--color-red-500);
  }
  
  .add-todo-button {
    padding: 0.5rem 1rem;
    background-color: var(--color-primary);
    color: var(--color-white);
    border: none;
    border-radius: 0.375rem;
    cursor: pointer;
    transition: background-color 0.2s;
  }
  
  .add-todo-button:hover {
    background-color: rgba(79, 70, 229, 0.9);
  }
  
  .error-message {
    margin-top: 0.25rem;
    font-size: 0.875rem;
    color: var(--color-red-600);
  }
  
  /* Loading State */
  .loading-container {
    display: flex;
    justify-content: center;
    margin: 2rem 0;
  }
  
  .loading-spinner {
    height: 3rem;
    width: 3rem;
    border-radius: 50%;
    border: 2px solid transparent;
    border-top-color: var(--color-primary);
    border-bottom-color: var(--color-primary);
    animation: spin 1s linear infinite;
  }
  
  @keyframes spin {
    0% { transform: rotate(0deg); }
    100% { transform: rotate(360deg); }
  }
  
  /* Error State */
  .error-container {
    background-color: var(--color-red-100);
    border: 1px solid var(--color-red-400);
    color: var(--color-red-700);
    padding: 0.75rem 1rem;
    border-radius: 0.375rem;
    margin-bottom: 1.5rem;
    position: relative;
  }
  
  .error-title {
    font-weight: 700;
  }
  
  .error-message-text {
    display: block;
  }
  
  @media (min-width: 640px) {
    .error-message-text {
      display: inline;
    }
  }
  
  /* Empty Todos Message */
  .empty-todos-message {
    text-align: center;
    padding: 2rem 0;
    color: var(--color-gray-500);
  }
  
  /* Todos List */
  .todos-container {
    background-color: var(--color-white);
    border-radius: 0.5rem;
    box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
    overflow: hidden;
    margin-bottom: 1.5rem;
  }
  
  .todos-list {
    list-style-type: none;
    padding: 0;
    margin: 0;
    border-top: 1px solid var(--color-gray-200);
  }
  
  .todo-item {
    padding: 1rem 1.5rem;
    border-bottom: 1px solid var(--color-gray-200);
  }
  
  .todo-item:hover {
    background-color: var(--color-gray-50);
  }
  
  .todo-content {
    display: flex;
    align-items: center;
  }
  
  .todo-checkbox {
    height: 1.25rem;
    width: 1.25rem;
    color: var(--color-primary);
    border-radius: 0.25rem;
    border: 1px solid var(--color-gray-300);
  }
  
  .todo-checkbox:focus {
    outline: none;
    box-shadow: 0 0 0 2px rgba(79, 70, 229, 0.2);
  }
  
  .todo-title {
    margin-left: 0.75rem;
    color: var(--color-gray-900);
  }
  
  .todo-completed {
    text-decoration: line-through;
    color: var(--color-gray-500);
  }
  
  .todo-status {
    margin-left: auto;
    font-size: 0.75rem;
    padding: 0.25rem 0.5rem;
    border-radius: 9999px;
  }
  
  .status-completed {
    background-color: var(--color-green-100);
    color: var(--color-green-800);
  }
  
  .status-pending {
    background-color: var(--color-yellow-100);
    color: var(--color-yellow-800);
  }
  
  /* Pagination */
  .pagination-container {
    display: flex;
    justify-content: space-between;
    align-items: center;
  }
  
  .pagination-info {
    font-size: 0.875rem;
    color: var(--color-gray-700);
  }
  
  .pagination-controls {
    display: flex;
    gap: 0.5rem;
  }
  
  .pagination-button {
    padding: 0.25rem 0.75rem;
    border-radius: 0.375rem;
    border: 1px solid var(--color-gray-300);
    background-color: var(--color-white);
    font-size: 0.875rem;
    font-weight: 500;
    cursor: pointer;
  }
  
  .pagination-button:hover:not(.button-disabled) {
    background-color: var(--color-gray-50);
  }
  
  .button-active {
    background-color: var(--color-primary);
    color: var(--color-white);
    border-color: var(--color-primary);
  }
  
  .button-disabled {
    opacity: 0.5;
    cursor: not-allowed;
  }
  </style>