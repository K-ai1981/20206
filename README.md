<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>TaskFlow | Modern Task Management</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <link href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.0.0/css/all.min.css" rel="stylesheet">
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Inter:wght@300;400;500;600;700&display=swap');
        
        body {
            font-family: 'Inter', sans-serif;
            background-color: #f8fafc;
        }

        .task-card {
            transition: all 0.2s ease;
        }

        .task-card:hover {
            transform: translateY(-2px);
            box-shadow: 0 4px 12px rgba(0, 0, 0, 0.05);
        }

        .completed-task {
            text-decoration: line-through;
            opacity: 0.6;
        }

        .custom-scrollbar::-webkit-scrollbar {
            width: 6px;
        }

        .custom-scrollbar::-webkit-scrollbar-track {
            background: #f1f1f1;
        }

        .custom-scrollbar::-webkit-scrollbar-thumb {
            background: #cbd5e1;
            border-radius: 10px;
        }

        .gradient-bg {
            background: linear-gradient(135deg, #6366f1 0%, #a855f7 100%);
        }

        #empty-state {
            display: none;
        }

        .tasks-container:empty + #empty-state {
            display: flex;
        }
    </style>
</head>
<body class="min-h-screen pb-12">

    <!-- Header -->
    <header class="gradient-bg text-white py-12 px-4 mb-8">
        <div class="max-w-3xl mx-auto">
            <div class="flex justify-between items-center mb-6">
                <div>
                    <h1 class="text-3xl font-bold tracking-tight">TaskFlow</h1>
                    <p class="text-indigo-100 mt-1" id="current-date"></p>
                </div>
                <div class="text-right">
                    <div class="text-2xl font-bold" id="completion-stats">0/0</div>
                    <div class="text-xs uppercase tracking-wider opacity-80">Tasks Completed</div>
                </div>
            </div>
            
            <!-- Progress Bar -->
            <div class="w-full bg-white/20 rounded-full h-2 mb-8">
                <div id="progress-bar" class="bg-white h-2 rounded-full transition-all duration-500" style="width: 0%"></div>
            </div>

            <!-- Add Task Input Area -->
            <div class="bg-white rounded-2xl shadow-xl p-2 flex flex-col md:flex-row gap-2">
                <input type="text" id="task-input" placeholder="What needs to be done?" 
                    class="flex-1 px-4 py-3 rounded-xl text-gray-800 focus:outline-none text-lg">
                
                <div class="flex gap-2 p-1">
                    <select id="priority-input" class="bg-gray-50 text-gray-600 px-3 py-2 rounded-lg border-none focus:ring-2 focus:ring-indigo-500 text-sm">
                        <option value="low">Low</option>
                        <option value="medium" selected>Medium</option>
                        <option value="high">High</option>
                    </select>
                    
                    <button id="add-btn" class="bg-indigo-600 hover:bg-indigo-700 text-white px-6 py-2 rounded-xl font-semibold transition-colors flex items-center gap-2">
                        <i class="fas fa-plus"></i>
                        <span>Add</span>
                    </button>
                </div>
            </div>
        </div>
    </header>

    <!-- Main Content -->
    <main class="max-w-3xl mx-auto px-4">
        
        <!-- Filters -->
        <div class="flex gap-4 mb-6 overflow-x-auto pb-2 custom-scrollbar">
            <button onclick="filterTasks('all')" class="filter-btn active px-4 py-2 rounded-full bg-indigo-100 text-indigo-700 font-medium text-sm whitespace-nowrap">All Tasks</button>
            <button onclick="filterTasks('active')" class="filter-btn px-4 py-2 rounded-full bg-white text-gray-600 hover:bg-gray-100 font-medium text-sm border border-gray-200 whitespace-nowrap">Active</button>
            <button onclick="filterTasks('completed')" class="filter-btn px-4 py-2 rounded-full bg-white text-gray-600 hover:bg-gray-100 font-medium text-sm border border-gray-200 whitespace-nowrap">Completed</button>
            <div class="ml-auto">
                <button onclick="clearCompleted()" class="text-gray-400 hover:text-red-500 text-sm font-medium transition-colors">Clear Completed</button>
            </div>
        </div>

        <!-- Task List -->
        <div id="tasks-container" class="space-y-3 tasks-container">
            <!-- Tasks injected here -->
        </div>

        <!-- Empty State -->
        <div id="empty-state" class="flex-col items-center justify-center py-20 text-center">
            <div class="bg-gray-100 rounded-full w-20 h-20 flex items-center justify-center mb-4 mx-auto">
                <i class="fas fa-clipboard-list text-3xl text-gray-400"></i>
            </div>
            <h3 class="text-xl font-semibold text-gray-700">No tasks found</h3>
            <p class="text-gray-500">Add a task to get started with your day.</p>
        </div>

    </main>

    <!-- Notification Toast -->
    <div id="toast" class="fixed bottom-8 left-1/2 -translate-x-1/2 bg-gray-900 text-white px-6 py-3 rounded-full shadow-2xl flex items-center gap-3 opacity-0 transition-opacity duration-300 pointer-events-none z-50">
        <i class="fas fa-check-circle text-green-400"></i>
        <span id="toast-message">Task added successfully</span>
    </div>

    <script>
        // State Management
        let tasks = JSON.parse(localStorage.getItem('tasks')) || [];
        let currentFilter = 'all';

        // Elements
        const taskInput = document.getElementById('task-input');
        const priorityInput = document.getElementById('priority-input');
        const addBtn = document.getElementById('add-btn');
        const tasksContainer = document.getElementById('tasks-container');
        const statsDisplay = document.getElementById('completion-stats');
        const progressBar = document.getElementById('progress-bar');
        const dateDisplay = document.getElementById('current-date');
        const toast = document.getElementById('toast');
        const toastMsg = document.getElementById('toast-message');

        // Initialize
        function init() {
            renderTasks();
            updateStats();
            updateDate();
            
            // Event Listeners
            addBtn.addEventListener('click', addTask);
            taskInput.addEventListener('keypress', (e) => {
                if (e.key === 'Enter') addTask();
            });
        }

        function updateDate() {
            const options = { weekday: 'long', year: 'numeric', month: 'long', day: 'numeric' };
            dateDisplay.innerText = new Date().toLocaleDateString(undefined, options);
        }

        function showToast(msg) {
            toastMsg.innerText = msg;
            toast.style.opacity = '1';
            setTimeout(() => {
                toast.style.opacity = '0';
            }, 3000);
        }

        function addTask() {
            const text = taskInput.value.trim();
            const priority = priorityInput.value;
            
            if (!text) return;

            const newTask = {
                id: Date.now(),
                text,
                priority,
                completed: false,
                createdAt: new Date().toISOString()
            };

            tasks.unshift(newTask);
            saveTasks();
            renderTasks();
            taskInput.value = '';
            showToast('Task added');
        }

        function toggleTask(id) {
            tasks = tasks.map(t => t.id === id ? { ...t, completed: !t.completed } : t);
            saveTasks();
            renderTasks();
        }

        function deleteTask(id) {
            tasks = tasks.filter(t => t.id !== id);
            saveTasks();
            renderTasks();
            showToast('Task removed');
        }

        function clearCompleted() {
            const count = tasks.filter(t => t.completed).length;
            if (count === 0) return;
            tasks = tasks.filter(t => !t.completed);
            saveTasks();
            renderTasks();
            showToast(`Cleared ${count} completed tasks`);
        }

        function saveTasks() {
            localStorage.setItem('tasks', JSON.stringify(tasks));
            updateStats();
        }

        function updateStats() {
            const total = tasks.length;
            const completed = tasks.filter(t => t.completed).length;
            const percentage = total === 0 ? 0 : (completed / total) * 100;
            
            statsDisplay.innerText = `${completed}/${total}`;
            progressBar.style.width = `${percentage}%`;
        }

        function filterTasks(type) {
            currentFilter = type;
            document.querySelectorAll('.filter-btn').forEach(btn => {
                btn.classList.remove('bg-indigo-100', 'text-indigo-700');
                btn.classList.add('bg-white', 'text-gray-600');
            });
            event.target.classList.add('bg-indigo-100', 'text-indigo-700');
            renderTasks();
        }

        function getPriorityColor(priority) {
            switch(priority) {
                case 'high': return 'text-red-500 bg-red-50';
                case 'medium': return 'text-orange-500 bg-orange-50';
                case 'low': return 'text-green-500 bg-green-50';
                default: return 'text-gray-500 bg-gray-50';
            }
        }

        function renderTasks() {
            tasksContainer.innerHTML = '';
            
            let filteredTasks = tasks;
            if (currentFilter === 'active') filteredTasks = tasks.filter(t => !t.completed);
            if (currentFilter === 'completed') filteredTasks = tasks.filter(t => t.completed);

            if (filteredTasks.length === 0) {
                // Empty state handled by CSS :empty selector sibling
                return;
            }

            filteredTasks.forEach(task => {
                const priorityClass = getPriorityColor(task.priority);
                
                const taskEl = document.createElement('div');
                taskEl.className = `task-card bg-white p-4 rounded-xl shadow-sm border border-gray-100 flex items-center gap-4 ${task.completed ? 'opacity-75' : ''}`;
                
                taskEl.innerHTML = `
                    <button onclick="toggleTask(${task.id})" class="flex-shrink-0 w-6 h-6 rounded-full border-2 border-indigo-200 flex items-center justify-center transition-all ${task.completed ? 'bg-indigo-500 border-indigo-500 text-white' : 'hover:border-indigo-500'}">
                        ${task.completed ? '<i class="fas fa-check text-[10px]"></i>' : ''}
                    </button>
                    <div class="flex-1 min-w-0">
                        <p class="text-gray-800 font-medium truncate ${task.completed ? 'completed-task' : ''}">${task.text}</p>
                        <div class="flex items-center gap-2 mt-1">
                            <span class="text-[10px] uppercase font-bold px-2 py-0.5 rounded ${priorityClass}">
                                ${task.priority}
                            </span>
                        </div>
                    </div>
                    <button onclick="deleteTask(${task.id})" class="text-gray-300 hover:text-red-500 transition-colors p-2">
                        <i class="far fa-trash-alt"></i>
                    </button>
                `;
                
                tasksContainer.appendChild(taskEl);
            });
        }

        // Run Init
        init();
    </script>
</body>
</html>
