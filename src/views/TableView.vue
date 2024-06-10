<script setup>
import axios from 'axios'
import { onMounted, ref, computed, watch } from 'vue'
// Количество элементов на странице

let itemsPerPage = ref(10)
let itemCount = ref(0)

const lists = ref([])
const selectedName = ref('')
const selectedClass = ref('')

const items = ref([])
const searchQuery = ref('')
const sortKey = ref('')
const sortOrders = ref({
  id: 1, // По умолчанию сортируем по ID в порядке возрастания
  access: 1,
  classL: 1,
  name: 1,
  mark: 1,
  year: 1
})

let currentPage = ref(1) // Текущая страница

onMounted(async () => {
  await fetchData()
  await listData()
})

// Функция для получения данных с сервера
async function fetchData() {
  try {
    const { data } = await axios.get('https://11baf9f70829eb1e.mokky.dev/data')
    console.log(data)
    items.value = data
  } catch (e) {
    console.log(error)
  }
}

// Функция для получения данныхс сервера для списка в создании элемента
async function listData() {
  try {
    const { data } = await axios.get('https://11baf9f70829eb1e.mokky.dev/list')
    lists.value = data
  } catch (e) {
    console.log(error)
  }
}

const filteredItems = computed(() => {
  let filtered = items.value.slice()

  if (searchQuery.value) {
    const filterKey = searchQuery.value.toLowerCase()
    filtered = filtered.filter((item) => {
      // Учитывание значения "Да" и "Нет" для поля access
      const accessValue = item.access ? 'Да' : 'Нет'
      return Object.entries(item).some(([key, value]) => {
        if (key === 'access') {
          return String(accessValue).toLowerCase().includes(filterKey)
        }
        return String(value).toLowerCase().includes(filterKey)
      })
    })
  }

  if (sortKey.value) {
    const key = sortKey.value
    const order = sortOrders.value[key]
    filtered.sort((a, b) => {
      const aValue = typeof a[key] === 'boolean' ? (a[key] ? 1 : 0) : a[key]
      const bValue = typeof b[key] === 'boolean' ? (b[key] ? 1 : 0) : b[key]
      return (aValue === bValue ? 0 : aValue > bValue ? 1 : -1) * order
    })
  }

  // Вычисляем индексы элементов на текущей странице
  const startIndex = (currentPage.value - 1) * itemsPerPage.value
  const endIndex = startIndex + itemsPerPage.value
  itemCount = filtered.length
  // console.log('Фильтр1', itemCount)
  return filtered.slice(startIndex, endIndex)
})

function sortBy(key) {
  sortKey.value = key
  sortOrders.value[key] *= -1
}

function setPage(page) {
  currentPage.value = page
}

// Вычисляем общее количество страниц
const pageCount = ref(1)

watch(filteredItems, () => {
  pageCount.value = Math.ceil(itemCount / itemsPerPage.value)
})

function searchInput() {
  setPage(1) // Установка текущей страницы в 1 при изменении строки поиска
}

// Функция для удаления элемента из массива и JSON файла
async function deleteItem(index) {
  const startIndex = (currentPage.value - 1) * itemsPerPage.value
  const adjustedIndex = startIndex + index
  const deletedItem = items.value.splice(adjustedIndex, 1)[0]
  try {
    await axios.delete(`https://11baf9f70829eb1e.mokky.dev/data/${deletedItem.id}`)
  } catch (error) {
    console.error(error)
    // Если удаление не удалось, добавьить удаленный элемент обратно в массив
    items.value.splice(adjustedIndex, 0, deletedItem)
  }
}

// Состояние для новой строки
const newRecord = ref({
  id: '',
  access: '',
  classL: '',
  name: '',
  mark: '',
  year: ''
})
// Флаг для отображения модального окна
const showModal = ref(false)

// Переменные состояния для проверки заполнения полей
const showAccessError = ref(false)
const showClassLError = ref(false)
const showNameError = ref(false)
const showMarkError = ref(false)
const showYearError = ref(false)

// Функция для открытия модального окна
function openModal() {
  showModal.value = true
}

// Функция для закрытия модального окна и сброса данных новой строки
function closeModal() {
  showModal.value = false
  // Сброс данных новой строки
  newRecord.value = {
    id: '',
    access: '',
    classL: '',
    name: '',
    mark: '',
    year: ''
  }
}

// Функция для добавления новой строки
async function addRecord() {
  // Проверяем, что все поля заполнены
  if (
    newRecord.value.access &&
    newRecord.value.classL &&
    newRecord.value.name &&
    newRecord.value.mark &&
    newRecord.value.year
  ) {
    // Преобразуем значение access из строки в булевое значение Понял, если значения "true" и "false" передаются в виде строк, а не булевых значений, то мы можем преобразовать их в соответствующие булевые значения внутри нашего объекта newRecord
    newRecord.value.access = newRecord.value.access === 'true'

    try {
      // Отправка данных новой строки на сервер
      await axios.post('https://11baf9f70829eb1e.mokky.dev/data', newRecord.value)
      // Получение обновленных данных с сервера
      await fetchData()
      // Закрытие модального окна
      closeModal()
    } catch (error) {
      console.error(error)
    }
  } else {
    // Показываем сообщения об ошибках только для незаполненных полей
    validateAccess()
    validateClassL()
    validateName()
    validateMark()
    validateYear()
  }
}

// Функции для проверки заполнения полей
function validateAccess() {
  showAccessError.value = !newRecord.value.access
}

function validateClassL() {
  showClassLError.value = !newRecord.value.classL
}

function validateName() {
  showNameError.value = !newRecord.value.name
}

function validateMark() {
  showMarkError.value = !newRecord.value.mark
}

function validateYear() {
  showYearError.value = !newRecord.value.year
}

// Флаг для отображения модального окна редактирования
const showEditModal = ref(false)
const showEditNameError = ref(false) // Добавьте эту строку

// Данные выбранной строки для редактирования
const editedRecord = ref({
  id: '',
  access: '',
  classL: '',
  name: '',
  mark: '',
  year: ''
})

// Переменные состояния для проверки заполнения полей редактирования
const showEditAccessError = ref(false)
const showEditClassLError = ref(false)
const showEditMarkError = ref(false)
const showEditYearError = ref(false)

// Функция для открытия модального окна редактирования
function editItem(item) {
  // Заполните editedRecord данными выбранной строки по индексу index
  editedRecord.value = { ...item }

  // Отображаем модальное окно
  showEditModal.value = true
}

// Функция для закрытия модального окна редактирования
function closeEditModal() {
  // Сбрасываем данные редактирования и закрываем модальное окно
  editedRecord.value = {
    id: '',
    access: '',
    classL: '',
    name: '',
    mark: '',
    year: ''
  }
  showEditModal.value = false
}

// Функция для сохранения отредактированной записи
async function saveEditedRecord() {
  // Проверяем, что все поля заполнены
  if (
    editedRecord.value.access &&
    editedRecord.value.classL &&
    editedRecord.value.name &&
    editedRecord.value.mark &&
    editedRecord.value.year
  ) {
    // Преобразуем значение access из строки в булевое значение
    editedRecord.value.access = editedRecord.value.access === 'true'
    // console.log('idddddddd', editedRecord.value)
    try {
      // Отправка данных отредактированной записи на сервер для обновления
      await axios.patch(
        `https://11baf9f70829eb1e.mokky.dev/data/${editedRecord.value.id}`,
        editedRecord.value
      )

      // Обновление данных на клиенте
      await fetchData()
      // Закрытие модального окна редактирования записи
      closeEditModal()
    } catch (error) {
      console.error(error)
    }
  } else {
    // Показываем сообщения об ошибках только для незаполненных полей
    validateEditAccess()
    validateEditClassL()
    validateEditName()
    validateEditMark()
    validateEditYear()
  }
}

// Функции для проверки заполнения полей при редактировании записи
function validateEditAccess() {
  showEditAccessError.value = !editedRecord.value.access
}

function validateEditClassL() {
  showEditClassLError.value = !editedRecord.value.classL
}

function validateEditName() {
  showEditNameError.value = !editedRecord.value.name
}

function validateEditMark() {
  showEditMarkError.value = !editedRecord.value.mark
}

function validateEditYear() {
  showEditYearError.value = !editedRecord.value.year
}

// Флаги для управления видимостью колонок
const showId = ref(true)
const showAccess = ref(true)

// Флаг для отображения кнопок "+" и "-"
const showColumnControls = ref(false)

// Флаг для скрытия всех колонок
const hideAll = ref(false)

// Функция для отображения кнопок "+" и "-"
function toggleColumns() {
  showColumnControls.value = !showColumnControls.value
}

// Функция для отображения всех колонок
function showAllColumns() {
  hideAll.value = false
}

// Функция для скрытия всех колонок
function hideAllColumns() {
  hideAll.value = true
}

// Состояние для отображения/скрытия столбцов
const visibleColumns = ref({
  id: true,
  name: true,
  access: true,
  classL: true,
  mark: true,
  year: true
})

// Функция для скрытия/отображения столбца
function toggleColumn(column) {
  visibleColumns.value[column] = !visibleColumns.value[column]
}

// Функция для проверки видимости столбца
function showColumn(column) {
  return visibleColumns.value[column]
}
</script>

<template>
  <div>
    <h1>Электронная ведомость</h1>
    <form id="search">
      <input name="query" v-model="searchQuery" @input="searchInput" placeholder="Поиск" />
    </form>

    <!-- Модальное окно для ввода данных новой строки -->
    <div v-if="showModal" class="modal">
      <div class="modal-content">
        <h2>Добавить запись</h2>
        <div>
          <div>
            <label>Сдан:</label>
            <input type="radio" id="accessYes" value="true" v-model="newRecord.access" />
            <label for="accessYes">Да</label>
            <input type="radio" id="accessNo" value="false" v-model="newRecord.access" />
            <label for="accessNo">Нет</label>

            <span v-if="showAccessError" class="error-message"
              >Это поле обязательно для заполнения</span
            >
          </div>
        </div>
        <div>
          <label for="classL">Дисциплина:</label>
          <select id="classL" v-model="newRecord.classL">
            <option v-for="item in lists[0].classL" :key="item" :value="item">{{ item }}</option>
          </select>

          <span v-if="showClassLError" class="error-message">Это поле обязательно для выбора</span>
        </div>
        <div>
          <label for="name">Преподаватель:</label>
          <select id="name" v-model="newRecord.name">
            <option v-for="item in lists[0].name" :key="item" :value="item">{{ item }}</option>
          </select>

          <span v-if="showNameError" class="error-message">Это поле обязательно для выбора</span>
        </div>
        <div>
          <label for="mark">Оценка:</label>
          <input type="number" id="mark" v-model="newRecord.mark" min="0" />

          <span v-if="showMarkError" class="error-message"
            >Это поле обязательно для заполнения</span
          >
        </div>
        <div>
          <label for="year">Дата:</label>
          <input type="date" id="year" v-model="newRecord.year" />

          <span v-if="showYearError" class="error-message"
            >Это поле обязательно для заполнения</span
          >
        </div>
        <button @click="addRecord">Добавить</button>
        <button class="close" @click="closeModal">Закрыть&times;</button>
      </div>
    </div>

    <!-- Модальное окно для редактирования записей -->
    <div v-if="showEditModal" class="modal">
      <div class="modal-content">
        <h2>Редактировать запись (ID: {{ editedRecord.id }})</h2>
        <div>
          <label>Сдан:</label>
          <div>
            <input type="radio" id="editAccessYes" value="true" v-model="editedRecord.access" />
            <label for="editAccessYes">Да</label>
            <input type="radio" id="editAccessNo" value="false" v-model="editedRecord.access" />
            <label for="editAccessNo">Нет</label>
            <span v-if="showEditAccessError" class="error-message"
              >Это поле обязательно для заполнения</span
            >
          </div>
        </div>
        <div>
          <label for="editClassL">Дисциплина:</label>
          <select id="editClassL" v-model="editedRecord.classL">
            <option v-for="item in lists[0].classL" :key="item" :value="item">{{ item }}</option>
          </select>
          <span v-if="showEditClassLError" class="error-message"
            >Это поле обязательно для выбора</span
          >
        </div>
        <div>
          <label for="editName">Преподаватель:</label>
          <select id="editName" v-model="editedRecord.name">
            <option v-for="item in lists[0].name" :key="item" :value="item">{{ item }}</option>
          </select>
          <span v-if="showEditNameError" class="error-message"
            >Это поле обязательно для выбора</span
          >
        </div>
        <div>
          <label for="editMark">Оценка:</label>
          <input type="number" id="editMark" v-model="editedRecord.mark" min="0" />
          <span v-if="showEditMarkError" class="error-message"
            >Это поле обязательно для заполнения</span
          >
        </div>
        <div>
          <label for="editYear">Дата:</label>
          <input type="date" id="editYear" v-model="editedRecord.year" />
          <span v-if="showEditYearError" class="error-message"
            >Это поле обязательно для заполнения</span
          >
        </div>
        <button @click="saveEditedRecord">Сохранить</button>
        <button class="close" @click="closeEditModal">Закрыть&times;</button>
      </div>
    </div>
    <!-- Кнопка для отображения кнопок "+" и "-" -->
    <button @click="toggleColumns">Настройка колонок</button>

    <!-- Кнопки "+" и "-" для управления колонками -->
    <div v-if="showColumnControls">
      <div>
        <button @click="showAllColumns">+</button>
        <button @click="hideAllColumns">-</button>
      </div>
      <button @click="toggleColumn('id')">ID</button>
      <button @click="toggleColumn('access')">Сдан</button>
      <button @click="toggleColumn('classL')">Дисциплина</button>
      <button @click="toggleColumn('name')">Преподаватель</button>
      <button @click="toggleColumn('mark')">Оценка</button>
      <button @click="toggleColumn('year')">Дата</button>
    </div>
    <table v-if="filteredItems.length">
      <table v-if="!hideAll">
        <thead>
          <tr>
            <th v-if="showColumn('id')" @click="sortBy('id')" :class="{ active: sortKey === 'id' }">
              ID<span class="arrow" :class="sortOrders['id'] > 0 ? 'asc' : 'dsc'"></span>
            </th>
            <th
              v-if="showColumn('access')"
              @click="sortBy('access')"
              :class="{ active: sortKey === 'access' }"
            >
              Сдан<span class="arrow" :class="sortOrders['access'] > 0 ? 'asc' : 'dsc'"></span>
            </th>
            <th
              v-if="showColumn('classL')"
              @click="sortBy('classL')"
              :class="{ active: sortKey === 'classL' }"
            >
              Дисциплина<span
                class="arrow"
                :class="sortOrders['classL'] > 0 ? 'asc' : 'dsc'"
              ></span>
            </th>
            <th
              v-if="showColumn('name')"
              @click="sortBy('name')"
              :class="{ active: sortKey === 'name' }"
            >
              Преподаватель<span
                class="arrow"
                :class="sortOrders['name'] > 0 ? 'asc' : 'dsc'"
              ></span>
            </th>
            <th
              v-if="showColumn('mark')"
              @click="sortBy('mark')"
              :class="{ active: sortKey === 'mark' }"
            >
              Оценка<span class="arrow" :class="sortOrders['mark'] > 0 ? 'asc' : 'dsc'"></span>
            </th>
            <th
              v-if="showColumn('year')"
              @click="sortBy('year')"
              :class="{ active: sortKey === 'year' }"
            >
              Дата<span class="arrow" :class="sortOrders['year'] > 0 ? 'asc' : 'dsc'"></span>
            </th>
            <th><button @click.prevent="openModal">Добавить</button></th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="item in filteredItems" :key="item.id">
            <td v-if="showColumn('id')">{{ item.id }}</td>
            <td v-if="showColumn('access')">{{ item.access ? 'Да' : 'Нет' }}</td>
            <!-- Vue.js шаблон для преобразования значений true и false в "есть" и "нет" -->
            <td v-if="showColumn('classL')">{{ item.classL }}</td>
            <td v-if="showColumn('name')">{{ item.name }}</td>
            <td v-if="showColumn('mark')">{{ item.mark }}</td>
            <td v-if="showColumn('year')">{{ item.year }}</td>
            <td>
              <button @click="editItem(item)">Редактировать</button>
              <button @click="deleteItem(index)">Удалить</button>
            </td>
          </tr>
        </tbody>
      </table>
    </table>
    <p v-else>Совпадений не найдено.</p>
    <!-- Пагинация -->
    <div class="pagination">
      <button :disabled="currentPage === 1" @click="setPage(currentPage - 1)">Предыдущая</button>
      <!-- <span>{{ currentPage }}</span> -->
      <button :disabled="currentPage === pageCount" @click="setPage(currentPage + 1)">
        Следующая
      </button>
      <button
        v-for="pageNumber in pageCount"
        :key="pageNumber"
        @click="setPage(pageNumber)"
        :class="{ active: pageNumber === currentPage }"
      >
        {{ pageNumber }}
      </button>
    </div>
    <!-- Поле ввода для количества элементов на странице -->
    <div>
      <label>Элементов на странице:</label>
    </div>
    <input type="number" @input="searchInput" v-model.number="itemsPerPage" min="1" />
  </div>
</template>

<style>
.error-message {
  color: red;
  font-size: 0.8rem;
}
.active {
  background-color: #42b983;
  color: white;
}
#search {
  margin-bottom: 10px;
}
#search input {
  border: 2px solid grey; /* Задаем толщину и цвет рамки */
  border-radius: 3px; /* Задаем скругление углов рамки */
  padding: 8px; /* Задаем внутренний отступ вокруг текста */
  outline: none;
}
table {
  border-radius: 3px;
  background-color: grey;
}

th {
  background-color: #42b983;
  color: whitesmoke;
  cursor: pointer;
  user-select: none;
}

td {
  /* background-color: #d5b942;
  color: #260c1a; */
  background-color: #ffffff;
  color: #000000;
}

th,
td {
  min-width: 120px;
  padding: 10px 20px;
}

th.active {
  color: #fff;
}

th.active .arrow {
  opacity: 1;
}

.arrow {
  display: inline-block;
  vertical-align: middle;
  width: 0;
  height: 0;
  margin-left: 5px;
  opacity: 0.66;
}

.arrow.asc {
  border-left: 4px solid transparent;
  border-right: 4px solid transparent;
  border-bottom: 4px solid #fff;
}

.arrow.dsc {
  border-left: 4px solid transparent;
  border-right: 4px solid transparent;
  border-top: 4px solid #fff;
}
</style>
