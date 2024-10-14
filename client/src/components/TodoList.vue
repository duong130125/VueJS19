<template>
  <div class="container">
    <h2>Quản lý công việc</h2>
    <div class="card add-task">
      <input
        type="text"
        placeholder="Nhập tên công việc"
        v-model="newTask"
        @keydown.enter="addTask"
        ref="taskInput"
      />
      <p v-if="errorMessage" class="error-message">{{ errorMessage }}</p>
      <button class="add-btn" @click="addTask" :disabled="isLoading">
        Thêm công việc
      </button>
    </div>
    <div class="card filters">
      <button :class="{ active: filter === 'all' }" @click="filter = 'all'">
        Tất cả
      </button>
      <button
        :class="{ active: filter === 'completed' }"
        @click="filter = 'completed'"
      >
        Hoàn thành
      </button>
      <button
        :class="{ active: filter === 'active' }"
        @click="filter = 'active'"
      >
        Đang thực hiện
      </button>
    </div>
    <div v-if="isLoading" class="loading">
      <div class="spinner"></div>
      <p>Đang tải dữ liệu...</p>
    </div>
    <div v-else class="task-list" style="max-height: 200px; overflow-y: auto">
      <div class="task" v-for="task in paginatedTasks" :key="task.id">
        <input
          type="checkbox"
          :checked="task.status"
          @change="toggleTask(task)"
        />
        <span :class="{ completed: task.status }">{{ task.name }}</span>
        <div class="actions">
          <span class="edit" @click="editTask(task)">✎</span>
          <span class="delete" @click="deleteTask(task.id)">🗑</span>
        </div>
      </div>
    </div>
    <div class="clear-buttons">
      <button
        class="clear-completed"
        @click="clearCompleted"
        :disabled="isLoading"
      >
        Xóa công việc hoàn thành
      </button>
      <button class="clear-all" @click="clearAll" :disabled="isLoading">
        Xóa tất cả công việc
      </button>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted } from "vue";
import axios from "axios";

const newTask = ref("");
const tasks = ref([]);
const filter = ref("all");
const isLoading = ref(false);
const errorMessage = ref("");

const fetchTasks = async () => {
  isLoading.value = true;
  try {
    const response = await axios.get("http://localhost:8080/jobs");
    tasks.value = response.data;
  } catch (error) {
    console.error("Error fetching tasks:", error);
  } finally {
    isLoading.value = false;
  }
};

const filteredTasks = computed(() => {
  let sortedTasks = [...tasks.value].sort(
    (a, b) => new Date(a.createdAt) - new Date(b.createdAt)
  );

  switch (filter.value) {
    case "completed":
      return sortedTasks.filter((task) => task.status);
    case "active":
      return sortedTasks.filter((task) => !task.status);
    default:
      return sortedTasks;
  }
});

const paginatedTasks = computed(() => {
  return filteredTasks.value.slice(0, 10);
});

const addTask = async () => {
  errorMessage.value = "";
  if (newTask.value.trim() === "") {
    errorMessage.value = "Tên công việc không được để trống.";
    return;
  }

  if (
    tasks.value.some(
      (task) => task.name.toLowerCase() === newTask.value.trim().toLowerCase()
    )
  ) {
    errorMessage.value = "Tên công việc đã tồn tại.";
    return;
  }

  isLoading.value = true;
  try {
    const response = await axios.post("http://localhost:8080/jobs", {
      name: newTask.value,
      status: false,
      createdAt: new Date().toISOString(),
    });
    tasks.value.unshift(response.data);
    newTask.value = "";
    focusInput();
  } catch (error) {
    console.error("Error adding task:", error);
  } finally {
    isLoading.value = false;
  }
};

const focusInput = () => {
  const input = document.querySelector(
    "input[placeholder='Nhập tên công việc']"
  );
  input && input.focus();
};

const toggleTask = async (task) => {
  isLoading.value = true;
  try {
    const response = await axios.patch(
      `http://localhost:8080/jobs/${task.id}`,
      {
        status: !task.status,
      }
    );
    Object.assign(task, response.data);
  } catch (error) {
    console.error("Error toggling task:", error);
  } finally {
    isLoading.value = false;
  }
};

const deleteTask = async (id) => {
  isLoading.value = true;
  try {
    await axios.delete(`http://localhost:8080/jobs/${id}`);
    tasks.value = tasks.value.filter((task) => task.id !== id);
  } catch (error) {
    console.error("Error deleting task:", error);
  } finally {
    isLoading.value = false;
  }
};

const editTask = async (task) => {
  const newTitle = prompt("Sửa công việc:", task.name);
  if (newTitle !== null && newTitle.trim() !== "") {
    isLoading.value = true;
    try {
      const response = await axios.patch(
        `http://localhost:8080/jobs/${task.id}`,
        {
          name: newTitle,
        }
      );
      Object.assign(task, response.data);
    } catch (error) {
      console.error("Error editing task:", error);
    } finally {
      isLoading.value = false;
    }
  }
};

const clearCompleted = async () => {
  isLoading.value = true;
  const completedTasks = tasks.value.filter((task) => task.status);
  for (const task of completedTasks) {
    try {
      await axios.delete(`http://localhost:8080/jobs/${task.id}`);
    } catch (error) {
      console.error("Error deleting completed task:", error);
    }
  }
  tasks.value = tasks.value.filter((task) => !task.status);
  isLoading.value = false;
};

const clearAll = async () => {
  isLoading.value = true;
  for (const task of tasks.value) {
    try {
      await axios.delete(`http://localhost:8080/jobs/${task.id}`);
    } catch (error) {
      console.error("Error deleting task:", error);
    }
  }
  tasks.value = [];
  isLoading.value = false;
};

onMounted(() => {
  fetchTasks();
});
</script>
